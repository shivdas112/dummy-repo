# TaskFlow — End-to-End AWS + DevOps + Observability Project

> **A complete, production-shaped project you can build, deploy, and demo.**
> Everything in this repo is real: real Terraform, real Kubernetes manifests, real
> CI/CD, real Prometheus rules, real security gates. Nothing is pseudo-code.

**Stack touched:** AWS (VPC, EC2, EKS, ECR, S3, ALB, RDS, IAM, Secrets Manager, CloudWatch) ·
Terraform · Git/GitHub · Docker · Kubernetes · GitHub Actions + Jenkins · SonarQube · Trivy ·
Prometheus · Grafana · Alertmanager · Kyverno · External Secrets

---

## Table of contents

1. [The project in one page](#1-the-project-in-one-page)
2. [Architecture](#2-architecture)
3. [Tool map — what is used where](#3-tool-map--what-is-used-where)
4. [Repository file structure](#4-repository-file-structure)
5. [Prerequisites](#5-prerequisites)
6. [Phase 0 — Local setup and Git strategy](#phase-0--local-setup-and-git-strategy)
7. [Phase 1 — Run the app locally with Docker Compose](#phase-1--run-the-app-locally-with-docker-compose)
8. [Phase 2 — Terraform remote state bootstrap](#phase-2--terraform-remote-state-bootstrap)
9. [Phase 3 — Provision AWS with Terraform](#phase-3--provision-aws-with-terraform)
10. [Phase 4 — Cluster access and add-ons](#phase-4--cluster-access-and-add-ons)
11. [Phase 5 — Build and push images to ECR](#phase-5--build-and-push-images-to-ecr)
12. [Phase 6 — Deploy to Kubernetes](#phase-6--deploy-to-kubernetes)
13. [Phase 7 — Code quality and security scanning](#phase-7--code-quality-and-security-scanning)
14. [Phase 8 — CI/CD with GitHub Actions](#phase-8--cicd-with-github-actions)
15. [Phase 9 — CI/CD with Jenkins (alternative)](#phase-9--cicd-with-jenkins-alternative)
16. [Phase 10 — Monitoring with Prometheus and Grafana](#phase-10--monitoring-with-prometheus-and-grafana)
17. [Phase 11 — Alerting](#phase-11--alerting)
18. [Phase 12 — Security hardening checklist](#phase-12--security-hardening-checklist)
19. [Phase 13 — Cost, teardown, and verification](#phase-13--cost-teardown-and-verification)
20. [Phase 14 — GitOps with Argo CD](#phase-14--gitops-with-argo-cd)
21. [Phase 15 — Progressive delivery with Argo Rollouts](#phase-15--progressive-delivery-with-argo-rollouts-on-alb)
20. [Interview questions you must be able to answer](#20-interview-questions-you-must-be-able-to-answer)
21. [Troubleshooting](#21-troubleshooting)
22. [Where to take it next](#22-where-to-take-it-next)

---

## 1. The project in one page

### What you are building

**TaskFlow** is a small three-tier task manager:

| Tier | Technology | Where it runs |
|------|-----------|---------------|
| Frontend | Static HTML/CSS/JS served by nginx | 2 pods behind an ALB |
| Backend API | Node.js 20 + Express + `prom-client` | 2–20 pods (HPA) on EKS |
| Database | PostgreSQL 16 on RDS | Private subnets, Multi-AZ in prod |

It is deliberately *simple in business logic* and *serious in operations* — the
point of the project is the platform around it, not the app.

### Why this project is a strong portfolio piece

It demonstrates the exact chain a real platform team owns:

```
code  →  git  →  CI (test, Sonar, Trivy)  →  image  →  ECR
                                                          ↓
                          Terraform-provisioned EKS + RDS + ALB
                                                          ↓
                       Kubernetes (HPA, probes, NetworkPolicy, IRSA)
                                                          ↓
                 Prometheus → Alertmanager → Grafana (SLO dashboards)
```

### Deliverables you end up with

- A live URL served by an AWS Application Load Balancer
- A Grafana dashboard showing golden signals for your own service
- A CI pipeline that **fails** when code quality or image CVEs regress
- A GitOps loop where the cluster reconciles itself from Git (drift heals in ~3 min)
- Canary deployments that auto-abort on a Prometheus-detected regression
- A `terraform destroy` that tears the whole thing down cleanly
- Screenshots and a repo you can walk an interviewer through end to end

### Resume bullets this project justifies

- Provisioned a production-style AWS environment (VPC with 2 AZs, private subnets, NAT, EKS 1.30 managed node groups, RDS PostgreSQL 16, ALB, S3, IAM/IRSA) entirely with reusable Terraform modules and remote state in S3 + DynamoDB.
- Built a GitHub Actions CI/CD pipeline with OIDC keyless AWS auth, SonarQube quality gates, Trivy image/IaC scanning, ECR image push, and automated rollout with smoke tests and automatic rollback.
- Deployed a containerised Node.js/PostgreSQL application to EKS with zero-downtime rolling updates, HPA, PodDisruptionBudgets, restricted Pod Security Standards, and default-deny NetworkPolicies.
- Implemented full observability with the Prometheus Operator and Grafana: RED metrics, SLO burn-rate alerting, node/pod/database dashboards, and Alertmanager routing to Slack and PagerDuty.
- Eliminated static AWS credentials by adopting OIDC federation for CI and IRSA for pods, and centralised DB secrets in AWS Secrets Manager via External Secrets Operator.
- Migrated delivery from push-based CI to GitOps with Argo CD (ApplicationSets, sync waves, drift self-healing), removing cluster credentials from the pipeline entirely.
- Implemented progressive delivery with Argo Rollouts: weighted ALB canary (10→30→60→100%) gated by three Prometheus analysis metrics, with automatic abort and zero-downtime rollback.

---

## 2. Architecture

```
                        ┌──────────────────────────── Internet ────────────────────────────┐
                        │                                                                  │
                        ▼                                                                  │
        ┌───────────────────────────────┐                                                  │
        │  Route 53 / your domain       │                                                  │
        └───────────────┬───────────────┘                                                  │
                        ▼                                                                  │
        ┌───────────────────────────────┐        ACM certificate (HTTPS :443)              │
        │  Application Load Balancer    │◄────── WAF (optional OWASP rules)                │
        │  (public subnets, 2 AZs)      │                                                  │
        └───────────────┬───────────────┘                                                  │
                        │  target-type: ip                                                  │
        ════════════════│═══════════ VPC 10.20.0.0/16 ═══════════════════════════════════════
                        ▼                                                                  │
   ┌────────────────────────────────────────────┐     ┌──────────────────────────────┐      │
   │  Public subnets  10.20.1.0/24, .2.0/24     │     │  NAT Gateways (egress only)  │      │
   │  • ALB  • Bastion (SSM, no SSH)            │     └──────────────┬───────────────┘      │
   └────────────────────────────────────────────┘                    │                      │
   ┌────────────────────────────────────────────┐                    │                      │
   │  Private subnets 10.20.11.0/24, .12.0/24   │◄───────────────────┘                      │
   │                                            │                                           │
   │   ┌──────────── EKS 1.30 cluster ────────┐ │      ┌─────────────────────────────┐      │
   │   │ namespace: taskflow                  │ │      │  RDS PostgreSQL 16          │      │
   │   │  ┌──────────────┐ ┌──────────────┐  │ │◄────►│  private, encrypted (KMS)   │      │
   │   │  │ taskflow-web │ │ taskflow-api │  │ │ 5432 │  automated backups          │      │
   │   │  │ nginx :8080  │ │ node :3000   │  │ │      │  logs → CloudWatch          │      │
   │   │  │ 2..6 pods    │ │ 2..20 pods   │  │ │      └─────────────────────────────┘      │
   │   │  └──────────────┘ └──────────────┘  │ │                     ▲                       │
   │   │  HPA · PDB · NetworkPolicy · IRSA    │ │                     │ secret               │
   │   ├──────────────────────────────────────┤ │      ┌──────────────┴──────────────┐       │
   │   │ kube-system: ALB controller,         │ │      │  Secrets Manager            │       │
   │   │   EBS CSI, CoreDNS, metrics-server   │ │      │  (rotated DB credentials)   │       │
   │   ├──────────────────────────────────────┤ │      └──────────────▲──────────────┘       │
   │   │ monitoring: Prometheus · Grafana ·   │ │                     │                      │
   │   │   Alertmanager · node-exporter · KSM │ │      ┌──────────────┴──────────────┐       │
   │   │   CloudWatch exporter (RDS/ALB)      │ │      │ External Secrets Operator   │       │
   │   └──────────────────────────────────────┘ │      └─────────────────────────────┘       │
   │                                            │                                           │
   │   Managed node group: t3.medium ×2 (2→4)   │      ┌─────────────────────────────┐      │
   └────────────────────────────────────────────┘      │  S3: app assets, tfstate,    │      │
                                                       │  LB access logs (KMS, SSE)   │      │
                                                       └─────────────────────────────┘      │
```

### Request flow, in words

1. A user hits `https://taskflow.example.com`.
2. DNS resolves to the **ALB** in the public subnets. TLS terminates at the ALB
   using an **ACM** certificate.
3. The ALB forwards to **pod IPs directly** (`target-type: ip`) in the private
   subnets — the target group health check hits `/healthz`.
4. `PATH /api/*` → `taskflow-api` Service → Node.js pods on port 3000.
   Everything else → `taskflow-web` Service → nginx pods on port 8080.
5. nginx also proxies `/api/` to the API service, so the SPA works without CORS issues.
6. The API reads its DB credentials from a Kubernetes Secret that the
   **External Secrets Operator** continuously syncs from **AWS Secrets Manager**
   (which itself was populated by Terraform).
7. The API connects to **RDS PostgreSQL** in the private subnets on port 5432.
8. **Prometheus** scrapes `/metrics` from the API every 15 s (via a ServiceMonitor),
   plus node-exporter and kube-state-metrics. **Grafana** renders the dashboards,
   **Alertmanager** routes alerts to Slack/PagerDuty.

### Design decisions you must be able to defend

| Decision | Why | Trade-off |
|---|---|---|
| EKS + RDS instead of a self-hosted DB pod | Managed control plane and managed backups; no etcd/DB ops | Cost (~$73/mo control plane + RDS) |
| Pod IPs as ALB targets (`target-type: ip`) | One hop fewer, better for large clusters, no NodePort sprawl | Needs VPC CNI (default on EKS) |
| IRSA + GitHub OIDC instead of access keys | No long-lived secrets to leak or rotate | Slightly more setup |
| External Secrets Operator instead of a `Secret` manifest | Secrets never live in Git; rotation is automatic | One more controller to run |
| Kustomize overlays instead of Helm charts for *our* app | Plain YAML is easier to read in a portfolio; overlays handle env differences | Less parameterised than Helm |
| Terraform, not CDK/CloudFormation | Multi-cloud-safe, huge module ecosystem, plan/apply review workflow | State management is on you |
| Default-deny NetworkPolicy | Blast-radius reduction | Every new service needs an allow rule |

---

## 3. Tool map — what is used where

| Tool | Purpose in this project | Key files |
|---|---|---|
| **Git / GitHub** | Version control, branch protection, PR reviews, OIDC identity provider | `.gitignore`, branch strategy below |
| **Terraform** | All AWS infrastructure, modular, remote state | `terraform/**` |
| **Docker** | Multi-stage builds, non-root images, local compose stack | `app/*/Dockerfile`, `docker/docker-compose.yml` |
| **Amazon ECR** | Immutable-tag image registry with scan-on-push | `terraform/modules/ecr` |
| **Kubernetes / EKS** | Workload runtime, HPA, probes, PDB, NetworkPolicy | `kubernetes/base`, `kubernetes/overlays` |
| **Kustomize** | Per-environment overlays (image tag, replicas, config) | `kubernetes/overlays/{dev,prod}` |
| **GitHub Actions** | CI/CD: test → Sonar → Trivy → build → push → deploy | `.github/workflows/*.yml` |
| **Jenkins** | Alternative CI/CD implementation of the same pipeline | `ci/Jenkinsfile` |
| **SonarQube** | Static analysis + quality gate (coverage, duplication, code smells) | `security/sonar-project.properties` |
| **Trivy** | Vulnerability, secret, and IaC misconfiguration scanning | `security/trivy.yaml`, CI steps |
| **Prometheus** | Metrics collection (RED + USE) | `monitoring/prometheus/*` |
| **Grafana** | Dashboards (golden signals, pods, nodes, RDS) | `monitoring/grafana/dashboards/*.json` |
| **Alertmanager** | Alert routing/dedup (Slack, PagerDuty) | `monitoring/prometheus/values-kube-prometheus-stack.yaml` |
| **Kyverno** | Admission policies (no `:latest`, must set limits) | `security/kyverno/cluster-policies.yaml` |
| **AWS Secrets Manager** | Source of truth for DB credentials | `terraform/modules/rds` |
| **External Secrets Operator** | Syncs AWS secrets → Kubernetes Secrets | `kubernetes/base/external-secret.yaml` |
| **AWS Load Balancer Controller** | Turns `Ingress` into a real ALB | `kubernetes/base/ingress.yaml` |
| **CloudWatch** | VPC flow logs, RDS logs, Container Insights | `terraform/modules/vpc`, `modules/eks` |
| **Argo CD** | GitOps: cluster pulls desired state from Git, drift auto-heals | `gitops/argocd/*` |
| **Argo Rollouts** | Canary deployments with ALB weight shifting + automated analysis | `gitops/rollouts/*` |

### AWS services covered (your checklist)

- **EC2** — bastion host (SSM-only, IMDSv2, encrypted root volume) + EKS worker nodes (managed node group)
- **S3** — application bucket (versioned, SSE-KMS, public access blocked, TLS-only policy, lifecycle rules), access-log bucket, Terraform state bucket
- **Load balancer** — ALB created declaratively by the AWS Load Balancer Controller from an `Ingress`
- **IAM** — cluster role, node role, IRSA roles (ALB controller, app pod, EBS CSI, External Secrets), GitHub OIDC deploy role, VPC flow-log role, RDS enhanced-monitoring role
- **Database** — RDS PostgreSQL 16, private subnets, KMS-encrypted, Multi-AZ in prod, automated backups, performance insights, log exports
- **Bonus** — Secrets Manager, KMS, CloudWatch Logs, ECR, NAT Gateway, VPC Flow Logs, EBS (PVCs)

---

## 4. Repository file structure

```
taskflow-devops/
├── README.md                            # quick-start
├── DEVOPS_PROJECT_GUIDE.md              # ← this document
│
├── app/                                 # the application (build context for Docker)
│   ├── backend/
│   │   ├── Dockerfile                   # 3-stage: deps → test → hardened runtime
│   │   ├── package.json
│   │   ├── .env.example
│   │   ├── .dockerignore
│   │   ├── src/
│   │   │   ├── index.js                 # bootstrap, DB retry, graceful shutdown
│   │   │   ├── app.js                   # express routes + probes + /metrics
│   │   │   ├── db.js                    # pg pool + schema init
│   │   │   └── metrics.js               # prom-client registry + RED middleware
│   │   └── tests/app.test.js            # jest + supertest
│   └── frontend/
│       ├── Dockerfile                   # nginx:1.27-alpine
│       ├── nginx.conf                   # reverse proxy + security headers + gzip
│       ├── .dockerignore
│       └── public/{index.html,app.js}
│
├── docker/
│   └── docker-compose.yml               # local 3-tier stack + optional monitoring
│
├── terraform/
│   ├── versions.tf                      # provider pinning
│   ├── backend.tf                       # S3 + DynamoDB remote state
│   ├── providers.tf                     # aws / kubernetes / helm providers
│   ├── variables.tf                     # documented input variables
│   ├── main.tf                          # module composition
│   ├── outputs.tf                       # cluster name, ECR URLs, role ARNs…
│   ├── environments/
│   │   ├── dev/{backend.hcl,terraform.tfvars}
│   │   └── prod/{backend.hcl,terraform.tfvars}
│   └── modules/
│       ├── vpc/                         # VPC, 4 subnets, NAT×2, flow logs
│       ├── eks/                         # cluster, node group, OIDC, add-ons
│       ├── rds/                         # postgres, secret, SG, monitoring
│       ├── s3/                          # app + log buckets, KMS
│       ├── ecr/                         # repos, immutable tags, lifecycle
│       ├── iam/                         # IRSA roles + GitHub OIDC role
│       └── bastion/                     # SSM-only EC2 instance
│
├── kubernetes/
│   ├── base/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml               # PSS "restricted" enforced
│   │   ├── serviceaccounts.yaml         # IRSA annotations
│   │   ├── configmap.yaml
│   │   ├── external-secret.yaml         # SecretStore + ExternalSecret
│   │   ├── api-deployment.yaml          # 3 probes, preStop, read-only FS
│   │   ├── web-deployment.yaml
│   │   ├── services.yaml
│   │   ├── ingress.yaml                 # ALB annotations (creates the ALB)
│   │   ├── hpa.yaml                     # autoscaling/v2
│   │   ├── pdb.yaml
│   │   └── networkpolicy.yaml           # default deny + explicit allows
│   ├── overlays/
│   │   ├── dev/{kustomization,patch-*}.yaml
│   │   └── prod/{kustomization,patch-*}.yaml
│   └── monitoring/
│       ├── servicemonitor.yaml          # scrape config (must live in `monitoring` ns)
│       └── taskflow-prometheusrule.yaml # 15 alert rules as a CRD
│
├── monitoring/
│   ├── prometheus/
│   │   ├── prometheus-local.yml         # dev/docker-compose config
│   │   ├── alert-rules.yml              # 14 SLO/infra alert rules
│   │   ├── values-kube-prometheus-stack.yaml
│   │   └── values-cloudwatch-exporter.yaml
│   └── grafana/dashboards/
│       └── taskflow-overview.json       # 17-panel golden-signals dashboard
│
├── .github/workflows/                   # GitHub only reads this path
│   ├── ci-cd.yml                        # app pipeline
│   └── terraform.yml                    # infra pipeline (plan on PR)
│
├── ci/
│   └── Jenkinsfile                      # Jenkins equivalent
│
├── gitops/                              # Phase 14 + 15
│   ├── README.md                        # GitOps flow + the 3 demos
│   ├── argocd/
│   │   ├── values-argocd.yaml           # HA, SSO, RBAC, Slack, internal ALB
│   │   ├── projects/taskflow-project.yaml    # AppProject security boundary
│   │   └── appsets/
│   │       ├── taskflow-appset.yaml     # 1 Application per env (dev auto, prod manual)
│   │       └── platform-appset.yaml     # add-ons via multi-source + sync waves
│   └── rollouts/
│       ├── services-stable-canary.yaml  # root / stable / canary Services
│       ├── rollout-api.yaml             # 10→30→60→100% canary
│       ├── analysistemplates.yaml       # Prometheus gates (success rate, p95, DB)
│       ├── ingress-canary.yaml          # use-annotation + shared ALB
│       └── hpa-rollout.yaml             # HPA targeting the Rollout
│
├── security/
│   ├── sonar-project.properties
│   ├── trivy.yaml
│   ├── .trivyignore
│   ├── .pre-commit-config.yaml
│   └── kyverno/cluster-policies.yaml
│
└── scripts/
    ├── 01-bootstrap-tfstate.sh          # S3 + DynamoDB state bucket
    ├── 02-install-addons.sh             # metrics-server, ESO, kube-prom-stack
    ├── 03-deploy.sh                     # kustomize + kubectl + rollout wait
    ├── 04-smoke-test.sh                 # health + CRUD + metrics
    ├── 05-teardown.sh                   # ordered destroy
    ├── 06-rotate-db-password.sh         # secret rotation demo
    ├── 00-verify.sh                     # 6 offline checks, no AWS needed
    ├── 07-install-gitops.sh             # Argo CD + Rollouts + app-of-apps
    └── 08-canary-demo.sh                # build, push, and watch a canary
```

---

## 5. Prerequisites

### Install these locally

| Tool | Version | Install (macOS/Linux) |
|---|---|---|
| AWS CLI v2 | ≥ 2.15 | `curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o aws.zip && unzip aws.zip && sudo ./aws/install` |
| Terraform | ≥ 1.7 | `brew install terraform` / [releases](https://releases.hashicorp.com/terraform/) |
| kubectl | within ±1 minor of cluster | `brew install kubectl` |
| kustomize | ≥ 5.4 | `brew install kustomize` |
| Helm | ≥ 3.14 | `brew install helm` |
| Docker Desktop / engine | ≥ 25 | docker.com |
| Node.js | 20 LTS | `nvm install 20` |
| jq, git, make | any recent | `brew install jq git make` |
| Trivy (optional, local scans) | ≥ 0.54 | `brew install trivy` |

### Accounts / access you need

- An **AWS account** where you can create VPCs, EKS, RDS, IAM roles.
  *Use a dedicated sandbox account or an IAM user with `PowerUserAccess` + `IAMFullAccess` for learning.*
- A **GitHub repository** (public is fine) to host CI.
- A **SonarCloud** account (free tier) *or* a self-hosted SonarQube container. The free
  SonarCloud tier is the fastest path — no server to run.
- *(Optional)* A **domain** in Route 53 for HTTPS. Without one, you use the raw ALB DNS name over HTTP.

### Cost warning — read this first

| Resource | Approx. monthly cost (us-east-1) |
|---|---|
| EKS control plane | **$73** |
| 2 × t3.medium nodes | ~$60 |
| NAT gateways × 2 | ~$65 (+ data) |
| RDS db.t3.micro, single-AZ | ~$15 |
| ALB | ~$18 |
| Bastion t3.micro | ~$8 |
| **Total dev** | **≈ $230–250/month** |

> **Do the whole build in one or two sittings, take your screenshots, then run
> `./scripts/05-teardown.sh dev`.** A weekend of learning costs ~$15–25.
> Set an AWS Budget alert at $50 before you start — instructions in Phase 3.

---

## Phase 0 — Local setup and Git strategy

### 0.1 Configure the AWS CLI

```bash
aws configure
# AWS Access Key ID:     <from IAM console — use an admin/dev user, never root>
# AWS Secret Access Key: <...>
# Default region name:   us-east-1
# Default output format: json

aws sts get-caller-identity      # verify
```

Then export two shell variables you'll reuse everywhere:

```bash
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

### 0.2 Create the repository

```bash
git init taskflow-devops && cd taskflow-devops
# copy in the files from this project
git add . && git commit -m "feat: initial TaskFlow platform scaffold"
git branch -M main
git remote add origin git@github.com:<you>/taskflow-devops.git
git push -u origin main
```

### 0.3 Branching model (Trunk-based, simplified)

```
main      ────●──────────────●──────────────●──────►  auto-deploy to PROD (manual approval)
               \            /  \            /
develop    ─────●──●──●────●────●──●──●────●────────►  auto-deploy to DEV
                    \  /
feature/*   ─────────●──────►  PR → CI runs tests, Sonar, Trivy
```

Rules to configure in **GitHub → Settings → Branches**:

- Protect `main`: require PR, 1 approval, passing `Lint & Test` + `SonarQube Scan`, no force pushes.
- Require conversation resolution and signed commits (optional but impressive).
- Enable **Environments**: `dev` (no reviewers), `production` (required reviewer = you).
  This is what makes `deploy-prod` in the workflow pause for a human click.

### 0.4 `.gitignore` (at repo root)

```gitignore
# Node
node_modules/
coverage/
*.log

# Terraform — NEVER commit state or plans
**/.terraform/*
*.tfstate
*.tfstate.*
*.tfplan
tfplan
crash.log
override.tf
override.tf.json
.terraform.lock.hcl.bak

# Secrets
.env
.env.*
*.pem
*.key
kubeconfig*
secrets.yaml

# OS / editors
.DS_Store
.idea/
.vscode/
```

> **Golden rule:** if it can contain a credential or a state file, it does not
> go in Git. Secrets live in AWS Secrets Manager; state lives in S3.

### 0.5 Install the local pre-commit hooks (shifts security left)

```bash
pip install pre-commit
pre-commit install
pre-commit run --all-files     # first run downloads the hook environments
```

Now a `terraform fmt` violation, a hardcoded secret (gitleaks), a bad Dockerfile
(hadolint), or a HIGH Terraform misconfiguration (trivy) blocks the commit
*before* it ever reaches CI.

---

## Phase 1 — Run the app locally with Docker Compose

**Always get the app working locally before touching the cloud.** Debugging a
container is 100× cheaper than debugging it inside EKS.

### 1.1 Start the stack

```bash
cd docker
docker compose up --build
```

What you get:

| URL | Service |
|---|---|
| http://localhost:8080 | Frontend (nginx) |
| http://localhost:3000/healthz | API liveness |
| http://localhost:3000/readyz | API readiness (checks Postgres) |
| http://localhost:3000/metrics | Prometheus metrics |
| http://localhost:5432 | PostgreSQL |

Open http://localhost:8080, add a task, tick it, delete it.

### 1.2 Add monitoring to the local stack

```bash
docker compose --profile monitoring up -d
# Prometheus → http://localhost:9090  (Status ▸ Targets shows taskflow-api UP)
# Grafana    → http://localhost:3001  (admin / admin)
```

In Prometheus, query `rate(taskflow_http_requests_total[1m])` while clicking
around the UI — you are watching your own app's metrics.

### 1.3 Verify the tests pass

```bash
cd ../app/backend
npm install
npm test
# 7 tests, coverage reported. These are the exact tests CI will run.
```

### 1.4 What the Dockerfiles do (know this for interviews)

**Backend** — three stages:
1. `deps`: `npm ci --omit=dev` for a lean production dependency tree.
2. `test`: full install + `npm test` (used by CI, never shipped).
3. `runtime`: `node:20-alpine`, `dumb-init` as PID 1 so `SIGTERM` is forwarded
   correctly, a **non-root user (UID 10001)**, `NODE_ENV=production`, and a
   `HEALTHCHECK`.

**Frontend** — `nginx:1.27-alpine`, the site config installed as a *template*
into `/etc/nginx/templates/`, and `NGINX_ENVSUBST_FILTER="BACKEND_(HOST|PORT)"`
so only our two variables get substituted (nginx's own `$host`, `$remote_addr`,
etc. must survive). Containers listen on **8080**, not 80, so they run as non-root.

**Why multi-stage matters:** the final backend image contains no dev dependencies,
no test files, and no build toolchain — smaller image (≈ 150 MB vs ≈ 1 GB) means
faster pulls, fewer CVEs, and a smaller attack surface.

---

## Phase 2 — Terraform remote state bootstrap

**Why:** Terraform state is the single most valuable file in the project. It maps
your code to real AWS resources. If it lives on your laptop you can't collaborate,
you have no history, and one lost file orphans hundreds of dollars of resources.

### 2.1 Create the state bucket + lock table (AWS CLI, once)

```bash
cd taskflow-devops
export PROJECT=taskflow AWS_REGION=us-east-1
./scripts/01-bootstrap-tfstate.sh
```

This creates:
- **S3 bucket** `taskflow-tfstate-<ACCOUNT_ID>` — versioned (you can roll back a
  bad state), SSE-encrypted with AES256, public access fully blocked, TLS-only
  bucket policy.
- **DynamoDB table** `taskflow-tfstate-lock` — pay-per-request, `LockID` hash key.
  When someone runs `apply`, Terraform writes a lock item; a second concurrent
  run fails fast instead of corrupting state.

### 2.2 Point the environments at it

Edit both `terraform/environments/*/backend.hcl` and replace `<ACCOUNT_ID>`:

```hcl
bucket         = "taskflow-tfstate-123456789012"
key            = "taskflow/dev/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "taskflow-tfstate-lock"
encrypt        = true
```

Two environments ⇒ two keys ⇒ two independent state files in one bucket.
That is exactly how you keep `dev` and `prod` from ever touching each other.

### 2.3 Initialise

```bash
cd terraform
terraform init -backend-config=environments/dev/backend.hcl
# "Successfully configured the backend "s3"!"
```

> **Workspaces vs directories.** This project uses *backend-config per
> environment* (cleaner separation, different vars per env). If an interviewer
> asks about `terraform workspace`, explain the trade-off: workspaces share one
> backend and one config, which is fine for identical environments but gets
> messy when prod and dev diverge (which they always do).

---

## Phase 3 — Provision AWS with Terraform

### 3.1 What `terraform apply` creates, in dependency order

```
vpc        →  VPC, IGW, 2 public + 2 private subnets, 2 EIPs, 2 NAT GWs,
              route tables, VPC Flow Logs → CloudWatch
s3         →  KMS key + alias, app bucket (versioned, SSE-KMS, TLS-only,
              lifecycle), access-log bucket, S3 access logging
ecr        →  2 repositories, IMMUTABLE tags, scan-on-push, lifecycle policy
eks        →  cluster IAM role, cluster SG, node SG, control plane (all 5 log
              types), OIDC provider, node IAM role, managed node group,
              EBS CSI add-on (+IRSA), CloudWatch observability add-on,
              EKS access entries
rds        →  DB subnet group, SG (5432 from nodes+bastion only), parameter
              group (force_ssl=1, slow query log), random password,
              Secrets Manager secret+version, RDS instance, enhanced monitoring
bastion    →  SSM-only EC2 in a public subnet, encrypted gp3 root, IMDSv2-only
iam        →  IRSA roles: ALB controller, app pod, external-secrets;
              GitHub OIDC provider + deploy role
helm       →  AWS Load Balancer Controller (via IRSA)
```

### 3.2 Set a budget alert first (2 minutes, protects your wallet)

```bash
aws budgets create-budget \
  --account-id "$AWS_ACCOUNT_ID" \
  --budget '{"BudgetName":"taskflow-dev","BudgetLimit":{"Amount":"50","Unit":"USD"},
             "TimeUnit":"MONTHLY","BudgetType":"COST"}' \
  --notifications-with-subscribers '[{"Notification":{"NotificationType":"ACTUAL",
      "ComparisonOperator":"GREATER_THAN","Threshold":80,"ThresholdType":"PERCENTAGE"},
      "Subscribers":[{"SubscriptionType":"EMAIL","Address":"you@example.com"}]}]'
```

### 3.3 Plan, review, apply

```bash
cd terraform
terraform init -backend-config=environments/dev/backend.hcl -reconfigure
terraform validate
terraform fmt -recursive

terraform plan -var-file=environments/dev/terraform.tfvars -out=tfplan
# Read the plan. Expect ~90-110 resources for dev.

terraform apply tfplan          # 15–20 minutes (EKS is slow)
```

**While it runs, read the plan output.** Being able to explain *why* a resource
is being created — and what would be destroyed on the next change — is the single
biggest signal of Terraform competence in an interview.

### 3.4 Verify

```bash
terraform output
# cluster_name      = "taskflow-dev"
# ecr_repository_urls = { backend = ".../taskflow-dev/backend", frontend = "..." }
# ci_deploy_role_arn = "arn:aws:iam::...:role/taskflow-dev-github-actions-deploy"
# db_endpoint        = <sensitive>

aws eks list-clusters
aws rds describe-db-instances --query 'DBInstances[].{id:DBInstanceIdentifier,status:DBInstanceStatus,class:DBInstanceClass}'
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'   # empty until an Ingress exists
```

### 3.5 Save the image repository URLs

```bash
export ECR_BACKEND=$(terraform output -json ecr_repository_urls | jq -r .backend)
export ECR_FRONTEND=$(terraform output -json ecr_repository_urls | jq -r .frontend)
echo "$ECR_BACKEND  $ECR_FRONTEND"
```

### 3.6 The module contract (what each module takes and returns)

| Module | Inputs | Outputs |
|---|---|---|
| `vpc` | `name, vpc_cidr, public/private_subnet_cidrs, azs, cluster_name` | `vpc_id, public_subnet_ids, private_subnet_ids` |
| `eks` | `name, kubernetes_version, vpc_id, private_subnet_ids, node_*` | `cluster_name, cluster_endpoint, cluster_certificate_authority_data, oidc_provider_arn, node_security_group_id` |
| `rds` | `name, vpc_id, subnet_ids, allowed_security_group_id, db_*` | `db_endpoint, db_address, secret_arn, security_group_id` |
| `s3` | `name_prefix, account_id` | `app_bucket_name, app_bucket_arn, kms_key_arn` |
| `ecr` | `name_prefix, keep_last` | `repository_urls, repository_arns` |
| `iam` | `name, oidc_provider_*, s3_bucket_arn, db_secret_arn, github_repo, ecr_repository_arns` | `alb_controller_role_arn, app_pod_role_arn, external_secrets_role_arn, github_actions_role_arn` |
| `bastion` | `name, vpc_id, subnet_id, instance_type` | `instance_id, security_group_id` |

Good Terraform answers when asked "why modules?":
reuse (dev and prod call the same code), blast-radius control (a module owns
one concern), reviewability (a 60-line module is auditable; a 900-line root
module is not), and versioning (modules can be tagged and pinned).

---

## Phase 4 — Cluster access and add-ons

### 4.1 Point kubectl at the cluster

```bash
aws eks update-kubeconfig --name taskflow-dev --region us-east-1
kubectl get nodes -o wide
kubectl get pods -A
```

You should see 2 nodes `Ready` and the EKS system pods (CoreDNS, kube-proxy,
aws-node, ebs-csi, cloudwatch-agent, alb-controller).

### 4.2 Install the add-ons

```bash
cd taskflow-devops
CLUSTER=taskflow-dev ./scripts/02-install-addons.sh
```

| Add-on | Why it exists |
|---|---|
| **metrics-server** | Provides the CPU/memory numbers HPA needs. Without it HPA shows `<unknown>`. |
| **External Secrets Operator** | Watches `ExternalSecret` CRs and writes real `Secret`s from AWS Secrets Manager. |
| **kube-prometheus-stack** | Prometheus Operator + Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics, as one bundle. |
| **Grafana dashboard ConfigMap** | Labelled `grafana_dashboard=1`, so Grafana's sidecar auto-imports it. |
| **Kyverno** | Admission controller that blocks `:latest` images and pods without limits. |

### 4.3 Verify each piece

```bash
kubectl -n kube-system get pods | grep -E 'metrics-server|alb-controller|ebs'
kubectl -n external-secrets get pods
kubectl -n monitoring get pods
kubectl get crd | grep -E 'servicemonitor|externalsecret|prometheusrules'

kubectl top nodes                 # metrics-server works
kubectl -n monitoring port-forward svc/monitoring-grafana 3001:80
# → http://localhost:3001  (admin / password printed by the install script)
```

---

## Phase 5 — Build and push images to ECR

### 5.1 Create a GitHub OIDC trust before you need it

Terraform already created `taskflow-dev-github-actions-deploy`. Open
`terraform/modules/iam/main.tf` and set your repo:

```hcl
github_repo = "your-github-username/taskflow-devops"
```

Then re-apply. Now GitHub Actions can assume that role with **zero stored
secrets** — the trust policy only accepts tokens whose `sub` claim matches
`repo:your-username/taskflow-devops:*`.

Grab the ARN:

```bash
terraform output -raw ci_deploy_role_arn
# arn:aws:iam::123456789012:role/taskflow-dev-github-actions-deploy
```

Add it to GitHub → Settings → Secrets and variables → Actions:

| Secret name | Value |
|---|---|
| `AWS_DEPLOY_ROLE_ARN` | the ARN above |
| `AWS_ACCOUNT_ID` | `123456789012` |
| `SONAR_TOKEN` | from SonarCloud → My Account → Security |
| `SONAR_HOST_URL` | `https://sonarcloud.io` (or your SonarQube URL) |

### 5.2 Manual first push (do this once, by hand, to prove it works)

```bash
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com"

cd app/backend
docker build --target runtime -t "$ECR_BACKEND:v1" .
docker push "$ECR_BACKEND:v1"

cd ../frontend
docker build -t "$ECR_FRONTEND:v1" .
docker push "$ECR_FRONTEND:v1"

# confirm scan results
aws ecr describe-images --repository-name taskflow-dev/backend \
  --query 'imageDetails[].{tag:imageTags[0],scan:imageScanStatus.status,size:imageSizeInBytes}'
```

### 5.3 Why `IMMUTABLE` tags + `scan-on-push`

- **Immutable**: once `v1` is pushed it can never be overwritten. If a deploy
  of `v1` was bad, you *know* what code is inside `v1`. Mutable tags make
  rollbacks and audits meaningless.
- **Scan-on-push**: ECR runs a free Basic vulnerability scan on every push.
  Trivy in CI is the gate; this is the safety net.

---

## Phase 6 — Deploy to Kubernetes

### 6.1 Fill in your account ID everywhere it appears

```bash
grep -rl "ACCOUNT_ID" kubernetes/ | xargs sed -i.bak "s/ACCOUNT_ID/$AWS_ACCOUNT_ID/g"
find kubernetes -name '*.bak' -delete
```

Also patch the RDS endpoint into your overlay:

```bash
DB_HOST=$(terraform -chdir=terraform output -raw db_endpoint | cut -d: -f1)
sed -i "s|REPLACED_BY_RDS_ENDPOINT|$DB_HOST|" kubernetes/base/configmap.yaml
# (or set it in overlays/dev/patch-configmap.yaml)
```

And the ExternalSecret's secret key must match what Terraform named:

```yaml
# kubernetes/base/external-secret.yaml
dataFrom:
  - extract:
      key: taskflow/dev/rds/credentials   # ← must match local.name = project-env
```

### 6.2 Apply the overlay

```bash
kubectl apply -k kubernetes/overlays/dev
# or: ./scripts/03-deploy.sh dev v1
```

### 6.3 Watch it come up

```bash
kubectl -n taskflow get pods -w
kubectl -n taskflow describe pod -l app.kubernetes.io/name=taskflow-api
kubectl -n taskflow logs -l app.kubernetes.io/name=taskflow-api --tail=50

# External Secret resolved?
kubectl -n taskflow get externalsecret
kubectl -n taskflow get secret taskflow-db-credentials -o jsonpath='{.data.DB_USER}' | base64 -d; echo
```

`ExternalSecret` status must be `SecretSynced=True`. If it's `SecretSyncedError`,
the IRSA role's namespace/ServiceAccount name does not match the trust policy —
that's the #1 cause.

### 6.4 Get the ALB URL

```bash
kubectl -n taskflow get ingress taskflow -w
# NAME       CLASS  HOSTS   ADDRESS                                        PORTS
# taskflow   alb    *       k8s-taskflow-xxxx.us-east-1.elb.amazonaws.com  80

curl -i "$(kubectl -n taskflow get ingress taskflow -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')/healthz"
```

**ALB provisioning takes 2–4 minutes** after the Ingress is created. The
controller has to create the load balancer, target groups, listeners, and
security groups.

### 6.5 Run the smoke test

```bash
./scripts/04-smoke-test.sh
# → health, create, list, update, delete, metrics — all green
```

### 6.6 Exercises that prove you understand it

```bash
# 1. Zero-downtime rolling update
kubectl -n taskflow set image deployment/taskflow-api \
  api=$ECR_BACKEND:v1 --record
kubectl -n taskflow rollout status deployment/taskflow-api
# In another terminal, loop curl -I against /healthz — you should never see a 5xx.

# 2. Rollback
kubectl -n taskflow rollout undo deployment/taskflow-api

# 3. Watch HPA react to load
kubectl -n taskflow get hpa -w &
kubectl -n taskflow run loadgen --rm -it --image=busybox --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://taskflow-api:3000/api/tasks; done'
# replicas should climb toward maxReplicas, then settle back after 5 min

# 4. Prove the NetworkPolicy works
kubectl -n default run nope --rm -it --image=curlimages/curl --restart=Never -- \
  curl -m 5 http://taskflow-api.taskflow.svc:3000/healthz
# → times out. Cross-namespace traffic is denied by default.
```

### 6.7 How the probes are tuned (and why)

| Probe | Path | Purpose | Failure consequence |
|---|---|---|---|
| `startupProbe` | `/healthz` | Give the app up to 60 s (12×5 s) to boot without liveness killing it | Pod is restarted after 60 s |
| `livenessProbe` | `/healthz` | Is the process wedged? | Pod restarted |
| `readinessProbe` | `/readyz` | Can it serve traffic (DB reachable)? | Pod removed from ALB target group — **not** restarted |

`/healthz` deliberately does **not** touch the database. If liveness checked the
DB, a 30-second RDS blip would restart every pod at once — an outage caused by
your own health check. Readiness is where dependency checks belong.

---

## Phase 7 — Code quality and security scanning

### 7.1 SonarQube / SonarCloud

Create a SonarCloud project for your repo, then generate a token. Locally:

```bash
brew install sonar-scanner
export SONAR_TOKEN=xxxxx
sonar-scanner -Dsonar.host.url=https://sonarcloud.io   # reads security/sonar-project.properties
```

The **quality gate** (`sonar.qualitygate.wait=true`) makes the pipeline fail when
any of these regress on *new* code:

- Coverage on new code < 80%
- Duplicated lines on new code > 3%
- Any new blocker/critical issue
- Maintainability rating worse than A
- Security hotspots not reviewed

> The phrase that matters in interviews: **"quality gate on new code, not on the
> whole codebase"** — you can't fix 10 years of legacy debt in one sprint, but you
> can stop making it worse.

### 7.2 Trivy — four scan types, four different questions

```bash
# 1. Filesystem / dependency CVEs  (does my npm tree have a known-bad package?)
trivy fs --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 app/

# 2. Container image CVEs  (does my base image have unpatched OS packages?)
trivy image --severity CRITICAL --ignore-unfixed --exit-code 1 \
  "$ECR_BACKEND:v1"

# 3. IaC misconfiguration  (is my Terraform insecure?)
trivy config --severity HIGH,CRITICAL terraform/
# typical findings: S3 without encryption, SG open to 0.0.0.0/0, no IMDSv2

# 4. Secrets  (did anyone commit an access key?)
trivy fs --scanners secret .
```

Trivy also scans Kubernetes YAML: `trivy config kubernetes/` flags missing
resource limits, `privileged: true`, running as root, and absent securityContexts.

### 7.3 How the gates are wired in CI

| Stage | Tool | Blocks on | Where |
|---|---|---|---|
| Lint + unit tests | npm/eslint/jest | any failure | `test` job |
| Static analysis | SonarQube | quality gate fails | `sonarqube` job |
| Dependency CVEs | Trivy fs | HIGH/CRITICAL with a fix available | `security-scan` job |
| Secrets | Trivy secret | any finding | `security-scan` job |
| IaC | Trivy config | report-only (start here, tighten later) | `security-scan` job |
| Image | Trivy image | CRITICAL with a fix available | `build-and-push` |
| Runtime admission | Kyverno | `:latest` tag, missing limits | cluster |

`--ignore-unfixed` is not laziness: an unfixed CVE has no remediation path, so
failing the build just trains everyone to bypass the gate. Track those in
`.trivyignore` with an owner and an expiry date instead.

---

## Phase 8 — CI/CD with GitHub Actions

The workflows live in `.github/workflows/` — the only path GitHub reads, so they
are ready to run as soon as you push.

```bash
git add .github && git commit -m "ci: add pipelines" && git push
```

### 8.1 The pipeline graph

```
                    ┌──────────┐
   push/PR ────────►│   test   │  npm ci · lint · jest · coverage artifact
                    └────┬─────┘
              ┌──────────┴──────────┐
        ┌─────▼─────┐        ┌──────▼──────┐
        │ sonarqube │        │ security-   │  trivy fs / config / secret
        │ (gate)    │        │ scan        │
        └─────┬─────┘        └──────┬──────┘
              └──────────┬──────────┘
                  ┌──────▼─────────────┐
                  │ build-and-push     │  matrix: backend, frontend
                  │ (skip on PR)       │  buildx · trivy image · ECR push · SBOM
                  └──────┬─────────────┘
          ┌──────────────┴──────────────┐
   develop branch                 main branch
   ┌──────▼──────┐              ┌───────▼────────┐
   │ deploy-dev  │              │ deploy-prod    │  ← requires manual approval
   │ auto        │              │ + canary       │    (GitHub Environment)
   └─────────────┘              └────────────────┘
```

### 8.2 The three things that make this pipeline modern

**1. OIDC instead of access keys.**

```yaml
permissions:
  id-token: write
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
    aws-region: us-east-1
```

GitHub mints a short-lived OIDC token; AWS STS exchanges it for temporary
credentials valid for one job. Nothing long-lived is stored. The role's trust
policy pins `repo:you/taskflow-devops:*`, so a fork cannot assume it.

**2. Environment protection = human approval for prod.**

`environment: production` with a required reviewer means the job *stops* and
waits. That is your change-approval gate, visible in the Actions UI, with an
audit trail.

**3. Layered caching.** `cache-from/to: type=gha` on buildx plus `cache: npm` on
setup-node cuts a 6-minute build to under 2 minutes on repeat runs — and it's
free.

### 8.3 Watch it run

Push to `develop`:

```bash
git checkout -b develop && git push -u origin develop
```

Then open the repo → **Actions → CI/CD**. Click into `Build & Push (backend)` and
read the Trivy table — this is the artefact you screenshot for your portfolio.

### 8.4 The Terraform pipeline

`terraform.yml` runs on any change under `terraform/`:
`fmt -check → init → validate → Checkov → plan`, and posts the plan as a PR
comment (truncated to 60k chars). On merge to `main` it applies the saved plan.

Two details worth knowing:
- `terraform plan -out=tfplan` then `apply tfplan` guarantees you apply **exactly**
  what was reviewed — not a re-computed plan that drifted.
- `-no-color` keeps the PR comment readable.

---

## Phase 9 — CI/CD with Jenkins (alternative)

Same pipeline, different engine — useful because many enterprises still run
Jenkins, and you should show you can do both.

### 9.1 Minimal Jenkins setup

```bash
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts-jdk17
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Then install plugins: **Docker Pipeline, Amazon ECR, SonarQube Scanner,
Kubernetes CLI, Pipeline: Stage View, Slack Notification, HTML Publisher,
JUnit, AWS Steps**.

### 9.2 Configure

1. **Manage Jenkins → Tools**: add NodeJS 20, SonarQube Scanner.
2. **Manage Jenkins → System → SonarQube servers**: name it `SonarQube`,
   URL + token credential.
3. **Credentials**: `aws-credentials` (AWS creds or an assumed role),
   `aws-account-id` (secret text), `sonar-token`, `slack-webhook`.
4. **New Item → Pipeline → Pipeline script from SCM** → point at your repo,
   script path `ci/Jenkinsfile`.

### 9.3 Jenkins vs GitHub Actions — the comparison table

| Dimension | GitHub Actions | Jenkins |
|---|---|---|
| Hosting | SaaS, zero maintenance | Self-hosted, you patch it |
| Cost | Free minutes on public repos | Server + your time |
| Config | YAML in repo | Groovy in repo (`Jenkinsfile`) |
| Ecosystem | Marketplace actions | 1800+ plugins (a liability too) |
| Dynamic agents | Runners (GitHub-hosted or self-hosted) | Kubernetes plugin → pods per build |
| Secretless AWS auth | Native OIDC support | Possible via plugin/IRSA on agents |
| Best at | Cloud-native repos on GitHub | Complex on-prem, legacy, huge custom logic |
| Failure mode | Vendor lock-in, minute limits | Configuration drift, plugin CVEs |

Say this in an interview: *"I'd pick GitHub Actions for a GitHub-hosted
greenfield product and Jenkins where the org already has a Jenkins estate, on-prem
runners, or needs custom Groovy logic. The pipeline stages are identical — only
the DSL differs."*

### 9.4 Jenkins agents on Kubernetes (the scaling answer)

Instead of static agents, the Kubernetes plugin spins a **pod per build** from a
pod template (`jnlp` + `docker` + `kubectl` containers), then destroys it. Builds
never pollute each other and you get horizontal scale for free. Mention this —
it's the standard "how do you make Jenkins reliable?" answer.

---

## Phase 10 — Monitoring with Prometheus and Grafana

### 10.1 The mental model (memorise this)

```
application / node-exporter / kube-state-metrics     ← what exposes numbers
        ↓  /metrics (text format)
Prometheus  ── scrapes every 15 s, stores time series (TSDB)
        ├── PromQL queries → Grafana (visualise)
        └── recording/alerting rules → Alertmanager → Slack / PagerDuty / email
```

- **Pull, not push.** Prometheus fetches `/metrics`; the app never phones home.
  Benefits: Prometheus decides what to scrape, and `up == 0` is itself a signal.
- **Labels, not columns.** Every series is `metric_name{label=value}`. That's why
  `sum by (route)` works.
- **Four metric types:** `Counter` (only goes up — requests), `Gauge` (up/down —
  memory), `Histogram` (buckets — latency, gives you `_bucket`, `_sum`, `_count`),
  `Summary` (client-side quantiles).

### 10.2 What the app exposes

`app/backend/src/metrics.js` registers:

| Metric | Type | Labels | Use |
|---|---|---|---|
| `taskflow_http_requests_total` | Counter | `method, route, status_code` | Rate & error-rate (the R in RED) |
| `taskflow_http_request_duration_seconds` | Histogram | `method, route, status_code` | p50/p95/p99 latency |
| `taskflow_tasks_created_total` | Counter | – | Business KPI |
| `taskflow_db_up` | Gauge | – | 1/0 DB reachability |
| `taskflow_*` default metrics | Gauge/Counter | – | event-loop lag, heap, GC, RSS |

### 10.3 How scraping is wired on Kubernetes

The **Prometheus Operator** watches `ServiceMonitor` objects instead of a static
`prometheus.yml`. `kubernetes/monitoring/servicemonitor.yaml` says *"scrape every
Service in namespace `taskflow` labelled `app.kubernetes.io/name=taskflow-api`,
on the `http` port, path `/metrics`, every 15 s."* Add a new service, add a
ServiceMonitor, no Prometheus restart.

```
Service (labelled)  ←  ServiceMonitor (selected by label `release: kube-prometheus-stack`)
        ↑
Prometheus instance (created by the operator, selectorLabels relaxed via
                      serviceMonitorSelectorNilUsesHelmValues: false)
```

### 10.4 Verify scraping works

```bash
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090
# Prometheus → Status ▸ Target health
#   taskflow/taskflow-api/0         UP
#   node-exporter/…                 UP
#   kube-state-metrics/…            UP
```

In the query box, try each of these and understand the output:

```promql
# 1. Requests per second, total
sum(rate(taskflow_http_requests_total[5m]))

# 2. Error ratio (the SLO formula)
sum(rate(taskflow_http_requests_total{status_code=~"5.."}[5m]))
  / sum(rate(taskflow_http_requests_total[5m]))

# 3. p95 latency from a histogram
histogram_quantile(0.95,
  sum by (le) (rate(taskflow_http_request_duration_seconds_bucket[5m])))

# 4. Is the app reachable at all?
up{job=~".*taskflow.*"}

# 5. CPU cores per pod
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="taskflow", container!=""}[5m]))

# 6. Error budget burned in the last hour (99.9% SLO)
1 - (
  sum(rate(taskflow_http_requests_total{status_code=~"5.."}[1h]))
  / sum(rate(taskflow_http_requests_total[1h]))
) / 0.999
```

### 10.5 Grafana

```bash
kubectl -n monitoring port-forward svc/monitoring-grafana 3001:80
# admin / $(kubectl -n monitoring get secret monitoring-grafana -o jsonpath='{.data.admin-password}' | base64 -d)
```

The dashboard `monitoring/grafana/dashboards/taskflow-overview.json` (17 panels)
is auto-imported by the Grafana sidecar. It has five rows:

1. **Golden signals (stat panels)** — request rate, 5xx %, p95 latency, running
   pods, targets up, DB reachable — each with green/orange/red thresholds.
2. **Traffic** — requests/s by route, status-code distribution.
3. **Latency & resources** — p50/p95/p99, CPU per pod, memory per pod.
4. **Kubernetes** — replicas available vs desired, container restarts, RDS CPU.
5. **Nodes** — CPU, memory available, filesystem free.

If you build the dashboard by hand instead, always start from these four
questions: **Is it up? How fast is it? Is it erroring? Is it about to run out
of something?** Everything else is a detail.

### 10.6 Logs, metrics, traces — know the difference

| Signal | This project | What it answers |
|---|---|---|
| Metrics | Prometheus (RED + USE) | *Is something wrong, and how bad?* |
| Logs | `kubectl logs`, CloudWatch Logs (Container Insights), JSON via pino | *What exactly happened?* |
| Traces | Not included — extension: OpenTelemetry + Tempo/Jaeger | *Where in the call chain is it slow?* |

Also mention `kubectl logs -l app=x --tail=100 -f` and structured JSON logging
(`pino` writes one JSON object per line, which is what CloudWatch Logs Insights
and Loki want).

### 10.7 Prometheus retention and storage

`retention: 15d`, `retentionSize: 40GB`, backed by a 50 Gi `gp2` PVC (EBS via the
EBS CSI driver). In production you'd add **Thanos** or **Grafana Mimir** for
long-term storage and a single query view across clusters. Saying that sentence
out loud in an interview shows you know where the toy ends and production begins.

---

## Phase 11 — Alerting

### 11.1 The rules

`monitoring/prometheus/alert-rules.yml` ships 15 rules in four groups
(validated with `promtool check rules`):

| Group | Alerts |
|---|---|
| `taskflow-availability` | `TaskFlowApiDown` (crit), `TaskFlowHighErrorRate` (crit, >5% 5xx for 5m), `TaskFlowLatencyP95High` (warn, >500ms), `TaskFlowDatabaseDown` (crit) |
| `kubernetes-workloads` | `PodCrashLooping`, `PodNotReady`, `DeploymentReplicasMismatch`, `HPAMaxedOut` |
| `infrastructure` | `NodeHighCpu` (>85%), `NodeDiskPressure` (<15% free), `NodeMemoryLow` (<10%), `KubePodPendingTooLong` |
| `aws-rds` | `RDSHighCPU`, `RDSLowFreeStorage`, `RDSConnectionCountHigh` |

Install them the operator-native way — as a `PrometheusRule` CR, so the
Prometheus Operator hot-reloads them with no restart:

```bash
kubectl apply -f kubernetes/monitoring/taskflow-prometheusrule.yaml
kubectl -n monitoring get prometheusrule           # taskflow-alerts
```

The identical rules also exist in `monitoring/prometheus/alert-rules.yml`, which
is what the local docker-compose Prometheus loads (`promtool check rules` passes
on both).

### 11.2 Alert quality — the part most people get wrong

Every alert in this project follows the same discipline:

1. **Symptom-based, not cause-based.** `HighErrorRate` (users hurt) beats
   `CPUHigh` (maybe fine). CPU alerts are warnings; error-rate alerts page.
2. **`for:` duration.** Every rule waits (`for: 5m`) so a single scrape blip
   doesn't wake anyone at 3 a.m.
3. **Severity label drives routing.** `critical` → PagerDuty; `warning` → Slack.
4. **Annotations contain a summary, a description, and a `runbook` link.**
   An alert without a runbook is just noise.
5. **The 5% / 500 ms thresholds come from an SLO**, not from vibes.

### 11.3 Test that alerting actually works

```bash
# Make the API fail readiness (break the DB env var), then watch
kubectl -n taskflow set env deployment/taskflow-api DB_PASSWORD=wrong
kubectl -n taskflow get pods -w        # readiness fails → pods go 0/1
# Prometheus → Alerts → TaskFlowDatabaseDown pending → firing (after 1m)
# Alertmanager → Slack message arrives

kubectl -n taskflow rollout undo deployment/taskflow-api
```

Being able to say *"I broke it on purpose and confirmed the alert fired"* is a
much stronger interview story than *"I configured alerting."*

---

## Phase 12 — Security hardening checklist

Go through this list and be able to explain each line. This is where a
"DevOps learner" becomes "engineer".

### Identity & secrets

- [x] **No static AWS keys anywhere** — GitHub OIDC for CI, **IRSA** for pods
- [x] DB password generated by `random_password`, stored in **Secrets Manager**, KMS-encrypted
- [x] Secrets delivered to pods by **External Secrets Operator**, never committed
- [x] `.gitignore` excludes `.env`, `*.pem`, state files; **gitleaks** pre-commit + Trivy secret scan in CI
- [x] IAM least privilege: each IRSA role is scoped to one ServiceAccount, one namespace, specific resource ARNs
- [x] Bastion has **no SSH port**, no key pair — access via **SSM Session Manager** (audited, no inbound rules)
- [x] **IMDSv2 required** on the bastion (`http_tokens = "required"`) — blocks SSRF credential theft

### Network

- [x] Nodes and RDS in **private subnets**; only the ALB and bastion are public
- [x] One **NAT gateway per AZ** (egress without inbound exposure)
- [x] Security groups chain by SG reference, not CIDR ranges (RDS only accepts
      traffic from the node SG and the bastion SG)
- [x] **VPC Flow Logs** → CloudWatch, 30-day retention
- [x] Kubernetes **default-deny NetworkPolicy** with explicit allows
- [x] RDS `publicly_accessible = false`, `rds.force_ssl = 1`

### Workload

- [x] Pod Security Standards **`restricted`** enforced at the namespace (audit + warn too)
- [x] Containers run **non-root** (`runAsNonRoot`, UID 10001/101), `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, `seccompProfile: RuntimeDefault`
- [x] Resource **requests and limits** on every container (Kyverno enforces)
- [x] **Immutable ECR tags**, no `:latest` in manifests (Kyverno enforces)
- [x] Images **scanned twice**: Trivy in CI (gate) + ECR scan-on-push (safety net)

### Data

- [x] S3: versioning, SSE-KMS with bucket keys, public access blocked,
      TLS-only bucket policy, access logging, lifecycle expiry
- [x] RDS: encrypted storage with KMS, automated backups, deletion protection in prod
- [x] Terraform state: encrypted S3 + DynamoDB locking + versioning

### Detection & audit

- [x] EKS control-plane logging: `api, audit, authenticator, controllerManager, scheduler`
- [x] **EKS Access Entries** (not the legacy `aws-auth` ConfigMap) for cluster RBAC
- [x] Container Insights → CloudWatch for cluster/node metrics and logs
- [x] RDS logs (`postgresql`, `upgrade`) + enhanced monitoring
- [x] `kubectl` access via short-lived tokens from `aws eks get-token`

### The "what's still missing" answer (say this proactively)

> "For true production I'd add: private EKS endpoint + VPN/Transit Gateway,
> AWS WAF on the ALB, GuardDuty + Security Hub with the CIS/EKS benchmarks,
> Trivy Operator for continuous in-cluster scanning, sealed image signing
> (cosign + Kyverno verifyImages), Falco for runtime threat detection, and
> AWS Config rules for drift detection."

Knowing what you *didn't* build is as important as what you did.

---

## Phase 13 — Cost, teardown, and verification

### 13.1 The demo checklist (do this in order, take screenshots)

| # | Screenshot | Command / URL |
|---|---|---|
| 1 | Architecture diagram | this guide, §2 |
| 2 | `terraform apply` summary | terminal |
| 3 | `kubectl get nodes,pods -A` | terminal |
| 4 | Live app in the browser at the ALB URL | ALB DNS |
| 5 | `kubectl get ingress,hpa,pdb` | terminal |
| 6 | CI pipeline graph, all green | GitHub Actions |
| 7 | Trivy image scan table | Actions log |
| 8 | SonarQube dashboard + quality gate PASSED | SonarCloud |
| 9 | Grafana golden-signals dashboard with real traffic | port-forward |
| 10 | Prometheus alerts page (one firing, one pending) | :9090/alerts |
| 11 | `kubectl get externalsecret` → `SecretSynced` | terminal |
| 12 | A failed pipeline (broken test) then green again | Actions |
| 13 | AWS console: RDS, ALB, S3, IAM roles | console |
| 14 | Cost Explorer or Budgets widget | console |

### 13.2 Teardown — order matters

```bash
./scripts/05-teardown.sh dev
# 1. kubectl delete -k  (the Ingress deletion is what de-provisions the ALB;
#                        if you skip it, VPC deletion fails with DependencyViolation)
# 2. helm uninstall monitoring (frees the EBS volume behind Prometheus)
# 3. terraform destroy (RDS first, then nodes, then VPC)
```

Then sweep for orphans:

```bash
aws ec2 describe-volumes --filters Name=status,Values=available --query 'Volumes[].VolumeId'
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'
aws ec2 describe-nat-gateways --filter Name=state,Values=available --query 'NatGateways[].NatGatewayId'
aws ec2 describe-addresses --query 'Addresses[].AllocationId'   # unassociated EIPs cost money
aws logs describe-log-groups --query 'logGroups[].logGroupName'
aws ecr describe-repositories --query 'repositories[].repositoryName'
```

Anything still listed is still billing you. Delete it (EIPs are $3.6/mo each and
easy to forget).

### 13.3 Cost-control habits worth stating in an interview

- Tag everything (`Project`, `Environment`, `Owner`, `ManagedBy`) — the
  `default_tags` block in `providers.tf` does this for every resource.
- AWS Budget + alarm before you build.
- `db.t3.micro` + single-AZ in dev, Multi-AZ only in prod.
- One NAT in dev instead of two if you're cost-sensitive (this project uses two
  for AZ resilience — say which trade-off you chose and why).
- ECR lifecycle policy keeps 10 images, not 500.
- S3 lifecycle expires old builds after 90 days.
- **`terraform destroy` is the best cost optimiser.**

---

## Phase 14 — GitOps with Argo CD

### 14.1 Why change something that works?

The pipeline in Phase 8 is **push-based**: GitHub Actions holds cluster
credentials and runs `kubectl apply`. That works, but it has three structural
problems:

| Problem | Consequence |
|---|---|
| CI needs cluster-admin | A compromised workflow = a compromised cluster |
| No reconciliation | Someone `kubectl edit`s a deployment; Git says one thing, the cluster does another, forever |
| No audit of *state* | Git history shows intent, not what actually ran |

**GitOps inverts the direction:** a controller *inside* the cluster watches Git
and continuously reconciles. Push-based CD is a deployment event; GitOps is a
deployment *state*.

```
Push-based (Phase 8)              GitOps (Phase 14)
CI ──kubectl apply──► cluster     dev ──git push──► repo ◄──pull── Argo CD
     (needs creds)                                            │
                                              reconcile loop │
                                                              ▼
                                                            cluster
```

### 14.2 Install it

```bash
# Prereqs: repo URL + a GitHub PAT (repo scope) exported in the shell
export GITHUB_PAT=ghp_xxx
./scripts/07-install-gitops.sh
```

The script installs Argo CD (HA: 2 server + 2 repo-server + 2 appset replicas),
the Argo Rollouts controller and dashboard, registers the Git repo, and applies
the `AppProject` + two `ApplicationSet`s.

```bash
kubectl -n argocd get pods
kubectl -n argocd port-forward svc/argocd-server 8081:80   # → http://localhost:8081
```

### 14.3 The four objects that matter

**1. `AppProject`** (`gitops/argocd/projects/taskflow-project.yaml`) — a security
boundary, not a folder. It whitelists which **repos** may be read, which
**namespaces/clusters** may be written, and which **cluster-scoped kinds** are
allowed. Without it, any `Application` could deploy anywhere.

**2. `ApplicationSet`** (`gitops/argocd/appsets/taskflow-appset.yaml`) — a
template that generates one `Application` per environment from a list generator:

| Env | Branch | Auto-sync | Prune | Self-heal |
|---|---|---|---|---|
| dev | `develop` | ✅ | ✅ | ✅ |
| prod | `main` | ❌ (deliberate) | ❌ | ✅ |

> **Prod is not auto-synced on purpose.** Syncing prod should be a deliberate
> promotion (a PR that changes the image tag, merged by a human), not a side
> effect of a merge. You still get drift healing — hand-edits are reverted.

**3. `ignoreDifferences`** — the two fields Argo must *not* fight over:

```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers: [/spec/replicas]     # the HPA owns this
  - group: ""
    kind: Secret
    name: taskflow-db-credentials
    jsonPointers: [/data]              # External Secrets owns this
```

Forget these and you get the classic GitOps bug: Argo scales your app back to
`replicas: 2` every three minutes while the HPA tries to scale it up.

**4. Sync waves + multi-source** (`platform-appset.yaml`) — platform add-ons are
installed *by Argo*, in dependency order, with the Helm chart coming from the
upstream repo and the values file coming from **your** repo:

```yaml
sources:
  - repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 61.3.0
    helm:
      valueFiles: [$values/monitoring/prometheus/values-kube-prometheus-stack.yaml]
  - repoURL: https://github.com/you/taskflow-devops.git
    targetRevision: main
    ref: values                      # ← referenced as $values above
```

Waves: `external-secrets` (0) → `kyverno` + `argo-rollouts` (1) → `monitoring` (2)
→ `taskflow` app (10). CRDs exist before anything tries to use them.

### 14.4 The workflow from now on

```bash
git checkout develop
git commit -am "feat: new filter on the task list"
git push                     # CI builds + pushes the image, bumps the tag, commits back
# Argo CD notices within 3 minutes (or instantly with a webhook) and syncs
```

Enable the webhook so you don't wait: **GitHub repo → Settings → Webhooks → add**
`https://argocd.internal.example.com/api/webhook` (content type `application/json`).

### 14.5 Demo 1 — prove drift healing works

```bash
kubectl -n taskflow scale deploy/taskflow-web --replicas=1
kubectl -n argocd get app taskflow-dev \
  -o jsonpath='{.status.sync.status}{"\n"}'      # OutOfSync
# wait ~3 minutes
kubectl -n taskflow get deploy taskflow-web      # back to 2 — nobody touched it
```

That single command pair is worth more in an interview than a paragraph about
"declarative infrastructure".

---

## Phase 15 — Progressive delivery with Argo Rollouts on ALB

### 15.1 Canary vs blue-green vs rolling

| Strategy | Traffic split | Rollback cost | Needs |
|---|---|---|---|
| Rolling update (Phase 6) | Gradual pod-by-pod, but **not** controllable by weight | Restart old ReplicaSet | Nothing |
| Blue/green | 0% → 100% at once | Instant (flip back) | 2× capacity |
| **Canary** | 10% → 30% → 60% → 100% | Instant (weight → 0) | Weighted traffic splitting |

Canary is the sweet spot: real users exercise the new version on a small slice of
traffic, and Prometheus decides whether it survives.

### 15.2 Why Argo Rollouts and not Flagger

Both do canary + automated analysis. On AWS the deciding factor is **native ALB
support**:

- **Argo Rollouts** → `trafficRouting.alb` patches the ALB controller's action
  annotation → weighted target groups. **No service mesh required.**
- **Flagger** → excellent, but on EKS it needs Istio / Linkerd / App Mesh /
  NGINX ingress. That's 6–10 extra pods, extra CPU, and an extra thing to learn.

You already have an ALB. Use it.

### 15.3 How ALB weight splitting actually works

Three pieces have to line up:

```
Ingress (gitops/rollouts/ingress-canary.yaml)
  path /api → service taskflow-api-root, port: use-annotation
                        │
                        ▼
  ALB controller reads annotation:
  alb.ingress.kubernetes.io/actions.taskflow-api-root:
    {"Type":"forward","ForwardConfig":{"TargetGroups":[
        {"ServiceName":"taskflow-api-stable","ServicePort":"3000","Weight":90},
        {"ServiceName":"taskflow-api-canary","ServicePort":"3000","Weight":10}]}}
                        ▲
                        │ patched continuously by
  Rollout (gitops/rollouts/rollout-api.yaml)
    trafficRouting.alb: { ingress: taskflow-api-canary, rootService: taskflow-api-root }
```

Argo Rollouts owns the weights; the ALB controller owns the target groups. The
`group.name: taskflow` annotation makes this Ingress share the ALB created in
Phase 6 instead of provisioning (and billing for) a second one.

### 15.4 The rollout steps

```yaml
steps:
  - setWeight: 10
  - pause: { duration: 2m }
  - analysis: { templates: [{ templateName: taskflow-success-rate }] }
  - setWeight: 30
  - pause: { duration: 3m }
  - analysis: { templates: [{ templateName: taskflow-latency }] }
  - setWeight: 60
  - pause: { duration: 3m }
  - setWeight: 100
```

`maxSurge: 25%` / `maxUnavailable: 0` means no capacity dip during the shift.
`revisionHistoryLimit: 5` keeps five previous ReplicaSets for instant rollback.

### 15.5 The automated analysis — the part that makes it real

`gitops/rollouts/analysistemplates.yaml` defines three Prometheus queries that
act as gates:

| Template | Query result must satisfy | `failureLimit` |
|---|---|---|
| `taskflow-success-rate` | ≥ 0.95 (95% non-5xx) | 2 failures → abort |
| `taskflow-latency` | p95 < 0.5 s | 2 failures → abort |
| `taskflow-db-health` | `taskflow_db_up` ≥ 1 | 1 failure → abort |

Each runs every 60 s for 5 samples. One bad sample is tolerated (datapoints are
noisy); two in a row aborts the rollout and returns traffic to 100% stable.

> **The queries filter on `pod=~"taskflow-api-.*"`, not on `job=`.** Prometheus
> Operator versions label scraped series differently (`job` may be the
> ServiceMonitor name, `<ns>/<name>/0`, or something else). The pod name prefix
> is derived from the Rollout name and is stable everywhere. This is exactly the
> kind of detail that separates a working pipeline from a broken one.

### 15.6 Run the demos

```bash
# Demo 2 — a real canary
./scripts/08-canary-demo.sh
kubectl argo rollouts get rollout taskflow-api -n taskflow --watch
```

Watch the terminal output: a new ReplicaSet appears, weights climb 10 → 30 → 60
→ 100, and the AnalysisRun reports `Successful` at each gate.

```bash
# Demo 3 — automatic rollback, no human involved
kubectl argo rollouts set image taskflow-api -n taskflow \
  api="$ECR_BACKEND:does-not-exist"
kubectl argo rollouts get rollout taskflow-api -n taskflow --watch
# pods ImagePullBackOff → success-rate analysis fails → status ABORTED
# ALB weight returns to 100% stable. Users never noticed.
```

Manual controls you should know:

```bash
kubectl argo rollouts promote taskflow-api -n taskflow   # skip the next pause
kubectl argo rollouts abort   taskflow-api -n taskflow   # bail out now
kubectl argo rollouts undo    taskflow-api -n taskflow   # back to previous revision
kubectl argo rollouts dashboard                          # web UI on :3100
```

### 15.7 Migrating from Deployment to Rollout (the cutover)

You cannot run a `Deployment` and a `Rollout` for the same pods. Move in this
order:

```bash
# 1. Create the extra services and the rollout (do NOT delete the Deployment yet)
kubectl apply -f gitops/rollouts/services-stable-canary.yaml
kubectl apply -f gitops/rollouts/analysistemplates.yaml
kubectl apply -f gitops/rollouts/rollout-api.yaml

# 2. Point the HPA at the Rollout
kubectl apply -f gitops/rollouts/hpa-rollout.yaml

# 3. Switch the Ingress from the plain Service to the weighted one
kubectl apply -f gitops/rollouts/ingress-canary.yaml
#    and remove the /api path from kubernetes/base/ingress.yaml (already noted there)

# 4. Drain and remove the old Deployment
kubectl -n taskflow delete deployment taskflow-api
```

Once this is in Git, Argo CD manages the Rollout exactly like it managed the
Deployment — the `ignoreDifferences` entry for `argoproj.io/Rollout` in the
ApplicationSet already covers the HPA-vs-Argo ownership conflict.

### 15.8 Interview questions on the new material

33. **"Why GitOps over CI pushing to the cluster?"** — The cluster pulls; CI
    holds no cluster credentials; drift is detected and corrected continuously;
    Git is the audit log of *what is deployed*, not just what was requested.
34. **"What is an ApplicationSet and when would you not use it?"** — A generator
    that produces Applications from a template (list/cluster/git/PR generators).
    Don't use it for two wildly different apps — a plain Application is clearer.
35. **"Argo CD says OutOfSync but nothing changed."** — Usually `ignoreDifferences`
    is missing for a field another controller owns (HPA replicas, ESO secret
    data, cert-manager annotations), or a defaulted field differs. Read the diff
    in the UI; don't blind-sync.
36. **"Compare Argo Rollouts with Flagger."** — Both implement canary with metric
    analysis. Rollouts has native ALB/NGINX/Gateway-API support and a great CLI;
    Flagger is mesh-first and integrates tightly with Istio/Linkerd. On AWS with
    an ALB, Rollouts means zero extra infrastructure.
37. **"What stops a canary from taking down prod?"** — Weighted traffic (max 60%
    before full promotion), `maxUnavailable: 0`, three Prometheus gates with
    `failureLimit`, automatic abort → weight 0, and the previous ReplicaSet still
    warm for instant rollback.
38. **"How do you roll back a bad release in 10 seconds?"** — With Rollouts:
    `kubectl argo rollouts abort` (or let the analysis do it). With Rollouts +
    GitOps: revert the git commit; Argo reconciles. Both beat rebuilding an image.

---

## 20. Interview questions you must be able to answer

Grouped by area. Answer out loud before reading the hint.

### Terraform

1. **What is state and why is it in S3?** — State is the mapping from code to
   real resources, plus attribute caching and dependency metadata. S3 gives
   durability, versioning, and team sharing; DynamoDB gives locking so two
   engineers can't corrupt it concurrently.
2. **`plan` vs `apply` — and why `-out=tfplan`?** — Plan compares desired (code)
   with actual (state + refresh). Saving the plan and applying *that file* means
   you deploy exactly what was reviewed.
3. **What's the difference between `count` and `for_each`?** — `count` indexes by
   position, so removing a middle element shifts everything and causes churn;
   `for_each` keys by name, so removals are surgical. Prefer `for_each`.
4. **How do you handle secrets in Terraform?** — Never hardcode. `random_password`
   → Secrets Manager; mark variables `sensitive`; state is encrypted at rest;
   in real setups use dynamic DB secrets or SSM parameter references.
5. **How would you import existing resources?** — `import` blocks (Terraform ≥1.5)
   or the deprecated `terraform import` CLI, then reconcile the config until the
   plan is empty.
6. **Module structure?** — Root composes small single-purpose modules; environments
   are directories with their own var-file and backend key.

### Docker

7. **Multi-stage build — why?** — Build toolchain and dev dependencies never reach
   the runtime image: smaller image, fewer CVEs, faster pulls.
8. **Why `dumb-init`?** — PID 1 in a container must reap zombies and forward
   signals. Without it, `SIGTERM` from `kubectl delete pod` may be ignored and
   you get a 30-second SIGKILL instead of a graceful drain.
9. **How is the image kept small?** — Alpine base, `npm ci --omit=dev`, `.dockerignore`,
   no source maps, multi-stage.
10. **Why non-root?** — Container escape as root is far more dangerous; PSS
    `restricted` requires it, and many corporate policies mandate it.

### Kubernetes

11. **Liveness vs readiness vs startup probe.** — See §6.7.
12. **How does a rolling update achieve zero downtime?** — `maxUnavailable: 0`,
    `maxSurge: 1`, readiness gates traffic, `preStop: sleep 5` lets the ALB
    drain the endpoint, `terminationGracePeriodSeconds: 45`.
13. **HPA needs what?** — metrics-server, resource requests set on containers,
    and a controller loop hitting the API. `kubectl top pods` failing = HPA blind.
14. **What does a PDB do?** — Guarantees a minimum available replica count during
    *voluntary* disruptions (node drains, cluster upgrades). It does not protect
    against involuntary ones.
15. **How does an ALB get created from an Ingress?** — The AWS Load Balancer
    Controller watches Ingress objects, reads the `alb.ingress.kubernetes.io/*`
    annotations, and calls the ELBv2 API to create the ALB, listeners, and target
    groups, registering pod IPs as targets.
16. **IRSA — how does it work?** — The pod's ServiceAccount token is projected
    into the pod; AWS SDK exchanges it with STS (`AssumeRoleWithWebIdentity`)
    against the cluster's OIDC provider; the role's trust policy checks
    `sub = system:serviceaccount:<ns>:<sa>`. No credentials on disk.
17. **Rollback.** — `kubectl rollout undo deployment/x`, or GitOps revert. Also
    explain that `revisionHistoryLimit: 5` keeps old ReplicaSets around.

### AWS

18. **Public vs private subnet.** — Route table to an IGW vs to a NAT gateway.
    ALB in public, nodes/DB in private.
19. **Why a NAT gateway per AZ?** — A single NAT means a cross-AZ dependency and
    data charges; per-AZ NAT keeps failure domains isolated.
20. **ALB vs NLB vs CLB.** — ALB: L7, path/host routing, WAF, HTTP/2, WebSockets.
    NLB: L4, extreme performance, static IPs, preserves source IP. CLB: legacy.
21. **RDS Multi-AZ vs read replica.** — Multi-AZ is a synchronous standby for
    *availability* (failover, same endpoint). A read replica is asynchronous for
    *scale* (you must point reads at its endpoint).
22. **S3 security for a private app bucket.** — Block all public access, SSE-KMS,
    TLS-only bucket policy, versioning, access logging, least-privilege IAM.
23. **How does CI get into the cluster without keys?** — GitHub OIDC → IAM role →
    `aws eks update-kubeconfig` → the role is in an EKS **access entry** with
    `AmazonEKSClusterAdminPolicy`, so the token it gets is authorised in RBAC.

### CI/CD & observability

24. **What gates a merge in your pipeline?** — Lint, unit tests, Sonar quality
    gate, Trivy HIGH/CRITICAL, required review. Prod deploy additionally needs a
    GitHub Environment approval.
25. **How do you roll back a bad deploy?** — `kubectl rollout undo`, or re-deploy
    the previous immutable image tag; automatic rollback step on smoke-test
    failure; GitOps revert (Argo CD) as the mature answer.
26. **Counter vs Gauge vs Histogram.** — See §10.1. Histograms are what make
    `histogram_quantile` (p95/p99) possible.
27. **Why is `up == 0` a good alert but `CPU > 80%` a noisy one?** — `up` is a
    direct user-impact symptom; CPU >80% might be perfectly healthy utilisation.
28. **What's an SLO and how does it relate to alerting?** — An SLO is a target
    (99.9% of requests succeed over 30 days). Error budget = 1 − SLO. Alerts fire
    on **burn rate** (budget being consumed too fast), not on raw thresholds.
29. **Metrics vs logs vs traces.** — See §10.6.

### Behavioural / design

30. **"Walk me through what happens when a developer pushes code."** — PR → CI
    (test, sonar, trivy) → merge to develop → images built and pushed with the
    git SHA as an immutable tag → Kustomize sets that tag → `kubectl apply` →
    rollout waits on readiness → smoke test hits the ALB → Prometheus picks up the
    new pods → Grafana shows the new revision and alert rules watch the error
    rate. On `main`, the same thing but behind a manual approval.
31. **"What would you change to make this production-grade?"** — Private EKS
    endpoint, WAF, Multi-AZ RDS + read replica, Argo CD (GitOps) instead of
    imperative `kubectl apply`, blue/green or canary with Flagger, Thanos for
    long-term metrics, ExternalDNS, cert-manager, backups tested via restore
    drills, and a documented on-call runbook per alert.
32. **"Something's broken in prod at 2 a.m. What do you do?"** — Check the
    Grafana golden signals → identify whether it's app, DB, or node → read the
    firing alert's runbook → `kubectl get pods` / `describe` / `logs` → if it
    correlates with a recent deploy, roll back first and investigate after →
    write the postmortem with a timeline and action items.

---

## 21. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `terraform init` → "Error acquiring the state lock" | A previous run crashed | `terraform force-unlock <LOCK_ID>` (only after confirming nobody is running) |
| `terraform destroy` → `DependencyViolation` on the VPC | ALB / ENIs still exist | Delete the Ingress first, wait 2–3 min for the ALB controller to clean up, then retry |
| `kubectl` says "You must be logged in to the server (Unauthorized)" | Your IAM principal isn't in an EKS access entry | `aws eks create-access-entry` + associate `AmazonEKSClusterAdminPolicy`, or re-apply Terraform |
| Pods `ImagePullBackOff` | Wrong ECR URL, or node role lacks ECR read | Check `kubectl describe pod` events; verify `AmazonEC2ContainerRegistryReadOnly` on the node role |
| Pods `CrashLoopBackOff` | App error, missing env var, DB unreachable | `kubectl logs --previous`, `kubectl describe` |
| `ExternalSecret` → `SecretSyncedError` | IRSA trust policy mismatch (namespace/SA name) | Compare `system:serviceaccount:taskflow:taskflow-api` with the role's `sub` condition |
| Ingress has no ADDRESS after 5 min | ALB controller can't call AWS, or subnets lack the ELB role tag | `kubectl -n kube-system logs deploy/aws-load-balancer-controller`; check `kubernetes.io/role/elb=1` on public subnets |
| ALB returns 503 | No healthy targets | Health check path/port wrong, or readiness failing; check target group health in the console |
| ALB returns 504 | Backend too slow / security group blocking | Check SG rules on the node SG (ALB must reach pod port), and app latency |
| `HPA` shows `<unknown>` for targets | metrics-server missing or no resource requests | Install metrics-server; ensure `resources.requests` are set |
| Grafana "No data" | ServiceMonitor label mismatch, or scrape target down | Check Prometheus → Targets; the ServiceMonitor needs `release: kube-prometheus-stack` |
| `terraform apply` slow / times out on EKS add-ons | Node group still joining | Wait, then re-apply; add-on install is retried by Terraform |
| `Error: creating RDS: DBSubnetGroupDoesNotCoverEnoughAZs` | Only one private subnet | Use ≥2 private subnets in different AZs |
| Sonar gate fails on coverage | New code not covered | Add tests; or set the coverage threshold to what's realistic *and* ratchet it up over time |
| Trivy fails on a base-image CVE | Alpine/Debian package not yet patched | Rebuild with an updated base, or add a time-boxed `.trivyignore` entry with an owner |

---

## 22. Where to take it next

Ordered by interview value per hour of work:

1. ~~Argo CD (GitOps)~~ — **already built, see Phase 14.**
2. ~~Canary with automated analysis~~ — **already built, see Phase 15**
   (Argo Rollouts on ALB, not Flagger — no service mesh required).
3. **Trivy Operator** — continuous in-cluster vulnerability scanning of running
   workloads (not just at build time).
4. **cert-manager + ExternalDNS** — automatic TLS and automatic DNS for the ALB.
5. **Loki + Promtail** — centralised log search alongside your metrics.
6. **OpenTelemetry + Tempo/Jaeger** — distributed tracing through the frontend →
   API → DB chain.
7. **Chaos engineering** — `kubectl delete pod`, drain a node, kill the RDS
   failover, and verify your alerts and PDBs behave.
8. **Postmortem doc** — write a real one for a failure you deliberately caused.
   This single document impresses interviewers more than another dashboard.

---

## Appendix A — Quick reference commands

```bash
# ---- AWS ----
aws sts get-caller-identity
aws eks update-kubeconfig --name taskflow-dev --region us-east-1
aws ecr get-login-password --region us-east-1 | docker login --username AWS \
  --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com"
aws ssm start-session --target "$(terraform -chdir=terraform output -raw bastion_instance_id)"

# ---- Terraform ----
terraform init -backend-config=environments/dev/backend.hcl -reconfigure
terraform plan -var-file=environments/dev/terraform.tfvars
terraform apply tfplan
terraform state list
terraform state show 'module.eks.aws_eks_cluster.this'
terraform output -json | jq

# ---- kubectl ----
kubectl get all -n taskflow
kubectl -n taskflow describe pod <pod>
kubectl -n taskflow logs deploy/taskflow-api --tail=100 -f
kubectl -n taskflow exec -it deploy/taskflow-api -- sh
kubectl -n taskflow rollout status deploy/taskflow-api
kubectl -n taskflow rollout undo deploy/taskflow-api
kubectl -n taskflow get events --sort-by=.lastTimestamp
kubectl top pods -n taskflow
kubectl -n taskflow port-forward svc/taskflow-api 3000:3000

# ---- troubleshooting a stuck pod, in order ----
kubectl -n taskflow get pods
kubectl -n taskflow describe pod <pod>          # read Events at the bottom
kubectl -n taskflow logs <pod> --previous       # if it crashed
kubectl -n taskflow get events --sort-by=.lastTimestamp | tail -20

# ---- port-forwards you'll use a lot ----
kubectl -n monitoring port-forward svc/monitoring-grafana 3001:80
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-alertmanager 9093:9093
```

## Appendix B — What each file proves (mapping for your portfolio README)

| Claim | Evidence in this repo |
|---|---|
| "I can write modular Terraform" | `terraform/modules/*` with inputs/outputs, `default_tags`, validation blocks |
| "I manage state properly" | `terraform/backend.tf` + `scripts/01-bootstrap-tfstate.sh` |
| "I build secure containers" | `app/*/Dockerfile` (multi-stage, non-root, dumb-init, healthcheck) |
| "I know Kubernetes beyond `kubectl apply`" | HPA, PDB, NetworkPolicy, probes, PSS, IRSA, topology spread, preStop |
| "I understand ALB integration" | `kubernetes/base/ingress.yaml` annotations + `modules/vpc` subnet tags |
| "I do CI/CD properly" | `.github/workflows/ci-cd.yml`, `ci/Jenkinsfile`, OIDC, environments |
| "I shift security left" | `security/*`, Kyverno policies, Trivy in 4 modes, gitleaks pre-commit |
| "I can operate what I build" | `monitoring/*` — dashboards, 15 alert rules, routing, runbook links |
| "I do GitOps" | `gitops/argocd/*` — AppProject, ApplicationSet, sync waves, ignoreDifferences, drift self-heal demo |
| "I do progressive delivery" | `gitops/rollouts/*` — canary with ALB weight shifting + 3 Prometheus analysis gates |
| "I clean up after myself" | `scripts/05-teardown.sh` with correct ordering |

## Appendix C — Validation already performed on this repo

Everything below was executed against the files in this repository, so you are
not debugging my typos — you are learning the platform.

| Check | Command | Result |
|---|---|---|
| Terraform syntax + provider wiring | `terraform init -backend=false && terraform validate` | **valid** |
| Terraform formatting | `terraform fmt -recursive -check` | **clean** |
| Kubernetes manifests (dev + prod overlays) | `kustomize build kubernetes/overlays/dev \| kubeconform` | **20 resources, 0 invalid** |
| GitOps + Rollout manifests | `kubeconform gitops/rollouts/*.yaml` | **13 resources, 0 invalid** (8 CRDs skipped) |
| Prometheus alert rules | `promtool check rules` | **SUCCESS: 15 rules** |
| Grafana dashboard JSON | `json.load` + schema keys | **17 panels** |
| Backend unit tests | `npx jest --coverage` | **17 passed, 94.6% statements** |
| YAML well-formedness | `yaml.safe_load_all` over every `.yaml`/`.yml` | **31 files, 0 failures** |
| Docker Compose | `docker compose config` (run it yourself; Docker isn't needed for the rest) | expected clean |

### What is intentionally *not* verified here

- `terraform apply` — needs your AWS account and would cost money.
- Image builds / ECR pushes — need your registry.
- The live ALB, RDS, and Grafana URLs — those only exist after you deploy.

### The 60-second "does my work count" test

```bash
cd taskflow-devops
./scripts/00-verify.sh            # add --tests to also run the backend suite
```

It downloads terraform/kustomize/kubeconform/promtool into a **temp directory**
(deleted on exit — nothing is written into your repo), then runs six checks:
YAML well-formedness, kustomize build + kubeconform for both overlays, the GitOps
manifests, `promtool check rules`, the Grafana dashboard, and terraform
`fmt`/`validate`. Six green checkmarks, and you have proof the repo is real before
you spend a single dollar on AWS.

---

*Built as a learning vehicle for AWS + DevOps + SRE fundamentals. Fork it, break
it, fix it, and put it on your CV.*
