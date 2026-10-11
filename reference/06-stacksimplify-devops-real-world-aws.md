# 06: Ultimate DevOps Real-World Project Implementation on AWS (StackSimplify)

| | |
|---|---|
| **Source** | https://github.com/stacksimplify/devops-real-world-project-implementation-on-aws (+ CI/CD repo [aws-devops-github-actions-ecr-argocd3](https://github.com/stacksimplify/aws-devops-github-actions-ecr-argocd3)) |
| **Author** | Kalyan Reddy Daida (StackSimplify) |
| **Length** | 21 sections · 235 lectures · ~38 h · 55+ demos · **~17,000 lines of step-by-step READMEs** |
| **Type** | Full build, paid course. Reviewed from **the repos only** (videos not needed; the READMEs are the step docs) |
| **App** | **AWS retail-store sample app**: 5 microservices (UI Java, Catalog Go, Carts Java, Orders Java, Checkout Node) |
| **Stack** | Docker · Compose · Buildx (multi-arch) · **Terraform** (S3 backend, VPC module, EKS, add-ons) · EKS 1.33 · **Pod Identity** · Secrets Store CSI + ASCP · EBS CSI · **RDS MySQL/Postgres, ElastiCache Redis, DynamoDB, SQS** · LB Controller · **ExternalDNS** · ACM · Helm · **Karpenter (on-demand + Spot)** · Metrics Server + **HPA** · **ADOT → X-Ray, CloudWatch Logs, AMP, AMG** · GitHub Actions + **ECR via OIDC** · **Argo CD** |
| **Cost** | Author's estimate: **$30–50 for the whole course** with destroy-after-each-section scripts |
| **Related** | Same app as `~/projects/eks-lifecycle-notes` |

## What it builds (incremental, each section reuses the last)

| § | Topic | Notable details |
|---|---|---|
| 1–5 | Docker, Dockerfile, Compose, **Buildx** | All Dockerfile instructions incl. `HEALTHCHECK`, `USER`; multi-stage; multi-arch (amd64/arm64) |
| 6–7 | Terraform basics → EKS | Remote state S3 + lock; VPC module; `.tfvars`; EKS + private node group |
| 8 | K8s foundation | Pods, Deployments, Services, ConfigMaps, StatefulSets, **probes, requests/limits** |
| 9 | **Secrets** | K8s Secrets → **EKS Pod Identity Agent** → **Secrets Store CSI + AWS provider (ASCP)** → Catalog reads its DB password from Secrets Manager |
| 10 | Storage | EBS CSI (via Pod Identity), StatefulSet with PVC → then **RDS MySQL** instead |
| 11 | Ingress | LB Controller; HTTP → **HTTPS with ACM** |
| 12 | Helm | Basics, values, chart anatomy, **package + publish**, retail store via Helm |
| 13 | **EKS + add-ons in Terraform** | Pod Identity Agent, EBS CSI, LBC (Helm via TF), Secrets Store CSI + ASCP: *everything as code* |
| 14 | **AWS data plane** | RDS MySQL (catalog), RDS Postgres (orders), ElastiCache (checkout), DynamoDB (carts), SQS (orders); **per-service IAM role via Pod Identity** |
| 15–16 | **ExternalDNS** | As an EKS add-on with Pod Identity; Route 53 records created from Ingress |
| 17 | **Karpenter** | NodePools/EC2NodeClass, consolidation, **Spot**, **interruption handling via EventBridge → SQS**, PDBs |
| 18 | **HPA** | Metrics Server; CPU/memory HPA; HPA + Karpenter; **topology spread `ScheduleAnyway` vs `DoNotSchedule`** compared side by side |
| 19 | Helm + data plane | Per-service charts, env values, chart versions v1 → v2 |
| 20 | **Observability (OpenTelemetry)** | ADOT add-on + collector (Pod Identity); **auto-instrumentation** Java/Node; traces → **X-Ray** (incl. **filtering health-check spans, ~85% cost cut**); logs → **CloudWatch**; metrics → **Amazon Managed Prometheus** + **Managed Grafana** |
| 21 | **CI/CD** | GitHub Actions → **ECR via OIDC** (no keys), `sha-<7>` tag → commit to Helm values → **Argo CD** (auto-sync, prune, self-heal) |

## Audit of the actual code

| Area | Finding |
|---|---|
| **EKS access** | ✅ `authentication_mode = "API_AND_CONFIG_MAP"`, Access Entries (incl. Karpenter node role). ⚠️ `bootstrap_cluster_creator_admin_permissions = true` and not `API`-only |
| **API endpoint** | ✅ `env/prod.tfvars`: **private only**. ⚠️ `dev.tfvars` / default `terraform.tfvars`: public `0.0.0.0/0` |
| **Environments** | ✅ `env/dev`, `staging`, `prod` tfvars (different service CIDRs, instance types, endpoint modes). ⚠️ Same `cluster_name` in all three; not separate accounts |
| **Control-plane logs** | ✅ `enabled_cluster_log_types` set |
| **Workload identity** | ✅✅ **Pod Identity everywhere**: LBC, EBS CSI, ExternalDNS, ADOT, Karpenter, and **each app service** (catalog → Secrets Manager, carts → DynamoDB, orders → SQS) |
| **Add-ons** | ✅ All in Terraform: `aws_eks_addon` (pod-identity-agent, ebs-csi, external-dns, metrics-server, cert-manager, adot, kube-state-metrics, node-exporter) + `helm_release` (LBC, Secrets Store CSI, ASCP, Karpenter) |
| **Workloads** | ✅ readiness probes on custom health endpoints, requests + limits, `runAsNonRoot`, `readOnlyRootFilesystem`, drop ALL caps, per-service ServiceAccount, **HPA min 3 / max 12 + PDB `minAvailable: 2`** (avoids the 1-replica PDB trap), **topology spread across hostname and zone**. ⚠️ **No liveness probes** (defensible, but undiscussed). ❌ No NetworkPolicy |
| **Ingress** | ✅ `target-type: ip`, explicit health-check tuning on `/actuator/health/readiness`, HTTPS + ACM, ExternalDNS. ❌ No WAF |
| **Data plane** | ✅ RDS `publicly_accessible = false`, `storage_encrypted = true`, **ingress only from the EKS cluster SG**. ⚠️ `multi_az = false`, `skip_final_snapshot = true`, `deletion_protection = false`, backup retention 1–7 days, ElastiCache `transit_encryption_enabled = false` (cost/demo trade-offs, but not flagged) |
| **Secrets** | ✅ Secrets Manager → **Secrets Store CSI** (mounted). ⚠️ Helm charts still carry `kind: Secret` templates for the in-cluster DB mode |
| **⚠️ State file committed** | `backup-aws-data-plane-state-2025-12-02.json` is in the public repo and contains **2 non-empty DB `password` values and Secrets Manager `secret_string`s**. Probably for torn-down demo DBs, but it's the classic leak: **Terraform state holds secrets in plain text** |
| **CI** | ✅ OIDC to AWS, trust policy **scoped to the repo** (`repo:…:*`), ECR, SHA tag, Argo CD pulls. ❌ **No scanning of any kind** (no Gitleaks, Trivy, SAST, SBOM, signing). ❌ Builds the image **twice** (once per tag) instead of build once + tag twice. ⚠️ Trust condition `:*` allows any branch/PR/environment; pin to `ref:refs/heads/main`. ⚠️ Pushes `latest` too. ⚠️ CI covers the UI service only; one environment, no promotion |
| **Argo CD** | ✅ `automated` + `prune` + `selfHeal`. ⚠️ `project: default`; install via script, not as code; no hardening |
| **Cost** | ✅ Create/destroy scripts per section; **single NAT gateway** module option; Spot via Karpenter; documented cost estimates |

## Coverage against the three-layer guide

| Layer | Coverage |
|---|---|
| **0: Planning** | ⚠️ Architecture diagrams per concept (interactive site); explicit **cost estimates**; no inventory/sizing exercise |
| **1: Baseline** | ✅✅ The best so far: Pod Identity per service, add-ons as code, Access Entries, prod private endpoint, secure pod `securityContext`, Secrets Manager via CSI, managed data stores locked to the cluster SG, HTTPS. ❌ NetworkPolicy, WAF |
| **Reliability** | ✅✅ The best so far: probes, requests/limits, HPA, PDB done correctly, topology spread with the `ScheduleAnyway`/`DoNotSchedule` trade-off, Karpenter Spot + interruption handling |
| **2: Pipeline** | ❌ Essentially none: build + push only |
| **3: Enforcement** | ⚠️ Argo CD GitOps (self-heal). ❌ admission, PSA, signing |
| **Runtime / observability** | ✅✅ Full OTel: traces (X-Ray), logs (CloudWatch), metrics (AMP), dashboards (AMG), auto-instrumentation, **cost-aware trace filtering**. ❌ Falco |

## Adopt (best ideas for the final template)

1. **Everything platform-level in Terraform**: EKS add-ons, controller Helm releases, Pod Identity associations, IAM. Nothing installed by hand.
2. **Pod Identity per service** for real AWS dependencies (DynamoDB, SQS, Secrets Manager). This is the cleanest "each pod gets its own key" demo seen so far.
3. **Managed data plane** (RDS, ElastiCache, DynamoDB, SQS) with SGs that only allow the cluster SG, and credentials from Secrets Manager.
4. **Workload spec as a reference baseline**: readiness probe on a real health endpoint, requests + limits, non-root, read-only root filesystem, drop ALL caps, own ServiceAccount, HPA ≥ 3 + PDB `minAvailable: 2`, topology spread on hostname and zone.
5. **`ScheduleAnyway` vs `DoNotSchedule` side by side**: a great teaching device for spread trade-offs.
6. **Karpenter + Spot + interruption queue** for cost, with PDBs keeping it safe.
7. **Observability with cost awareness**: filter health-check spans before X-Ray (~85% fewer traces).
8. **Per-environment `.tfvars`** with prod stricter than dev (private endpoint, bigger instances).
9. **Incremental structure**: every section builds on the previous one, with create/destroy scripts and **cost estimates**. Exactly the newbie-friendly shape the final template needs.
10. **ALB health-check annotations** pointing at the readiness endpoint.

## Compromises (don't copy)

| Issue | Better |
|---|---|
| **Terraform state committed (with passwords)** | Never commit state; encrypted S3 backend with restricted access; use `manage_master_user_password = true` on RDS so the password never enters state |
| No security pipeline | Add the guide's Layer 2 (Gitleaks, Checkov, Trivy fs/image as gates, SBOM, Cosign) |
| Build twice for two tags | Build once, `docker tag`, push both; or `docker/build-push-action` with multiple tags |
| OIDC trust `repo:…:*` | `repo:org/repo:ref:refs/heads/main` (or an `environment:prod` subject) |
| No NetworkPolicy, no WAF, no PSA/Kyverno, no Falco | See guide Layers 1 and 3 |
| `bootstrap_cluster_creator_admin_permissions = true`, `API_AND_CONFIG_MAP` | `API` only, explicit admin entries |
| RDS single-AZ, no final snapshot, no deletion protection | Fine for cost, but say so; prod tfvars should flip these |
| Argo CD `project: default`, installed by script | Dedicated AppProject; Argo CD itself installed via Terraform/Helm and managed by Argo (app-of-apps) |
| CI only for UI, single environment | Matrix over services; promote the same digest dev → staging → prod |

## Open questions

- Do the videos discuss why there are no liveness probes? (Probably deliberate: avoids restart loops during dependency outages.)
- Section 22 (Gateway API) and 23 (Istio) are "coming soon"; check later for mTLS / L7 policy.
