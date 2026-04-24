# DevOps Portfolio

A collection of infrastructure and platform engineering projects built on AWS.
Each project is maintained in a private repository — this page documents the architecture,
design decisions, and tech stack for interview and review purposes.

---

## Projects

| # | Project | Repo | Status |
|---|---|---|---|
| 1 | [AWS VPC & IAM Infrastructure](#1-aws-vpc--iam-infrastructure) | [project_vpc_iam](https://github.com/Ajay0696/project_vpc_iam) (private) | Complete |
| 2 | [EKS Platform](#2-eks-platform) | [project_app_infra](https://github.com/Ajay0696/project_app_infra) (private) | In Progress |
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
`<resource>/<environment>/terraform.tfstate`.

### Resources Provisioned

| Resource | Name |
|---|---|
| VPC | `prod-us-east-1-ajayshandbook-vpc` |
| Internet Gateway | `prod-us-east-1-ajayshandbook-igw` |
| Public Subnets (×2) | `prod-us-east-1-ajayshandbook-public-subnet-a/b` |
| Private Subnets (×2) | `prod-us-east-1-ajayshandbook-private-subnet-a/b` |
| Public Route Table | `prod-us-east-1-ajayshandbook-public-rt` |
| Private Route Table | `prod-us-east-1-ajayshandbook-private-rt` |

**Network layout:** `10.0.0.0/16` VPC, public and private subnets in `us-east-1a/b`.
Private subnets and NAT gateway are toggled via `create_private_subnets` variable
(disabled by default to avoid NAT costs during practice).

### CI/CD Workflows

| Workflow | Trigger | What it does |
|---|---|---|
| Terraform Plan | Manual | Init → fmt check → validate → plan, uploads artifact |
| Terraform Apply | Manual | Init → apply |
| Terraform Destroy | Manual (requires typing `destroy` to confirm) | Init → destroy |

---

## 2. EKS Platform

Provisions a production-pattern EKS cluster on AWS using raw Terraform resources
(no community module). Includes node pools for ingress and platform workloads,
Karpenter for dynamic scaling of developer workloads, and Calico for CNI.

**Repo:** [github.com/Ajay0696/project_app_infra](https://github.com/Ajay0696/project_app_infra) (private)

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
                                        │ GitHub OIDC → AWS STS
                                        ▼
                    ┌───────────────────────────────────┐
                    │  AWS EKS (mgmt cluster)           │
                    │                                   │
                    │  ┌─────────────┐  ┌───────────┐  │
                    │  │ traefikext  │  │ traefikint│  │
                    │  │ t3a.small   │  │ t3a.small │  │
                    │  │ spot        │  │ spot      │  │
                    │  └─────────────┘  └───────────┘  │
                    │                                   │
                    │  ┌──────────────────────────────┐ │
                    │  │ clusterworkloads              │ │
                    │  │ t3a.medium × 2 (spot)        │ │
                    │  │ Loki · Grafana · Prometheus  │ │
                    │  │ Karpenter controller         │ │
                    │  └──────────────────────────────┘ │
                    │                                   │
                    │  ┌──────────────────────────────┐ │
                    │  │ Karpenter-managed nodes       │ │
                    │  │ (dynamically provisioned for  │ │
                    │  │  developer workloads)         │ │
                    │  └──────────────────────────────┘ │
                    └───────────────────────────────────┘
```

### Tech Stack

| Component | Technology |
|---|---|
| Infrastructure as Code | Terraform (~> 6.0 AWS provider) |
| State Backend | S3 (`use_lockfile = true`, no DynamoDB) |
| Kubernetes | EKS 1.33 (managed control plane) |
| CNI | Calico (overlay network, no ENI pod density limits) |
| Node Autoscaler | Karpenter v1.x |
| Pod IAM | EKS Pod Identity (replaces IRSA — no OIDC provider needed) |
| Node Capacity | Spot instances throughout (cost optimisation) |
| Instance Type | t3a (AMD — ~10% cheaper than Intel t3 equivalent) |
| CI/CD | GitHub Actions on `arc-runner-set-app-infra` (ARC on k3d) |

### Key Design Decisions

**Calico instead of VPC CNI**
VPC CNI assigns real VPC IPs to pods, limited by ENI secondary IP capacity per instance
(e.g. 17 pods max on `t3a.medium`). Calico uses an overlay network so pod density is
limited only by CPU/memory — essential for running many small developer workloads.

**EKS Pod Identity instead of IRSA**
IRSA requires creating an IAM OIDC provider and encoding the cluster OIDC URL into every
role's trust policy. Pod Identity uses a simpler trust policy
(`Principal: pods.eks.amazonaws.com`) and an `aws_eks_pod_identity_association` resource
to bind a role to a service account. No OIDC provider needed; roles can be reused across
clusters.

**Dedicated node pools with taints**
Each node pool has a `node-role=<name>:NoSchedule` taint. Workloads must explicitly
tolerate the taint to land on the pool. This guarantees:
- Traefik pods never compete with platform workloads for resources
- Platform workloads (Loki, Grafana, Karpenter controller) are isolated from external traffic

**Karpenter for developer workload scaling**
Managed node groups handle the fixed platform layer (min=1, max=2/3). Karpenter
handles the unpredictable developer workloads, provisioning and deprovisioning nodes
on demand to optimise cost.

**Spot instances throughout**
All managed node groups use SPOT capacity. For the practice cluster, interruption
handling (SQS + EventBridge) is omitted to reduce cost — acceptable because pod
disruption on spot reclaim is tolerable in a non-production environment.

### Node Pools

| Pool | Instance | Spot | Desired | Purpose |
|---|---|---|---|---|
| `traefikexternal` | t3a.small | Yes | 1 | Internet-facing Traefik |
| `traefikinternal` | t3a.small | Yes | 1 | Internal Traefik |
| `clusterworkloads` | t3a.medium | Yes | 2 | Loki, Grafana, Prometheus, Karpenter |

### EKS Addons

| Addon | Purpose |
|---|---|
| `kube-proxy` | Service iptables routing |
| `coredns` | In-cluster DNS |
| `aws-ebs-csi-driver` | EBS PersistentVolumes |
| `eks-pod-identity-agent` | Delivers AWS credentials to pods via Pod Identity |

### CI/CD Workflows

| Workflow | Trigger | What it does |
|---|---|---|
| Terraform Plan | Manual | Init → fmt check → validate → plan, uploads artifact |
| Terraform Apply | Manual | Init → apply |
| Terraform Destroy | Manual (requires typing `destroy` to confirm) | Init → destroy |

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
