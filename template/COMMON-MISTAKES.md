# Common mistakes: everything the reference projects got wrong

One list, grouped by phase. Each line says which reference project did it, so
you can see it in context (`../reference/NN-*.md`). **Read the section for a phase before you start that phase.** Rows marked `roadmap` come from the red flags and incident list of the *Cloud & Kubernetes Platform Engineering Roadmap* that this template was later checked against. Rows marked `podinfo run` come from running the template on Podinfo; the RCA is in that project's `platform/docs/rca/`.

## Planning (P0)

| Mistake | Ref | Fix |
|---|---|---|
| Creating a cluster before knowing the workload, owners, size or cost | 01, 02, 04, 05, 06 | P0 inventory, sizing, cost |
| Plan kept private, never agreed with the team | 03 warns | P0-11 sign-off |
| Multi-region treated as a routing problem (data replication ignored) | 03 | P10-11 |
| No traffic cutover or data migration plan when moving from VMs | 03 | Weighted DNS/ALB shifting, a rollback path, a data plan |
| Accepting "it's stateless" / "port 3000, I think" without checking | roadmap | Behavioural questions; values from the code (P0-2b) |
| Service manifest with no owner, so it goes stale and pages the wrong person | roadmap | A named manifest owner (P0-2b) |
| "Tests passed" assumed to mean "deployed" (nobody agreed who promotes or rolls back) | roadmap | The ownership contract (P0-11b) |
| Kubernetes chosen with no stated reason, or "because it's on job ads" passed off as architecture | roadmap | The "why EKS" ADR (P0-13) |
| Extra clusters "for safety" with no named trigger | roadmap | Each cluster needs a reason (P0-6, README "Scale the template") |

## Workstation & dev env (P1)

| Mistake | Ref | Fix |
|---|---|---|
| Tools installed by hand on the host / Windows PATH / an EC2 "workstation" | 04, 05 | `mise.toml` + devcontainer + DevPod |
| Admin IAM user with static access keys for everything | 04, 05 | SSO roles (P5-1) |
| Terraform state committed to Git (contains secrets) | 05, 06 | `.gitignore` on day one; S3 backend (P4-1) |
| Pre-commit hook as the only secret scanner (`--no-verify` skips it) | 07 plan | Gitleaks in CI too (P7-2) |
| `mise install` passes on the host only because it borrows a host tool (`uv`/`pipx` for `pipx:` tools); a clean container fails | podinfo run | Pin the helper (`uv`) in `mise.toml`; the P1-3 clean-container run is the proof, not the host |
| Docker-in-Docker dies in the dev container on an nftables-only host (NixOS): `can't initialize iptables table 'nat'` | podinfo run | `postStartCommand` switches to `iptables-nft` and starts dockerd (P1-3) |
| Secret-leak drill judged by exit code only: hooks never installed (`--hook-types` isn't a flag), or "rejected" because the hook crashed; and AWS's doc key `AKIAIOSFODNN7EXAMPLE` is allowlisted by gitleaks | podinfo run | Random `AKIA` key; the output must show `RuleID: aws-access-token`; `gitleaks-system` + `default_install_hook_types` (P1-4) |
| SIGTERM judged by an idle stop: the app exits on time, but in-flight requests are cut when a shutdown timeout (started before a pre-drain sleep) runs out, and the process still exits 0 | podinfo run | Time the stop with a long request in flight; size the shutdown timeout and grace period from the longest request (P1-6, P6-7) |
| Two agents (or a person and an agent) in one checkout: a branch switch sends the other's commit to the wrong branch, and `git add -A` sweeps in their unfinished files | podinfo run | A `git worktree` per worker; stage files by name |
| `curl https://mise.run \| sh` unpinned: every rebuild installs whatever mise is newest | podinfo run | `MISE_VERSION=v… sh` in `postCreateCommand` (P1-3) |
| kubectl pinned from an example, not from the cluster: 4 minors off kind's default, and that EKS minor already in billed extended support | podinfo run | Choose the Kubernetes minor first; one ADR pins kubectl, the kind node image and EKS together (P1-2) |

## Images (P2)

| Mistake | Ref | Fix |
|---|---|---|
| Single-stage image with build tools in production | 04 | Multi-stage |
| `npm start` as PID 1; SIGTERM ignored | 04 | `node server.js` / tini |
| Unpinned packages and base images | 01 | Pin versions and digests |
| Non-root image binding port 80 (works in Docker, fails in Kubernetes) | — | Listen on 8080 |
| `JAVA_OPTS` set but never passed to the JVM | 07 spec | `JAVA_TOOL_OPTIONS` |
| App code with a hardcoded fallback DB password | 04 | Fail fast if config is missing |
| Hardening test re-passes flags by hand (`--data-path=/data`), so it never sees that the image's own default differs from where the docs mount the volume | podinfo run | Test the image with its default arguments plus only the documented mounts (P2-4) |
| Errors answered with HTTP 200 and the real code only in the JSON body; a check that reads the body (or none) passes while clients, load balancers and metrics see success | podinfo run | Assert the HTTP status (`curl -w %{http_code}`); fix the app if it lies (P2-8) |
| Size gate in whole MB (`size/1024/1024`) rounds down: an 11.5 MiB image passes an 11 MiB budget | podinfo run | Compare bytes; drill just below and just above the budget (P2-5) |
| `time docker stop` < 2 s used as the SIGTERM test: wrongly fails apps that drain on purpose | podinfo run | Check exit 0 (not 137) before the 10 s SIGKILL; write the drain down |
| Debug endpoints switched off on the main port only; a metrics or gRPC listener still serves them (Go: `net/http/pprof` on `DefaultServeMux`) | podinfo run | Probe every listener; give each server its own mux; gate the gRPC twins too (P2-4) |

## Kubernetes basics (P3, P6)

| Mistake | Ref | Fix |
|---|---|---|
| No probes at all | 04, 05 | Readiness + liveness (+ startup for JVMs) |
| Liveness probe checks the database (one blip restarts everything) | user list | Liveness = process only |
| `replicas: 1` everywhere | 04, 05 | HPA min 3 |
| PDB on a 1-replica Deployment (drains and upgrades block) | user list | Replicas first, PDB min 2 |
| Nodes in several AZs but no topology spread | 02 | Zone spread `DoNotSchedule` |
| `replicas:` and an HPA on the same Deployment | — | Drop `replicas:` |
| Node-group min/max with no autoscaler | 04 | Karpenter / Cluster Autoscaler |
| NetworkPolicy never proven enforced | — | Deny test first (P3-7) |
| ClusterRoleBinding described as namespace-wide | 02 | It's cluster-wide; use RoleBinding |
| Keeping the P6-0 hand deploy after Argo CD takes over (two owners for the same objects) | — | `kubectl delete -k` the overlay before the first sync (P8-2) |
| Health check on `/` that returns 200 while the database is dead | roadmap | A readiness path that checks what the pod needs (P3-3) |
| Anything deployed into the `default` namespace | roadmap | A namespace per team/app, before the first deploy |
| Argo CD, Karpenter and monitoring on the same nodes as the apps | roadmap | A tainted platform pool + PriorityClass (P6-5b) |
| Quota sized from today's usage (HPA stalls at the wall) | roadmap | Criticality + headroom ≥ every `maxReplicas` (P6-5) |

## Cloud foundation (P4)

| Mistake | Ref | Fix |
|---|---|---|
| `eksctl create cluster` with CLI flags (not in Git) | 02, 04 | Terraform |
| EKS API endpoint public to `0.0.0.0/0` | 05, 06 dev | Private, or your IP only |
| `aws-auth` ConfigMap; invisible creator admin | 02, 06 | Access Entries, `API` mode |
| Controllers installed by hand / IAM policies patched by hand | 05 | Add-ons + Helm in Terraform/Argo CD |
| Kubernetes version past standard support | 05, 07 | Current default |
| Builds pull straight from npm/Maven/Docker Hub (no artifact proxy; rate limits, no audit, open egress) | all | ECR pull-through + CodeArtifact or Nexus (P4-8) |
| Public worker nodes with SSH open | 02 | Private nodes, SSM, no SSH |
| NodePort SG rules with `target-type: ip` | 07 spec | Controller-managed SGs (P9-3) |
| ElastiCache without transit encryption; RDS with no final snapshot in prod | 06 | Flip in prod tfvars |
| One security group shared by every data store | roadmap | One SG per data store (P4-9) |
| No S3/DynamoDB gateway endpoints, so AWS-to-AWS traffic pays NAT prices | roadmap | Gateway endpoints always; interface endpoints by volume (P4-3) |
| Unpooled connections × `maxReplicas` > the database's `max_connections` | roadmap | Pooling or RDS Proxy (P4-9) |
| One ALB per service | roadmap | One ALB, many hostnames (P9) |
| `.terraform.lock.hcl` git-ignored (each person gets different provider versions) | roadmap | Commit the lock file; never commit state (P4-2) |

## Identity & secrets (P5)

| Mistake | Ref | Fix |
|---|---|---|
| Kubeconfig / long-lived tokens / AWS keys in CI secrets | 04's scenarios 1–3, 05 | GitHub OIDC |
| OIDC trust `repo:org/repo:*` | 06 | Pin branch or environment |
| Long-lived Docker Hub token in CI | 04, 05 | ECR via OIDC |
| Plain Kubernetes Secrets in Git ("base64 is not encryption") | 01 | ESO / CSI / OpenBao |
| One AWS identity for all pods (node role) | 05 | Pod Identity per service |
| "OIDC token is encrypted" | 04 | It's **signed** (anyone can read it, nobody can forge it) |
| Widening a policy to `*` until the AccessDenied goes away | roadmap | Name the layer that denied it, fix only that (P5-8) |
| Testing access with your own admin session instead of the pod's role | roadmap | Test as the real principal (P5-8) |
| A bastion host and SSH keys "for debugging" | roadmap | Session Manager, `kubectl debug` (P5-1) |

## Pipeline (P7)

| Mistake | Ref | Fix |
|---|---|---|
| Every gate disabled (`--exit-code 0`, `|| true`, `continue-on-error`) | 01, 04 | Fail closed |
| Push first, scan later | 01, 05 | Scan the local image |
| Building twice for two tags | 06 | Build once, tag twice |
| Pushing or promoting `:latest` | 04, 06 | Digests only |
| `pull_request` workflow that force-pushes to `main` | 05 | Push only on `main`; no `-f` |
| CI deploys with `kubectl` + cluster credentials | 02, 04 | GitOps |
| Actions pinned to tags or `@master` | 04, most | Pin to SHA |
| CI for only 1 of N services | 05, 06 | Matrix |
| Scanning Terraform that isn't in the repo | 04 | Keep IaC in Git |
| "Tags are immutable" | 02 | Tags are mutable; digests aren't |

## GitOps & enforcement (P8)

| Mistake | Ref | Fix |
|---|---|---|
| Admission control left as "future work" | all | Kyverno + PSA (P8) |
| Argo CD `project: default`, never hardened | 06 | AppProjects, SSO, no public UI |
| Policies straight to Enforce | — | Audit → Enforce |
| A Kyverno mutation not in Git, so the app is OutOfSync forever | roadmap | Put the field in Git, or `ignoreDifferences` on that field (P8-6) |
| An HPA, a KEDA ScaledObject and Argo Rollouts all driving one workload's replicas | roadmap | One owner at a time, written down (P8-8) |

## Edge (P9)

| Mistake | Ref | Fix |
|---|---|---|
| Public domain over HTTP only | 05 | ACM + redirect |
| NodePort on public nodes | 02 | ALB via Gateway API (P9) |
| Readiness gates without readiness probes | user list | Probes first |
| CloudFront in front, ALB still open | user list | Prefix list + secret header |

## Operate (P10)

| Mistake | Ref | Fix |
|---|---|---|
| No observability | 01, 02, 03, 05 | P10-1..3 |
| Alerts on CPU instead of symptoms; alerts without runbooks | user list | SLO burn rate + runbooks |
| Backups never restored | all | DR drill |
| `loki-stack` chart / Promtail (both deprecated) | 08 | `grafana/loki` + Alloy |
| Alert webhook URLs in Helm values in Git | 08 | Secrets Manager → ESO |
| Disabling Argo CD self-heal to make a manual patch stick | 08 | Fix it in Git; break-glass procedure (P8-4) |
| `zap-baseline` described as active attacks | original notes | Baseline is passive |
| An alert that pages someone but links to no runbook | roadmap | `runbook_url` on every paging alert (P10-8c) |
| Security Hub turned on, hundreds of findings, never triaged | roadmap | Triage session; suppressions with review dates (P10-5b) |
| Auto-remediation with a broad role, no dry-run, no loop guard | roadmap | Scoped role, dry-run first, ignore its own events (P10-5c) |
| Cluster never upgraded until the end-of-support email (then extended-support fees) | roadmap | Quarterly upgrades from a runbook (P10-10d) |
| Recovery depends on the thing that's broken (runbooks behind the cluster's SSO, deploys only via Argo CD) | roadmap | Break-glass path; runbooks in Git; one drill with the platform down (P10-9) |

## Present it (P11)

| Mistake | Ref | Fix |
|---|---|---|
| Claims without evidence ("implemented supply-chain security") | — | A screenshot or log of the control firing, in `docs/evidence/` (P11-2) |
| Resume bullets with no number | — | Before/after numbers: image size, deploy time, DORA (P11-3, P10-10b) |
| Hiding the corners you cut | — | A trade-offs table with the production alternative (P11-4) |
| Relative links in published write-ups (they become dead links) | — | Absolute GitHub links (P11-6) |

## Multi-account (P12)

Not the first project's problem. When you get there, see the
[P12 common mistakes](12-multi-account.md#common-mistakes): workloads in the
management account, SCPs tested on prod first, one-sided cross-account trust,
KMS key policies forgotten for shared ECR.
