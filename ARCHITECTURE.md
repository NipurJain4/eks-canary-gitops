# EKS Canary + CI/CD — Architecture

**Project:** recharge-system (QuickCharge) on EKS with Flagger canary, Flux GitOps, and in-cluster Jenkins
**Account:** 821410798987 · **Region:** ap-south-1

---

## 1. High-Level Architecture (End-to-End)

```
                                 ┌───────────────────────────────────────────────┐
   Developer                     │                   GitHub                       │
      │                          │                                                │
      │  git push (app code)     │   ┌────────────────────┐   ┌───────────────┐  │
      └─────────────────────────►│   │ recharge-system    │   │ eks-canary-   │  │
                                 │   │ (app + Dockerfile  │   │ gitops        │  │
                                 │   │  + Jenkinsfile)    │   │ (K8s manifests│  │
                                 │   └─────────┬──────────┘   │  + image tag) │  │
                                 │             │              └───────▲───────┘  │
                                 └─────────────┼──────────────────────┼──────────┘
                                               │ (Jenkins builds)     │ (Jenkins commits tag)
                                               │                      │
   ┌───────────────────────────────────────────────────────────────────────────────────┐
   │  AWS  (account 821410798987, ap-south-1)                                            │
   │                                                                                     │
   │   ┌─────────────────────────── EKS Cluster (1.35) ──────────────────────────────┐  │
   │   │                                                                              │  │
   │   │   ┌──────────┐   build(Kaniko)   ┌───────────┐   push (node role)           │  │
   │   │   │ Jenkins  │──────────────────►│  agent    │────────────────────►┌──────┐ │  │
   │   │   │  (pod)   │  commit tag ──────►(to GitHub) │                     │ ECR  │ │  │
   │   │   └──────────┘                    └───────────┘                     └──┬───┘ │  │
   │   │                                                                        │ pull│  │
   │   │   ┌──────────┐   watches gitops repo                                   ▼     │  │
   │   │   │ Flux CD  │──────────────► applies manifests ──► Deployments/Canary       │  │
   │   │   └──────────┘                                                               │  │
   │   │                                                                              │  │
   │   │   ┌──────────┐  creates primary+canary, shifts traffic, checks metrics       │  │
   │   │   │ Flagger  │◄──── queries ──── Prometheus ◄── scrapes ── NGINX metrics      │  │
   │   │   └────┬─────┘                                                                │  │
   │   │        │ sets canary weight (10%→50%)                                         │  │
   │   │        ▼                                                                      │  │
   │   │   ┌─────────────┐      ┌──────────────────┐     ┌──────────────────┐          │  │
   │   │   │ NGINX       │─────►│ primary (stable) │     │ canary (new ver) │          │  │
   │   │   │ Ingress     │      │ recharge-system  │     │ recharge-system  │          │  │
   │   │   └──────┬──────┘      └──────────────────┘     └──────────────────┘          │  │
   │   └──────────┼───────────────────────────────────────────────────────────────────┘  │
   │              │ Service type=LoadBalancer                                             │
   │              ▼                                                                        │
   │        ┌───────────┐                                                                  │
   │        │   NLB     │  (internet-facing)                                               │
   │        └─────┬─────┘                                                                  │
   └──────────────┼──────────────────────────────────────────────────────────────────────┘
                  │
                  ▼
               Users (browser)
```

---

## 2. AWS Infrastructure (Terraform-provisioned)

```
AWS Account 821410798987 — ap-south-1
│
├── VPC (10.0.0.0/16)
│   ├── Public Subnet AZ-a   ──┐
│   ├── Public Subnet AZ-b   ──┤── Internet Gateway ──► Internet
│   │        └── NAT Gateway + EIP
│   ├── Private Subnet AZ-a  ──┐
│   └── Private Subnet AZ-b  ──┴── route 0.0.0.0/0 ──► NAT Gateway
│
├── EKS Cluster "eks-canary-practice" (v1.35)
│   ├── Control plane (AWS-managed, HA)
│   ├── OIDC provider (for IRSA)
│   ├── Managed Node Group: 2–4× t3.medium Spot (in private subnets)
│   └── Add-ons: coredns, kube-proxy, vpc-cni, pod-identity-agent, aws-ebs-csi-driver
│
├── ECR repository "podinfo"  (app images)
│
├── IAM Roles (IRSA / instance)
│   ├── Cluster role
│   ├── Node role (default-eks-node-group-*)  + ECR-push policy  [SCP-excluded]
│   ├── aws-lbc (AWS Load Balancer Controller)
│   ├── ebs-csi  (EBS CSI driver)
│   └── jenkins  (IRSA, reserved)
│
├── S3 bucket (Terraform remote state, native locking)
│
└── NLB (created at runtime by NGINX Service type=LoadBalancer, internet-facing)
```

---

## 3. GitOps / CI-CD Flow (Two-Repo Model)

```
┌──────────────────────┐         ┌──────────────────────────┐
│  recharge-system repo│         │  eks-canary-gitops repo   │
│  (APPLICATION)       │         │  (DESIRED CLUSTER STATE)  │
│  - index.html        │         │  - clusters/practice/     │
│  - Dockerfile        │         │  - infrastructure/        │
│  - Jenkinsfile       │         │  - apps/recharge-system/  │
└──────────┬───────────┘         └─────────────▲────────────┘
           │ 1. dev push                        │ 3. Jenkins commits new image tag
           ▼                                     │
   ┌───────────────┐  2. build+push    ┌─────────┴────────┐
   │   Jenkins     │──────────────────►│      ECR         │
   │ (Kaniko,      │                   └──────────────────┘
   │  node role)   │
   └───────────────┘
                                        4. Flux detects commit
                                                 │
                                                 ▼
                                        ┌──────────────────┐
                                        │    Flux CD       │ applies to cluster
                                        └────────┬─────────┘
                                                 ▼
                                        ┌──────────────────┐
                                        │    Flagger       │ runs canary analysis
                                        └──────────────────┘

KEY RULE: Jenkins only commits to Git. Flux is the only thing that changes the cluster.
```

---

## 4. Canary Deployment Flow (Flagger + NGINX)

```
New image tag committed → Flux applies → Flagger detects deployment change
         │
         ▼
   ┌─────────────────────────────────────────────────────────────┐
   │  Flagger Analysis Loop (every 30s)                           │
   │                                                              │
   │   1. Scale up canary pods (new version)                      │
   │   2. loadtester + traffic-generator send steady traffic      │
   │      ─────────► NGINX ─────► records request metrics         │
   │   3. Prometheus scrapes NGINX metrics (every 15s)            │
   │   4. Flagger queries success-rate (custom MetricTemplate)    │
   │                                                              │
   │   IF error-rate < 5%  → NGINX canary-weight += 10%           │
   │        10% → 20% → 30% → 40% → 50% (maxWeight)               │
   │        → PROMOTE: copy canary→primary, route 100% to primary │
   │                                                              │
   │   IF error-rate ≥ 5% (x5 checks) → ROLLBACK to primary       │
   └─────────────────────────────────────────────────────────────┘

Traffic split (example at 30% weight):
   Users ──► NGINX ──┬── 70% ──► primary (stable)
                     └── 30% ──► canary  (new version)
```

**Why continuous traffic matters:** Prometheus `rate()[1m]` needs multiple samples; without steady traffic the metric returns no data → false rollback. Fixed with 15s scrape + always-on traffic-generator.

---

## 5. EKS Version Upgrade Flow (1.34 → 1.35)

```
terraform apply (cluster_version 1.34 → 1.35)
         │
         ▼
STEP 1: Control plane upgrade (in-place, ~10 min, zero downtime, HA-managed)
         │  API server now 1.35; nodes still 1.34 (allowed: CP max 1 minor ahead)
         ▼
STEP 2: Add-ons updated to 1.35-compatible versions (coredns, kube-proxy, vpc-cni, ...)
         │
         ▼
STEP 3: Node group rolling replacement (max_unavailable 33%, ~15 min)
         ┌──────────────────────────────────────────────┐
         │ for each old 1.34 node:                       │
         │   1. launch new 1.35 node (surge)             │
         │   2. cordon old node (no new pods)            │
         │   3. drain: evict pods → reschedule on 1.35   │
         │   4. terminate old node                       │
         └──────────────────────────────────────────────┘
         │  App stays up (2 replicas + gradual drain)
         ▼
RESULT: control plane 1.35, all nodes v1.35.8, zero downtime ✅

Upgrade order rule: Control plane → Add-ons → Node groups (never nodes ahead of CP)
```

---

## 6. Network Path (Request Flow)

```
Browser
   │ HTTP (Host: recharge.example.com)
   ▼
NLB (internet-facing, public subnets)
   │
   ▼
NGINX Ingress Controller (pods on nodes)
   │ host/path match → weighted split (Flagger-managed)
   ├──────────► recharge-system-primary Service → primary pods
   └──────────► recharge-system-canary  Service → canary pods (during rollout)
```

Note: Flagger requires host-based routing. Without real DNS, access via `/etc/hosts`
(`<NLB_IP> recharge.example.com`) or `kubectl port-forward`.

---

## 7. Security / Credentials Model

```
┌────────────────────────────────────────────────────────────────┐
│  Who authenticates to AWS, and how (no static keys anywhere):    │
│                                                                  │
│  • You (Terraform/kubectl)   → SSO role (temp, SCP-excluded)     │
│  • AWS LB Controller (pod)   → IRSA role (OIDC)                  │
│  • EBS CSI driver (pod)      → IRSA role (OIDC)                  │
│  • Jenkins build (pod)       → NODE role (SCP-excluded, auto-    │
│                                 refreshing) → ECR push           │
│  • Jenkins → GitHub          → GitHub PAT (stored as Jenkins     │
│                                 credential 'github-token')       │
│                                                                  │
│  SCP guardrail (org mgmt acct) denies eks:*/ecr:* except for     │
│  excluded principals: user, default-eks-node-group-*             │
└────────────────────────────────────────────────────────────────┘
```

---

## 8. Component Summary

| Component | Namespace | Purpose |
|---|---|---|
| Flux controllers | flux-system | GitOps reconciliation |
| NGINX Ingress | ingress-nginx | Traffic router (NLB front) |
| Flagger | flagger-system | Canary controller |
| Prometheus | flagger-system | Metrics for canary |
| loadtester | flagger-system | Canary analysis traffic |
| Jenkins | jenkins | CI (build/push/commit) |
| recharge-system (primary/canary) | recharge-system | The app |
| traffic-generator | recharge-system | Steady traffic for stable metrics |
| EBS CSI, LB controller, coredns, kube-proxy, vpc-cni | kube-system | Platform |
```
