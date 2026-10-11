# 03: Onboard your applications to Kubernetes from VMs

| | |
|---|---|
| **Source** | https://www.youtube.com/watch?v=dnO02PKNLzE |
| **Author** | Abhishek Veeramalla |
| **Length / date** | 42:53 · 2025-05-26 |
| **Type** | Design/strategy talk, whiteboard + spreadsheet. **No code, no pipeline, no demo** |
| **Scenario** | A company runs ~100 microservices with Docker Compose on VMs behind a load balancer. You're the DevOps engineer asked to move it all to Kubernetes, from zero |

## What it covers: a six-level migration plan

| Level | Name | Key actions |
|---|---|---|
| **0** | Requirement gathering | Inventory every microservice → map each to an **owning team** → rank by **business criticality** (less / medium / important / critical) → pull **current CPU / memory / disk** from Prometheus/Grafana or CloudWatch → **estimate cost** (current VM bill vs EKS/AKS/GKE, AWS pricing calculator) → put it all in a spreadsheet with an onboarding priority and timeline → **share it with management and the dev teams** |
| **1** | Proof of concept | A small cluster (3 control-plane + 3 worker nodes). Pick **15–20 services across every criticality tier**, mixing stateless and stateful (a DB, an app, a cache, a queue). Write Deployments / StatefulSets / DaemonSets, Services, Ingress (start with the cloud provider's controller, e.g. ALB). Fix CrashLoopBackOffs with **liveness/readiness probes**. QA signs off. About 2–4 weeks |
| **2** | Dev cluster | Size from the spreadsheet with headroom (e.g. 40 CPU / 60 GB needed → 3 × 16 CPU / 24 GB). Minimum 3 control-plane nodes (managed on EKS) and 3+ workers. **Prefer fewer, bigger nodes** (upgrades are node by node). **Namespace per team** → RBAC per namespace, integrated with cloud IAM via OIDC. **ResourceQuota per namespace + LimitRange / requests & limits per pod.** About 30 days |
| **3** | Staging | Option A: share the dev cluster, separated by namespace (`dev-payments`, `stage-payments`), which needs strict RBAC. Option B (**his preference**): a **separate cluster**, sized closer to production. Purpose: a *stable* environment to reproduce production issues, because dev is unstable by design. Regulated companies add QA/UAT/pre-prod |
| **4** | Production | **Multi-AZ** node pools (1a/1b/1c) **plus topology spread constraints**, so pods actually land in every AZ. Autoscaling (Cluster Autoscaler / **Karpenter**). Observability (Prometheus/Grafana). Verified probes and resource settings |
| **5** | Scale production | **Multi-region**: one cluster per region, each with its own ingress, fronted by a global load balancer or **Route 53** geo/latency routing |

Mentioned only as "next levels": Helm, controllers, security, **Kyverno**, GitOps, Istio.

## Coverage against the three-layer guide

| Layer | Coverage |
|---|---|
| **Layer 0: Planning** *(not in the guide yet)* | ✅✅ **This video's whole contribution**: inventory, ownership, criticality, resource baseline, cost, phased rollout, stakeholder sign-off |
| **Layer 1: Baseline** | ✅ Namespace-per-team isolation, RBAC via IAM/OIDC, ResourceQuota + LimitRange (also a DoS / noisy-neighbour control), probes, multi-AZ, separate staging cluster. ❌ No NetworkPolicies, secrets, securityContext, image hardening, private networking |
| **Layer 2: Pipeline** | ❌ None |
| **Layer 3: Enforcement** | ❌ Kyverno and GitOps named as future work only |

## Adopt (best ideas for the final template)

1. **A planning phase before any cluster exists.** Inventory, owners, criticality tiers, measured resource baseline, cost comparison, written plan shared with stakeholders. Most demo projects skip this entirely.
2. **PoC with a deliberately varied sample**: every criticality tier, stateless and stateful.
3. **Migrate in waves by criticality**, least critical first.
4. **Namespace = team = RBAC boundary**, with IAM/OIDC-mapped access (fits EKS Access Entries with namespace-scoped access policies from the guide, §5.2).
5. **ResourceQuota (namespace) + LimitRange / requests & limits (pod)**: two levels of resource guardrails.
6. **Staging as a separate, production-like, stable cluster**, with strict RBAC if it's shared.
7. **Multi-AZ = nodes in every AZ *and* topology spread constraints.** Nodes alone don't give HA.
8. Size from **measured data**, not guesses; fewer, larger nodes to keep upgrades manageable.

## Compromises and gaps (don't copy)

| Issue | Why it matters / what's better |
|---|---|
| Security deferred to "level 6+" | Retrofitting NetworkPolicies, PSA and admission onto 100 running services is far harder than starting with them. Add them at **Level 1 (PoC)** so the PoC proves services run *hardened* (non-root, read-only filesystem), which is exactly where things CrashLoop |
| **No cutover strategy** | How does traffic move from VMs to Kubernetes? Missing: **weighted DNS / ALB weighted target groups** for gradual shifting, a rollback path, running both in parallel |
| **No data-layer plan** | Databases on VMs: migrate to RDS, StatefulSets, or leave them in place? Stateful migration is the hard part, and it's skipped |
| **Multi-region presented as simple** | Routing is the easy part; **data replication and consistency across regions** is the hard part, and it isn't mentioned. Multi-AZ gives HA; multi-region is mainly for **DR and latency** |
| No PodDisruptionBudgets | Needed alongside topology spread, or node drains/upgrades can take down every replica |
| No backup / DR | Velero, etcd/state backups, RPO/RTO targets |
| "110 pods per node" | That's the upstream default. On **EKS with the VPC CNI**, the real limit is **ENIs × IPs per instance type** (often much lower) unless **prefix delegation** is enabled, which directly affects node sizing. Good interview gotcha |
| Self-managed 3+3 PoC cluster | Fine for learning, but if the target is EKS, use EKS for the PoC too, otherwise you're testing a different platform |
| No CI/CD or GitOps | How manifests reach the cluster isn't covered |

## Interview value

The answer to *"How would you migrate an existing app estate to Kubernetes?"*:

> "I'd start before any cluster: inventory the services, map owners, rank by
> criticality, and size from measured CPU/memory with a cost comparison
> signed off by management. Then a PoC on a varied sample, already running
> hardened (non-root, probes, quotas), then dev, a separate production-like
> staging, then multi-AZ production with topology spread and PDBs. I'd migrate
> in waves from least critical, shifting traffic gradually with weighted
> routing so every wave can roll back."

## Open questions to check against later projects

- Does any project show an actual **VM → K8s traffic cutover**?
- Does any project handle the **database / stateful** side properly?
- Does any project do **multi-cluster / multi-region** with GitOps (e.g. ApplicationSets)?
