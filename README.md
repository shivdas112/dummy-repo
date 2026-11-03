# GitOps: Argo CD + Argo Rollouts

This directory turns the **push-based** pipeline (CI runs `kubectl apply`) into a
**pull-based** one (a controller inside the cluster continuously reconciles the
cluster to whatever is in Git).

```
┌──────────┐   git push    ┌────────────────┐   PR / merge   ┌──────────────┐
│ Dev      │──────────────►│ GitHub repo    │───────────────►│ main/develop │
└──────────┘               │ (source of     │                └──────┬───────┘
                           │  truth)        │                       │
                           └───────┬────────┘                       │
                                   │ poll / webhook every 3m        │
                                   ▼                                │
                        ┌─────────────────────┐                     │
                        │ Argo CD (in cluster)│◄────────────────────┘
                        │  diff → sync        │
                        └──────┬──────────────┘
                               │ kubectl-equivalent apply
                               ▼
                    ┌──────────────────────┐
                    │ Cluster state        │
                    │ Rollout → ALB weights│
                    └──────────────────────┘
```

**Why it matters (interview answer):**
> "With push-based CD, CI needs cluster-admin credentials and there is no
> guarantee the cluster matches Git — someone always `kubectl edit`s something.
> With Argo CD, the cluster pulls its desired state, CI never holds cluster
> credentials, and any manual drift is detected and reverted within minutes."

## Bootstrap order

```bash
# 0. EKS + add-ons already exist (Phases 3–4 of the main guide)
# 1. Argo CD, Argo Rollouts, and the app-of-apps
./scripts/07-install-gitops.sh

# 2. Watch the platform add-ons install themselves from Git
kubectl -n argocd get applications
kubectl -n argocd get applications -w
```

## What lives where

| Path | Purpose |
|---|---|
| `argocd/values-argocd.yaml` | Helm values: HA, SSO via GitHub OIDC, RBAC, Slack notifications, internal ALB ingress |
| `argocd/projects/taskflow-project.yaml` | `AppProject` — the security boundary (allowed repos, namespaces, resource kinds) |
| `argocd/appsets/taskflow-appset.yaml` | `ApplicationSet` — one Application per env (dev auto-syncs, prod does not) |
| `argocd/appsets/platform-appset.yaml` | Platform add-ons via Argo **multi-source** (Helm chart + values from this repo), ordered by sync waves |
| `rollouts/services-stable-canary.yaml` | `-root` / `-stable` / `-canary` Services |
| `rollouts/rollout-api.yaml` | Canary `Rollout`: 10% → 30% → 60% → 100% with automated analysis |
| `rollouts/analysistemplates.yaml` | Prometheus-based gates: success rate ≥ 95%, p95 < 500 ms, DB up |
| `rollouts/ingress-canary.yaml` | Ingress using `use-annotation` + `group.name` (shares the existing ALB) |
| `rollouts/hpa-rollout.yaml` | HPA that scales the Rollout instead of a Deployment |

## The three demos to run

**1. GitOps drift correction**
```bash
kubectl -n taskflow scale deploy/taskflow-web --replicas=1     # break it on purpose
# Argo CD shows OutOfSync, then selfHeal restores replicas within ~3 min
kubectl -n argocd get app taskflow-dev -o jsonpath='{.status.sync.status}'
```

**2. A real canary**
```bash
./scripts/08-canary-demo.sh        # bumps the image, then watches weights shift
kubectl argo rollouts get rollout taskflow-api -n taskflow --watch
```

**3. Automatic rollback**
```bash
# Deploy an intentionally broken image; the success-rate analysis aborts it and
# traffic snaps back to 100% stable with no human involved.
kubectl argo rollouts set image taskflow-api api="$ECR_BACKEND:broken" -n taskflow
kubectl argo rollouts get rollout taskflow-api -n taskflow --watch
```

## Things that bite people (and how this repo handles them)

| Trap | Fix in this repo |
|---|---|
| Argo CD fights the HPA over `spec.replicas` | `ignoreDifferences` on `/spec/replicas` |
| Argo CD reverts External-Secret-written data | `ignoreDifferences` on the Secret's `/data` |
| Second Ingress = second ALB = second bill | `alb.ingress.kubernetes.io/group.name: taskflow` |
| 262144-byte annotation limit on big CRDs | `ServerSideApply=true` |
| Prod wiped by auto-prune | prod Application has `prune: false`, `autoSync: false` |
| CRDs missing before the app needs them | sync waves: ESO (0) → Kyverno/Rollouts (1) → monitoring (2) → app (10) |
| Canary saturates nodes | `maxSurge: 25%`, `maxUnavailable: 0` |
