# DevOps Portfolio

A collection of infrastructure and platform engineering projects built on AWS.
Each project is maintained in a private repository — this page documents the architecture,
design decisions, and tech stack for interview and review purposes.

---

## Projects

| # | Project | Repo | Status |
|---|---|---|---|
| 1 | [AWS VPC & IAM Infrastructure](#1-aws-vpc--iam-infrastructure) | [project_vpc_iam](https://github.com/Ajay0696/project_vpc_iam) (private) | Complete |
| 2 | [EKS Platform (LB, DNS, S3)](#2-eks-platform-lb-dns-s3) | [project_app_infra](https://github.com/Ajay0696/project_app_infra) (private) | Planned |
| 3 | [ArgoCD GitOps](#3-argocd-gitops) | TBD | Planned |
| 4 | [Observability Stack (Loki, Fluent Bit, Thanos)](#4-observability-stack) | TBD | Planned |

---

## 1. AWS VPC & IAM Infrastructure

Provisions the base network and IAM foundation on AWS using Terraform, with a fully
automated CI/CD pipeline running on a local Kubernetes cluster.

**Repo:** [github.com/Ajay0696/project_vpc_iam](https://github.com/Ajay0696/project_vpc_iam) (private)

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Local Machine (k3d cluster)                            │
│                                                         │
│  ┌─────────────────┐     ┌──────────────────────────┐  │
│  │  ARC Controller │────▶│  Runner Pod (ephemeral)   │  │
│  │  (watches GH)   │     │  - Terraform pre-installed│  │
│  └─────────────────┘     └────────────┬─────────────┘  │
└───────────────────────────────────────┼─────────────────┘
                                        │ GitHub OIDC token
                                        ▼
                              ┌──────────────────┐
                              │   AWS STS        │
                              │  (verify token,  │
                              │  return creds)   │
                              └────────┬─────────┘
                                       │ Temporary credentials (1hr)
                                       ▼
                              ┌──────────────────┐
                              │   AWS            │
                              │  VPC, Subnets,   │
                              │  IGW, Route Tables│
                              └──────────────────┘
```

### Tech Stack

| Component | Technology |
|---|---|
| Infrastructure as Code | Terraform (~> 6.0 AWS provider) |
| State Backend | S3 (`use_lockfile = true`, no DynamoDB) |
| Bootstrap | AWS CloudFormation |
| CI/CD | GitHub Actions (manual `workflow_dispatch`) |
| Self-hosted Runners | Actions Runner Controller (ARC) on k3d |
| AWS Authentication | GitHub OIDC (no stored credentials) |
| Runner Auth to GitHub | GitHub App (production standard over PAT) |

### Key Design Decisions

**Chicken-egg problem for S3 state backend**
Terraform needs an S3 bucket to store state, but it can't create the bucket without
state. Solved by using CloudFormation to bootstrap the S3 buckets — CloudFormation has
no remote backend dependency.

**GitHub OIDC instead of stored AWS credentials**
Runner pods never hold long-lived AWS credentials. GitHub issues a short-lived OIDC
token to each job; AWS STS verifies it against the registered OIDC provider and returns
temporary credentials (1hr TTL). If the cluster is compromised, there are no credentials
to steal.

**ARC on local k3d instead of EC2**
Using EC2 runners would require the VPC to exist first — another chicken-egg problem.
Running ARC on a local k3d cluster removes that dependency entirely. Runners scale to
zero when idle (min=0, max=3) and terminate after each job.

**Multi-module workflow with per-environment state keys**
All three workflows (plan/apply/destroy) have a module selector dropdown. Each root
module has its own `providers.tf` with a unique state key following the convention
`<resource>/<environment>/terraform.tfstate`:

```
vpc-iam-state-bucket/
  vpc/mgmt/terraform.tfstate
  vpc/prod/terraform.tfstate    (future)
  iam/mgmt/terraform.tfstate    (future)
```

### Resources Provisioned

**VPC (`mgmt_vpc_creation` module)**

| Resource | Name |
|---|---|
| VPC | `prod-us-east-1-ajayshandbook-vpc` |
| Internet Gateway | `prod-us-east-1-ajayshandbook-igw` |
| Public Subnets (×2) | `prod-us-east-1-ajayshandbook-public-subnet-subnet-a/b` |
| Private Subnets (×2) | `prod-us-east-1-ajayshandbook-private-subnet-subnet-a/b` |
| Public Route Table | `prod-us-east-1-ajayshandbook-public-rt` |
| Private Route Table | `prod-us-east-1-ajayshandbook-private-rt` |
| NAT Gateway | `prod-us-east-1-ajayshandbook-nat-gw` |
| NAT EIP | `prod-us-east-1-ajayshandbook-nat-eip` |

**Network layout:** `10.0.0.0/16` VPC, public subnets in `us-east-1a/b`, private subnets
in `us-east-1a/b` (disabled by default, toggled via `create_private_subnets` variable).

### CI/CD Workflows

| Workflow | Trigger | What it does |
|---|---|---|
| Terraform Plan | Manual | Init → fmt check → validate → plan, uploads artifact |
| Terraform Apply | Manual | Init → apply |
| Terraform Destroy | Manual (requires typing `destroy` to confirm) | Init → destroy |

---

## 2. EKS Platform (LB, DNS, S3)

*Coming soon*

**Repo:** [github.com/Ajay0696/project_app_infra](https://github.com/Ajay0696/project_app_infra) (private)

Planned scope:
- EKS cluster provisioned via Terraform
- AWS Load Balancer Controller
- ExternalDNS with Route53
- S3 buckets for application storage

---

## 3. ArgoCD GitOps

*Coming soon*

Planned scope:
- ArgoCD installed on EKS
- GitOps-driven application deployments
- App-of-apps pattern

---

## 4. Observability Stack

*Coming soon*

Planned scope:
- **Loki** — log aggregation
- **Fluent Bit** — log shipping from pods to Loki
- **Thanos** — long-term Prometheus metrics storage
- Grafana dashboards
