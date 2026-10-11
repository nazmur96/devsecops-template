# Reference projects: compare and contrast

**Goal:** review 5–7 demo projects, record what each does well and where it
cuts corners, then merge the best of all of them with
[`../DEVSECOPS-GUIDE.md`](../DEVSECOPS-GUIDE.md) into **one final project
template that a newcomer can follow end to end without needing other
resources**.

**Method:** one file per project, same structure each time: what it builds →
coverage per layer → **Adopt** → **Compromises** → open questions. This index
keeps the running matrix.

## Projects

| # | Project | Author | Type | File |
|---|---|---|---|---|
| 01 | Day-7 End-to-End DevSecOps (GitHub Actions + Argo CD + EKS) | Abhishek Veeramalla | Build (YouTube) | [01](01-abhishek-day7-e2e-devsecops.md) |
| 02 | DevSecOps Mega Project (Jenkins + SonarQube + Trivy + ECR + EKS) | Cloud With VarJosh | Build (YouTube) | [02](02-varjosh-jenkins-eks-megaproject.md) |
| 03 | Onboard applications to Kubernetes from VMs | Abhishek Veeramalla | Design talk (YouTube) | [03](03-abhishek-vm-to-k8s-onboarding.md) |
| 04 | Production-Grade End-to-End DevSecOps Project (42 parts) | DevOpsShack (Aditya Jaiswal) | Build (Bilibili mirror of Udemy) + [repo](https://github.com/jaiswaladi246/3-Tier-User-Platform-Project) | [04](04-devopsshack-production-devsecops.md) |
| 05 | Ultimate DevOps Project + Resume Prep (OTel Astronomy Shop) | Abhishek Veeramalla | Build (Udemy) | [05](05-abhishek-udemy-otel-ultimate-devops.md) |
| 06 | Ultimate DevOps Real-World Project on AWS (retail-store app) | StackSimplify (Kalyan Reddy) | Build (paid course), reviewed from repos | [06](06-stacksimplify-devops-real-world-aws.md) |
| 07 | **Own plan**: Sock Shop Platform (94-ticket Jira backlog + technical spec) | user + Claude | Plan (not built) | [07](07-own-backlog-sockshop.md) |
| 08 | DevSecOps Platform Template (self-hosted observability example) | abhishekpanda0620 | Build (GitHub) | [08](08-abhishekpanda-devsecops-platform-template.md) |
| 09 | "Explain your end-to-end CI/CD pipeline" (interview-answer video) | YouTube (transcript pasted) | Talk | [09](09-cicd-interview-answer-video.md) |

**Result:** the merged template is in [`../template/`](../template/README.md).
Reference 07's format (epic → ticket → acceptance criteria with negative tests
and drills) is the template's format.

## Running matrix

✅ done well · ⚠️ done with a compromise · ❌ missing · — not in scope ·
04 column combines its **repo** and all 42 video parts; 08 and 09 are compared in their own files


| Area | 01 | 02 | 03 | 04 | 05 | 06 | Best so far |
|---|---|---|---|---|---|---|---|
| **L0: Planning** (inventory, owners, criticality, sizing, cost) | ⚠️ README | ⚠️ diagram | ✅✅ | ⚠️ architecture + app walkthrough | ⚠️ | ⚠️ diagrams + **cost estimates** | **03** (+ 06 cost) |
| **L0: Environments** (dev / staging / prod) | ❌ | ❌ | ✅ separate staging cluster | ⚠️ QA + prod **namespaces, one cluster** | ❌ | ⚠️ dev/staging/prod tfvars | **03** concept, **06** mechanics |
| **L0: Branching / promotion** | ❌ | ❌ | — | ✅ feature → qa → main, retag for prod (⚠️ promotes `:latest`) | ❌ | ❌ | **04** (fixed: promote digest) |
| **L1: Terraform state** | ⚠️ | ❌ eksctl | — | ❌ no IaC (eksctl + CLI) | ✅ S3 + DynamoDB | ✅ ⚠️ backup **committed with passwords** | **05** |
| **L1: VPC design** | ✅ | ⚠️ | — | ⚠️ eksctl defaults | ✅✅ 3 AZ, NAT/AZ | ✅ (+ single-NAT cost option) | **05** |
| **L1: Private networking / API** | ✅ | ⚠️ public nodes, SSH | — | ⚠️ eksctl defaults (public API) | ⚠️ **public API** | ✅ prod private API | **01 / 06** |
| **L1: Image hardening** | ✅ | ⚠️ | — | ⚠️ non-root, single-stage | ⚠️ mixed | ✅ multi-stage, multi-arch, USER, HEALTHCHECK | **06** |
| **L1: NetworkPolicies** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | **01** |
| **L1: Secrets** | ⚠️ K8s Secret | ⚠️ Jenkins creds | ❌ | ✅ Secrets Manager + ESO (video) | ❌ | ✅ Secrets Manager + CSI via Pod Identity | **04 / 06** |
| **L1: Cluster access** | — | ⚠️ aws-auth | ✅ ns-per-team RBAC | ✅ separate OIDC roles per env, `auth can-i` | ⚠️ creator admin | ✅ Access Entries | **06** + **04** per-env roles |
| **L1: Workload identity** | ❌ | ❌ | — | ⚠️ IRSA via eksctl (LBC, ESO, EBS CSI) | ⚠️ IRSA (LBC only) | ✅✅ Pod Identity everywhere | **06** |
| **L1: Add-ons as code** | — | ❌ | — | ❌ CLI + Helm by hand | ❌ eksctl + helm by hand | ✅✅ `aws_eks_addon` + `helm_release` | **06** |
| **L1: Ingress / edge** | ❌ | ❌ NodePort | ⚠️ | ✅ ALB + HTTPS + redirect | ⚠️ HTTP only | ✅ HTTPS + ExternalDNS + tuned health checks | **06** |
| **L1: Requests / limits / quotas** | ❌ | ❌ | ✅ quota + LimitRange | ❌ | ⚠️ memory limits | ✅ | **03 + 06** |
| **L1: Reliability** (probes, replicas, PDB, spread, HPA) | ❌ | ⚠️ rollout check | ✅ concept | ❌ 1 replica, no probes | ❌ | ✅✅ HPA ≥3, PDB 2, zone + host spread | **06** |
| **L1: Autoscaling / cost** | ❌ | ❌ | ⚠️ Karpenter mentioned | ❌ | ❌ | ✅✅ Karpenter + Spot + interruption queue | **06** |
| **L1: Managed data plane** | ❌ | ❌ | ❌ | ⚠️ MySQL in cluster | ❌ | ✅ RDS/ElastiCache/DynamoDB/SQS, SG from cluster | **06** |
| **L1: CI hardening** | ❌ | ✅ agent split | — | ✅ OIDC (⚠️ Docker Hub token, `contents: write` everywhere) | ❌❌ force-push to main | ⚠️ OIDC, trust `repo:*` | **04 / 06** |
| **L2: Secret scanning** | ❌ | ✅ trivy fs | — | ⚠️ Gitleaks (non-blocking) | ❌ | ❌ | **02** |
| **L2: SCA** | ✅ npm audit | ✅ trivy fs | — | ⚠️ trivy fs (non-blocking) | ❌ | ❌ | **01 / 02** |
| **L2: SAST + quality gate** | ❌ | ✅✅ | — | ⚠️ SonarQube (gate non-blocking) | ⚠️ lint | ❌ | **02** |
| **L2: IaC scan** | ✅ TF + K8s | ❌ | — | ⚠️ Checkov ×3 (non-blocking) | ❌ | ❌ | **01** |
| **L2: Lint / tests** | ✅ | ✅ | — | ⚠️ (non-blocking) | ✅ | ❌ | **01 / 02** |
| **L2: Image scan** | ⚠️ after push | ✅ before push | — | ✅ **before push** (non-blocking) | ❌ | ❌ | **02 / 04** |
| **L2: Gates actually block** | ❌ | ✅ | — | ❌❌ every gate disabled | ⚠️ | — | **02** |
| **L2: SBOM** | ❌ | ❌ | — | ✅ source + image SBOM | ❌ | ❌ | **04** |
| **L2: Signing** | ❌ | ❌ | — | ❌ | ❌ | ❌ | none yet |
| **L2: DAST** | ❌ | ❌ | — | ❌ | ❌ | ❌ | none yet |
| **L2: Deploy verify / rollback** | ❌ | ✅ rollout + rollback | ⚠️ | ✅ rollout status | ⚠️ | ⚠️ Argo self-heal | **02** |
| **L3: GitOps** | ✅ Argo CD | ❌ | — | ❌ kubectl from CI | ✅ Argo CD | ✅ Argo CD auto-sync + prune + self-heal | **06** |
| **L3: Admission (Kyverno / PSA)** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | none yet |
| **Runtime (Falco)** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | none yet |
| **Observability** | ❌ | ❌ | ⚠️ | ⚠️ self-hosted Prometheus + Loki (S3) + Alloy + Grafana; tracing only in a separate demo | ⚠️ shipped, not taught | ✅✅ ADOT → X-Ray, CloudWatch, AMP, AMG | **06** (managed); **04** (self-hosted shape) |
| **Backup / DR** | ❌ | ❌ | ❌ | ⚠️ DR discussed, skipped | ❌ | ⚠️ RDS backups, no final snapshot | none yet |
| **Migration cutover** | — | — | ❌ | — | — | — | none yet |
| **Newbie-friendliness** (explains app + steps) | ⚠️ | ⚠️ | ✅ | ✅ app explained in DevOps terms | ✅ | ✅✅ 17k lines of step READMEs, create/destroy scripts | **06** |
| **Resume / interview prep** | ❌ | ❌ | ⚠️ | ✅ | ✅ | ❌ | **04 / 05** |

## Emerging observations

1. **Every project has one strength and cuts corners elsewhere.** 01 hardens the baseline but doesn't gate; 02 gates properly but has a weak baseline; 03 plans well but has no implementation; 04 has the widest *pipeline* (incl. SBOM, scan-before-push, OIDC per env) but **disables every gate**; 05 has a solid Terraform foundation but weak defaults; **06 is the strongest platform** (Pod Identity, add-ons as code, reliability, Karpenter, observability) but has **no security pipeline at all**.
2. **The final template is roughly "06's platform + 04's pipeline (made blocking) + 02's gates + the guide's Layer 3".** 06 and 04 complement each other almost perfectly.
3. **No project covers Layer 3 (admission) or runtime (Falco), nor signing or DAST.** These come from the guide alone.
4. **Recurring slips worth a "common mistakes" section:** aws-auth vs Access Entries; ClusterRoleBinding scope; "immutable tags"; public EKS API by default; CI writing to `main` unsafely; gates set to `exit-code 0` / `continue-on-error`; **promoting `:latest` instead of the tested digest**; **committing Terraform state (it holds secrets)**; building the same image twice.
5. **Dependency order matters for a newcomer:** probes before readiness gates; replicas/HPA before PDBs; vpc-cni add-on before NetworkPolicy; Metrics Server before HPA; Pod Identity agent before any association; ESO/CSI before managed-DB secrets.
6. **06's structure is the model for the template's format:** incremental sections, each building on the last, with create/destroy scripts and per-section cost estimates.

## Still unanswered (watch for these in 04's videos and 07)

- SBOM + signing + Kyverno verify end to end
- DAST against a real staging environment
- Falco / runtime with alert routing
- Self-hosted observability (04 promises Prometheus/Loki/Alloy/Grafana/Tempo) vs 06's managed AMP/AMG
- Backup / DR (Velero) in practice
- VM → K8s traffic cutover; stateful/database migration
- Multi-cluster / multi-region GitOps (ApplicationSets)
- 04: how its GitHub OIDC trust policy is scoped; how ESO is wired; whether QA/prod really share one cluster
