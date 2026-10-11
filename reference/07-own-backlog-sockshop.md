# 07: Own plan, Sock Shop Platform (Jira backlog + technical spec)

| | |
|---|---|
| **Source** | `~/Downloads/jira-backlog (1).md` (94 tickets, 10 epics) + `~/Downloads/technical-spec.md` (21 sections, ADR-0001…0010) |
| **Author** | The user + Claude, ~2026-07 |
| **Type** | Plan only (nothing built yet) |
| **App** | Weaveworks **Sock Shop**: 8 services (Go, Java, Node), MongoDB ×3, MySQL, RabbitMQ. Upstream Dockerfiles/manifests **deleted** and rebuilt (ADR-0002). Hard mode: TeaStore → TrainTicket |
| **Stack** | Bash → Docker/Compose → GitHub Actions → **KinD (local bridge)** → Terraform/AWS → EKS + Helm → Argo CD + Kyverno + OpenBao + Falco + Argo Rollouts → Prometheus/Loki/Jaeger/SLOs + chaos + Velero + multi-region |

## Verdict

**This plan is already more complete than any of the six reference projects.**
It is the only one with: ADRs for every major decision, a single
**Critical User Journey** used by every verification, acceptance criteria that
include **negative tests and drills** ("a root pod is rejected", "a commit
with a fake key is rejected", "kill the primary and measure"), a **local
Kubernetes bridge** before the cloud, both database paradigms compared, and
an explicit dependency graph between epics.

Its format (epic → ticket → verifiable acceptance criteria → spec link) is
the **format the final template uses**.

## Strengths to keep (unique among all 7)

| Idea | Where |
|---|---|
| ADR per decision, linked from tickets | spec § ADR Index |
| Critical User Journey as the universal "does it work" test | spec § Application Inventory |
| Delete upstream scaffolding and rebuild (the operational layer is the lesson) | ADR-0002 |
| **Local KinD bridge** (3 CP + 3 workers) with failure drills before any cloud spend | E-3, ADR-0003 |
| Java-first manifests to surface the `startupProbe` lesson; `MaxRAMPercentage=75` | SOCKPLAT-8, 20 |
| Image **size budgets** as a blocking CI gate; base images pinned by digest | SOCKPLAT-9 |
| `preStop` + `terminationGracePeriodSeconds` tuned until **zero dropped requests** | SOCKPLAT-28 |
| ECR **immutable** tags + lifecycle; **no `:latest` ever pushed** | SOCKPLAT-15, 38 |
| OIDC trust scoped to repo **and `main`**, plus a **negative test** | SOCKPLAT-36 |
| GitHub Environments with prod reviewer + wait timer | SOCKPLAT-16 |
| **CI never deploys** (ADR-0009); CI pipeline must contain zero `kubectl apply` | SOCKPLAT-63 |
| GitOps self-heal documented as a **security control** + break-glass procedure | SOCKPLAT-64 |
| Kyverno Audit → Enforce with negative tests | SOCKPLAT-66 |
| OpenBao HA + KMS auto-unseal + K8s auth + per-service policy with negative test | SOCKPLAT-67–70 |
| Falco with simulated attacks + per-rule runbook | SOCKPLAT-71 |
| SLOs with **multi-window burn-rate alerts**; every alert links to a runbook | SOCKPLAT-74 |
| Chaos experiments where "silence is a finding" | SOCKPLAT-76 |
| Blameless post-mortem from telemetry only | SOCKPLAT-78 |
| Velero DR drill with measured RTO; CNPG PITR drill | SOCKPLAT-53, 79 |
| Drift detection (scheduled plan + CloudTrail alert) and Terratest | SOCKPLAT-41, 42 |
| Cost automation (dev stop at night, budget alerts) | SOCKPLAT-43, 81 |
| Trace-gated hard mode (no TrainTicket without tracing) | E-8 |

## Problems found

### Bugs / internal contradictions

| # | Issue | Fix |
|---|---|---|
| 1 | **SG spec contradicts `target-type: ip`.** ALB SG egress and node SG ingress use **NodePorts 30000–32767**, but the Ingress uses `target-type: ip`, so the ALB sends traffic **straight to pod IPs on the container port** (8079, 80), not to NodePorts. As written, health checks and traffic would be blocked (or the rules are dead) | Let the LB Controller manage SGs (default: it creates a frontend SG and a shared backend SG and adds node-SG rules for the target ports), or write rules for the **pod ports** from the ALB SG |
| 2 | Spec uses `kubernetes.io/ingress.class: alb` annotation | Deprecated; use `spec.ingressClassName: alb` |
| 3 | Ticket cross-references off by one: spec says graceful shutdown "verified in SOCKPLAT-27" (it's **28**); CNPG failover "measured in SOCKPLAT-53" (it's **54**) | Renumber links |
| 4 | Spec `RDS SG: 5432 from node SG`: fine, but RDS is only consumed by CNPG comparison, not by Sock Shop | Note RDS is for the paradigm comparison only (already implied by ADR-0006) |

### Outdated (written ~3 months ago, or older defaults)

| Item | Now |
|---|---|
| EKS **1.30** | Out of standard support; use the current default (1.33/1.34) and pin `kindest/node` to match |
| **IRSA everywhere** | Prefer **EKS Pod Identity** (simpler, no per-cluster OIDC trust edits, supported by LBC, EBS CSI, ExternalDNS, ADOT, Velero, cert-manager via AWS SDK). Keep IRSA as a lesson, as in reference 06 |
| **No mention of EKS Access Entries / auth mode** | Add: `authentication_mode = "API"`, explicit admin entries, `bootstrap_cluster_creator_admin_permissions = false` |
| MySQL 5.7 (upstream) | EOL; accepted as an upstream constraint (like ADR-0004), but document it |
| Cluster Autoscaler (E-5) and Karpenter (E-7) | Karpenter is the EKS default now; reference 06 shows Spot + interruption queue |

### Gaps (compared with the guide and reference projects)

| Gap | Seen done in | Where it belongs |
|---|---|---|
| **No Phase 0 planning**: no measured sizing, cost model, criticality/owners, threat model | 03, 06 (cost) | New epic before E-0 |
| **No dev environment layer**: no `mise.toml`/`devenv.nix`, `devcontainer.json`, DevPod; no inner loop (Skaffold/Tilt + mirrord) | `gitops-workstation/TOOLS.md` planes 2–3 | New epic replacing most of E-0 |
| **E-0 provisions the host with Bash** (packages, Docker, users on a VM) | — | Conflicts with the workstation rule "nothing project-specific on the host". Keep the Bash skills as an **exercise on a throwaway VM**, not as the workstation setup |
| **Gitleaks only as a pre-commit hook** | 04, guide | Also in CI (hooks are skippable with `--no-verify`) |
| **No IaC scanning in CI** (Checkov / Trivy config) | 01, 04 | E-2 / E-4 |
| **No Dockerfile lint** (hadolint) | 01 | E-1 |
| **No SBOM** | 04 | E-2 |
| **No image signing / provenance**, and Kyverno doesn't verify signatures | guide | E-2 (sign) + E-6 (`verifyImages`) |
| **No DAST** (ZAP baseline/full against staging) | guide | E-6/E-7 |
| **No Pod Security Admission** labels | guide | E-5 |
| **Promotion by digest** across environments not explicit (dev → prod) | 04 (by `:latest`, wrong), guide | E-6 |
| **No WAF / readiness gates / deregistration delay** at the edge | user's Section 11 list, 06 | E-5 |
| **Public-subnet, no-NAT VPC** (ADR-0005) + public+private endpoint | 01, 05, 06 use private nodes | Valid **cost** ADR, but add compensating controls: `public_access_cidrs` = your IP, no SSH, IMDSv2 hop limit 1, SGs tight; and document it as a conscious trade-off in interviews |
| No resume / interview deliverable | 04, 05 | Final epic |

## How it maps to the template

The template (`../template/`) keeps this plan's epics and ticket format, adds
a **Phase 0 (plan)** and a **dev-environment phase**, inserts the missing
DevSecOps controls into the epics where they belong, updates the outdated
items, and fixes the SG bug.
