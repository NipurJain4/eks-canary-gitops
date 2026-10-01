# EKS Canary Deployment & Version Upgrade — Project Documentation

**Author:** Nipur Jain (nipur@cloudthat.com)
**Account:** 821410798987 · **Region:** ap-south-1 (Mumbai)
**Period:** Sept 2026 – Oct 2026
**Status:** ✅ Complete (teardown pending)

---

## 1. Objective / Target

Build a hands-on lab to **learn and demonstrate**:

1. **Progressive canary deployments** — gradually shift traffic to a new app version, automatically promote if healthy or roll back if not.
2. **A zero-downtime EKS Kubernetes version upgrade** (1.34 → 1.35).
3. A complete **GitOps CI/CD pipeline** around it.

The application deployed is **`recharge-system`** — a static "QuickCharge" web page (HTML + Tailwind), served via nginx.

---

## 2. Technology Stack

| Layer | Tool | Role |
|---|---|---|
| Infrastructure | **Terraform** | Provision VPC, EKS, node group, ECR, IAM/IRSA, EBS CSI |
| Container Registry | **Amazon ECR** | Store the app image |
| CD (GitOps) | **Flux CD** | Reconcile manifests from GitHub into the cluster |
| Progressive delivery | **Flagger + NGINX Ingress** | Canary traffic shifting + metric analysis |
| Metrics | **Prometheus** | Success-rate/latency for canary gating |
| CI | **Jenkins (in-cluster)** | Build image (Kaniko) → push ECR → commit tag |
| Source | **GitHub** | App repo + GitOps repo |
| Region/Nodes | ap-south-1 · 2× t3.medium Spot | — |

**Repos:**
- App: `github.com/NipurJain4/recharge-system`
- GitOps: `github.com/NipurJain4/eks-canary-gitops`

---

## 3. Architecture

```
Developer
   │ git push (app code change)
   ▼
GitHub (recharge-system)
   │ (Jenkins polls / manual build)
   ▼
Jenkins (in EKS)  ──Kaniko build──► image ──push (node role)──► Amazon ECR
   │ commits new image tag
   ▼
GitHub (eks-canary-gitops)  ◄── Flux watches this repo
   │
   ▼
Flux CD (in EKS) ── applies manifests ──► Kubernetes
   │
   ▼
Flagger ── creates primary + canary, shifts traffic via NGINX ──► progressive rollout
   │ queries Prometheus (success-rate)
   ├─ healthy → promote 10%→50%→100%
   └─ unhealthy → auto-rollback
   ▼
NGINX Ingress ──► NLB ──► users
```

**Key GitOps principle:** Jenkins never runs `kubectl apply`. It only commits to Git; Flux owns the cluster state.

---

## 4. Infrastructure (Terraform)

Resources provisioned (~65):
- **VPC** — 2 AZs, public + private subnets, single NAT gateway, Internet Gateway
- **EKS cluster** — started at 1.34, public endpoint, OIDC/IRSA enabled
- **Managed node group** — 2× t3.medium Spot (min 2 / max 4), 33% surge for upgrades
- **Add-ons** — coredns, kube-proxy, vpc-cni, eks-pod-identity-agent, **aws-ebs-csi-driver**
- **ECR repo** — `podinfo` (reused name), scan-on-push, keep-last-10 lifecycle
- **IRSA roles** — AWS Load Balancer Controller, Jenkins, EBS CSI driver
- **Node role ECR-push policy** — lets in-cluster Jenkins push via node role
- **Remote state** — S3 backend with native locking (no DynamoDB), KMS disabled

Terraform file layout: `versions.tf, providers.tf, variables.tf, data.tf, vpc.tf, eks.tf, irsa.tf, ecr.tf, node-ecr-push.tf, ebs-csi-irsa.tf, jenkins-irsa.tf, outputs.tf, backend.tf`

---

## 5. How It Was Achieved (Step by Step)

1. **Scaffolded Terraform** (VPC → EKS → nodes → ECR → IRSA), validated offline.
2. **Scaffolded GitOps repo** (Flux Kustomizations, NGINX, Flagger, Prometheus, loadtester, app + Canary CR).
3. **Set up S3 remote state** (native locking, no DynamoDB).
4. **Provisioned infra** via `terraform apply`.
5. **Installed AWS Load Balancer Controller** (Helm + IRSA) → provisions the NLB.
6. **Bootstrapped Flux CD** against the GitHub repo → auto-deploys everything.
7. **Containerized recharge-system** (nginx + /healthz), pushed to ECR.
8. **Deployed the app + Flagger Canary** via Flux → canary initialized.
9. **Made it internet-reachable** (internet-facing NLB).
10. **Set up in-cluster Jenkins** (Flux HelmRelease) with Kaniko builds + node-role ECR auth.
11. **Ran the full CI/CD + canary flow** — code change → Jenkins → ECR → Git → Flux → Flagger → live.
12. **Performed EKS 1.34 → 1.35 upgrade** (control plane → add-ons → rolling node replacement, zero downtime).

---

## 6. Problems Faced & How They Were Solved

### 6.1 Service Control Policy (SCP) blocking EKS/ECR (recurring, biggest challenge)
- **Problem:** An org-level SCP (`p-jqf3wzw9` in management account `775267928995`) **explicitly denied** `eks:*` and `ecr:*` — overriding the account IAM policy. An SCP deny cannot be overridden by IAM.
- **How identified:** Error messages said *"explicit deny in a service control policy"* (vs. "no identity-based policy allows" for plain IAM gaps).
- **Solution:** Admin added **exclusions** to the SCP for specific principals (via `aws:userid` and `aws:PrincipalArn` "NotLike" conditions).
- **Recurred for multiple principals:** the user role, then the EKS **node role** (nodes couldn't pull images), then (would have) the Jenkins role.
- **Lesson:** Every *new* IAM principal needing EKS/ECR must be added to the SCP exclusion. Wildcards (`default-eks-node-group-*`) survive rebuilds.

### 6.2 Missing companion IAM permissions (one-at-a-time)
- **Problem:** `terraform apply` failed repeatedly on missing tag/describe actions (`iam:TagPolicy`, `logs:PutRetentionPolicy`, `ec2:DescribeVpcAttribute`, `ecr:SetRepositoryPolicy`, etc.) — the role had "create" but not companion "describe/tag" permissions.
- **Solution:** Admin added these in batches; documented them in `missing-permissions-policy.json`.

### 6.3 `iam:PassRole` denied for node group
- **Problem:** Creating the node group needs `iam:PassRole` to hand the node role to EKS; the original policy's PassRole had an unresolved `<ACCOUNT_ID>` placeholder.
- **Solution:** Admin granted scoped `iam:PassRole` for the node role.

### 6.4 Node group CREATE_FAILED — nodes couldn't pull images
- **Problem:** Node group failed with `NodeCreationFailure: Unhealthy nodes`. Root cause: `aws-node` (CNI) + `kube-proxy` pods stuck in `ImagePullBackOff` with *"authorization failed: no basic auth credentials"*.
- **Diagnosis:** The **node role** (separate principal) was still SCP-blocked from `ecr:GetAuthorizationToken`, so nodes couldn't authenticate to pull the EKS system images.
- **Solution:** Admin extended the SCP exclusion to `role/default-eks-node-group-*`. Deleted the failed node groups, cleaned state, re-applied → nodes joined successfully.

### 6.5 Canary kept auto-rolling-back (even though app was healthy)
- **Problem:** Flagger always rolled back with *"no values found for nginx metric request-success-rate"*. The app was 100% healthy (all HTTP 200).
- **Root cause (two parts):**
  1. **Prometheus scrape interval was 1 minute**, but Flagger's query used `rate(...[1m])` — a 1-min window with 1-min scraping yields only **1 sample**, and `rate()` needs ≥2 → returned empty.
  2. **Label mismatch** — Flagger's built-in nginx query uses `exported_namespace`; our metrics used `namespace`.
  3. **Bursty traffic** — the loadtester sent traffic in bursts with gaps; during gaps the error rate spiked on stray 404/504 noise.
- **Solution:**
  1. Set **Prometheus scrape interval to 15s** (so a 1-min window has ~4 samples).
  2. Wrote a **custom MetricTemplate** using the correct `namespace` label + `or on() vector(0)` fallback (no NaN on zero traffic).
  3. Added an **always-on traffic-generator** Deployment (continuous curl loop) + a longer, higher-concurrency loadtester, plus relaxed the error threshold to 5% for low-traffic noise.
- **Result:** Canary promotes cleanly; rollback still fires correctly on genuinely bad versions.

### 6.6 SSO credential expiry (1-hour) blocking automation
- **Problem:** SSO temporary credentials expire hourly; storing them in Jenkins would break every hour.
- **Solution:** Run Jenkins **in-cluster** using the **node role** (auto-refreshing, SCP-excluded) for ECR — no stored keys, nothing expires. (An EC2-with-instance-role option was also explored but its `eks-jenkins` role wasn't SCP-excluded.)

### 6.7 Jenkins pod stuck Pending (EKS 1.34 PVC provisioning)
- **Problem:** Jenkins `jenkins-0` stuck `Pending` — *"unbound immediate PersistentVolumeClaims"*. The `gp2` StorageClass used the **in-tree `kubernetes.io/aws-ebs` provisioner, which was removed in EKS 1.34+**, so nothing provisioned the volume.
- **Solution:** Added the **aws-ebs-csi-driver** addon (with IRSA role) + created a **gp3 CSI-based default StorageClass**. PVC bound, Jenkins scheduled.

### 6.8 NLB internal / app not reachable in browser
- **Problem:** The NGINX NLB defaulted to **internal** scheme → unreachable from the internet. Also the ingress host `recharge.example.com` has no real DNS.
- **Solution:** Added `aws-load-balancer-scheme: internet-facing` annotation (recreated the NLB); used `/etc/hosts` or port-forward for browser access (host-based routing is required by Flagger's nginx canary).

### 6.9 Terraform state drift after failed/partial applies
- **Problem:** Partial applies left resources that already existed (ECR repo, CloudWatch log group) → "already exists" errors on re-apply.
- **Solution:** `terraform import` to adopt existing resources into state; `terraform state rm` where needed.

### 6.10 SCP exclusion expired between sessions
- **Problem:** The exclusion was granted only until the 18th; after pausing the POC, it was removed → resume failed with SCP denies again.
- **Solution:** Raised a ticket (Rutvik approval) to re-grant the exclusion for 2 more days; included node + `eks-*` role patterns to cover rebuilds.

---

## 7. Key Concepts Learned

- **GitOps:** The Git repo is the single source of truth; Flux continuously reconciles the cluster to match it. Flux only watches the *GitOps* repo, not the app repo — CI bridges the two by committing image tags.
- **Canary / Flagger:** Flagger creates a `-primary` deployment, shifts traffic in weighted steps via NGINX, gates on Prometheus metrics, and promotes or rolls back automatically. It needs a traffic router (NGINX), metrics (Prometheus), and traffic (loadtester) to function.
- **EKS upgrade order:** control plane → add-ons → node groups. Control plane can be one minor ahead of nodes, never behind. Nodes are **replaced** (not upgraded in-place) — drained gradually for zero downtime.
- **SCP vs IAM:** An SCP is an org-level guardrail that can only *deny* (or allow-list exclusions); an explicit SCP deny overrides any IAM allow. Fixing requires the management-account admin.
- **IRSA & node-role credentials:** Pods get AWS creds either via IRSA (per-service-account role) or the node's instance role. Using the node role (already SCP-excluded) avoided extra approvals and the SSO-expiry problem.

---

## 8. Final State (Verified)

- **Control plane:** Kubernetes **1.35**
- **Nodes:** 2× v1.35.8, Ready
- **Workloads:** recharge-system (primary ×2), traffic-generator, Jenkins, Flagger, Prometheus, NGINX — all Running
- **Canary:** `Succeeded` (promotion verified)
- **Upgrade:** zero-downtime (app served throughout the ~25-min node roll)
- **CI/CD:** Jenkins → ECR → GitOps → Flux → Flagger — verified end-to-end with live visible changes

---

## 9. Cost

- Running rate ≈ **$0.22/hour** (EKS control plane $0.10 + 2× t3.medium Spot ~$0.056 + NAT ~$0.045 + NLB ~$0.016).
- POC total kept **well under ~$15** by tearing down between sessions.
- **Cost control:** `terraform destroy` (via `teardown.sh`) removes all billable resources; S3 state bucket (~cents/mo) retained for rebuilds.

---

## 10. Operational Scripts

- **`resume.sh`** — full rebuild: terraform apply → kubectl → build/push image → LB controller → Flux bootstrap → Flux deploys everything.
- **`teardown.sh`** — safe destroy: delete NLB (NGINX svc) → uninstall LB controller → `terraform destroy`.
- **`setup-tf-backend.sh`** — create S3 state bucket with versioning + native locking.
- **`missing-permissions-policy.json`** — the IAM/SCP permissions needed (for admin requests).

---

## 11. Repository Layout

```
canary_p/
├── terraform/              # Infrastructure as code (VPC, EKS, ECR, IRSA, EBS CSI)
├── gitops/                 # Flux-reconciled manifests
│   ├── clusters/practice/  # Flux bootstrap + Kustomizations
│   ├── infrastructure/     # nginx, flagger, prometheus, loadtester, jenkins
│   └── apps/recharge-system/  # app Deployment, HPA, Canary, traffic-generator
├── recharge-system/        # App source + Dockerfile + Jenkinsfile
├── resume.sh / teardown.sh / setup-tf-backend.sh
├── missing-permissions-policy.json
└── PROJECT-DOCUMENTATION.md (this file)
```

---

## 12. Lessons & Takeaways

1. **SCPs are the silent blocker** — when IAM looks correct but calls still fail, check for an SCP explicit deny. Each new principal (user, node, CI) may need its own exclusion.
2. **Canary metrics are timing-sensitive** — the Prometheus scrape interval must be well under the metric query window, and the canary needs continuous traffic to measure.
3. **EKS 1.34+ removed in-tree storage** — the EBS CSI driver addon is mandatory for PersistentVolumes.
4. **In-cluster CI + node role** elegantly solves SSO credential expiry without stored keys.
5. **GitOps separation** — app repo vs gitops repo; CI bridges them by committing image tags.
6. **Terraform import/state rm** are essential for recovering from partial applies.
7. **Zero-downtime upgrades** work via gradual node replacement + multiple replicas + proper surge settings.
