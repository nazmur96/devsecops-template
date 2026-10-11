# Common mistakes: everything the reference projects got wrong

One list, grouped by phase. Each line says which reference project did it, so
you can see it in context (`../reference/NN-*.md`). Read it before each phase.

## Planning (P0)

| Mistake | Ref | Fix |
|---|---|---|
| Creating a cluster before knowing the workload, owners, size or cost | 01, 02, 04, 05, 06 | P0 inventory, sizing, cost |
| Plan kept private, never agreed with the team | 03 warns | P0-11 sign-off |
| Multi-region treated as a routing problem (data replication ignored) | 03 | P10-11 |
| No traffic cutover or data migration plan when moving from VMs | 03 | Weighted DNS/ALB shifting, a rollback path, a data plan |

## Workstation & dev env (P1)

| Mistake | Ref | Fix |
|---|---|---|
| Tools installed by hand on the host / Windows PATH / an EC2 "workstation" | 04, 05 | `mise.toml` + devcontainer + DevPod |
| Admin IAM user with static access keys for everything | 04, 05 | SSO roles (P5-1) |
| Terraform state committed to Git (contains secrets) | 05, 06 | `.gitignore` on day one; S3 backend (P4-1) |
| Pre-commit hook as the only secret scanner (`--no-verify` skips it) | 07 plan | Gitleaks in CI too (P7-2) |

## Images (P2)

| Mistake | Ref | Fix |
|---|---|---|
| Single-stage image with build tools in production | 04 | Multi-stage |
| `npm start` as PID 1; SIGTERM ignored | 04 | `node server.js` / tini |
| Unpinned packages and base images | 01 | Pin versions and digests |
| Non-root image binding port 80 (works in Docker, fails in Kubernetes) | — | Listen on 8080 |
| `JAVA_OPTS` set but never passed to the JVM | 07 spec | `JAVA_TOOL_OPTIONS` |
| App code with a hardcoded fallback DB password | 04 | Fail fast if config is missing |

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

## Identity & secrets (P5)

| Mistake | Ref | Fix |
|---|---|---|
| Kubeconfig / long-lived tokens / AWS keys in CI secrets | 04's scenarios 1–3, 05 | GitHub OIDC |
| OIDC trust `repo:org/repo:*` | 06 | Pin branch or environment |
| Long-lived Docker Hub token in CI | 04, 05 | ECR via OIDC |
| Plain Kubernetes Secrets in Git ("base64 is not encryption") | 01 | ESO / CSI / OpenBao |
| One AWS identity for all pods (node role) | 05 | Pod Identity per service |
| "OIDC token is encrypted" | 04 | It's **signed** (anyone can read it, nobody can forge it) |

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
