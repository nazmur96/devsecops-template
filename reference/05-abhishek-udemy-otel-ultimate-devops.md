# 05: Ultimate DevOps Project and Resume Preparation (Udemy)

| | |
|---|---|
| **Source** | https://www.udemy.com/course/ultimate-devops-project-with-resume-preparation/ |
| **Author** | Abhishek Veeramalla |
| **Length** | 13 sections, about 11.5 h (curriculum from Udemy's public API) |
| **Type** | Full build, paid course |
| **App** | **OpenTelemetry Astronomy Shop**: ~20 microservices in Go, Java, Python, .NET, Node, Rust, etc., plus Kafka, Valkey, flagd, Envoy frontend-proxy |
| **Stack** | Docker · Docker Compose · **Terraform** (S3 + DynamoDB backend, VPC + EKS modules) · EKS · AWS Load Balancer Controller · Route 53 · **GitHub Actions** · **Argo CD** |
| **Your workspace** | `~/Projects/open-telemetry-learning` (notes for sections 1–10, `AGENTS.md` holding your Section 11 and 12 improvement lists) |
| **Related** | `~/projects/eks-lifecycle-notes` (phases 0–10, the source of the "Phase N" labels in your Section 12) |

## What the course builds, section by section

| § | Topic | What's actually built |
|---|---|---|
| 1–2 | Intro, architecture | Walkthrough of the microservices |
| 3 | Prerequisites | AWS account; **IAM user with AdministratorAccess + access keys**; EC2 workstation (t2.large) with Docker/kubectl/Terraform |
| 4 | Run locally | On EC2; open security group ports to reach it |
| 5 | Containerise | Dockerfiles for 3 services (Go product-catalog, Java ad, Python recommendation) incl. multi-stage; `docker init`; push to registry |
| 6–7 | Compose; why K8s | Concepts |
| 8 | **Terraform** | Remote state in S3 + DynamoDB lock. Custom VPC module: 3 AZs, public + private subnets with ELB tags, **NAT gateway per AZ**. EKS module: cluster + managed node group (t3.medium ×2) **in private subnets** |
| 9 | Deploy to K8s | ServiceAccount, Deployments, Services; LoadBalancer type and why it's wasteful → **Ingress + AWS LB Controller** (OIDC + IRSA via `eksctl`, Helm install) → ALB `internet-facing`, `target-type: ip` |
| 10 | Custom domain | Route 53 hosted zone, nameservers, Ingress host. **HTTP only** |
| 11 | CI/CD | GitHub Actions CI for **one** service (product-catalog): build + test, golangci-lint, docker build/push (Docker Hub, tag `github.run_id`), `sed` the manifest, commit to main. **Argo CD** installed and syncing the manifests (GitOps) |
| 12–13 | Resume + interview | Resume bullets for experienced/fresher; "tell me about the project", day-to-day, challenges |

## Audit of the actual code

| Area | Finding |
|---|---|
| EKS API endpoint | `vpc_config` sets only `subnet_ids`, so the endpoint is **public, open to 0.0.0.0/0**, with private access off |
| Cluster access | No `access_config`, so the cluster creator gets invisible admin; no Access Entries |
| Add-ons | VPC CNI / CoreDNS / kube-proxy left as defaults, not `aws_eks_addon` in Terraform, so you can't turn on VPC CNI network policy declaratively |
| LB Controller | Installed **imperatively** (`eksctl` + `helm` + a manual IAM policy patch), not in Terraform or Argo CD, so it's drift from day one |
| Manifests | 19 services, **every one `replicas: 1`**, **no readiness/liveness probes at all**, resources = **memory limits only, no requests**; some services carry `runAsNonRoot` from the upstream Helm chart, others nothing; images pinned to upstream `1.12.0` |
| Ingress | `internet-facing`, `target-type: ip`, no TLS, no WAF |
| CI workflow | Triggers on `pull_request`, yet pushes the image **and runs `git push origin HEAD:main -f`**. On a PR, `HEAD` is the PR's merge commit, so **an unreviewed PR force-pushes itself onto `main`**, and concurrent PRs overwrite each other. Long-lived Docker Hub token in secrets; no `permissions:` block; outdated actions (`setup-go@v2`, `setup-buildx@v1`); tags not pinned to SHAs; no scanning of any kind |
| Identity | Admin IAM user with static access keys (you replaced this with Authentik SSO ✅) |

## Coverage against the three-layer guide

| Layer | Coverage |
|---|---|
| **0: Planning** | ⚠️ Architecture walkthrough; "steps to onboard Kubernetes" lecture |
| **1: Baseline** | ✅ private nodes, NAT per AZ, 3 AZs, remote state + locking, Ingress instead of LB-per-service, IRSA for the controller. ❌ public API, no probes/requests/replicas, no NetworkPolicy, no TLS, admin access keys |
| **2: Pipeline** | ⚠️ build + test + lint for 1 of ~20 services. ❌ no secret/IaC/image scanning, SAST, SBOM, signing. ❌❌ the force-push to main from a PR |
| **3: Enforcement** | ✅ Argo CD (GitOps). ❌ no admission, no branch protection |
| **Runtime / observability** | ❌ not taught, even though the OTel demo ships a collector, Jaeger, Prometheus and Grafana you could use for free |

## Adopt

- Terraform **remote state + DynamoDB locking** before any infra, and **modules** for VPC/EKS.
- VPC design: 3 AZs, **NAT per AZ**, correct ELB subnet tags, nodes in private subnets.
- The **LoadBalancer → Ingress** progression and why one ALB beats one LB per service.
- `target-type: ip` (ALB → pod directly; what makes readiness gates matter).
- A realistic polyglot app (~20 services, Kafka, feature flags) instead of a toy, which makes NetworkPolicy and SLO work non-trivial.
- Ends with resume and interview prep (like project 04).

## Compromises (don't copy)

See the audit table. Biggest: CI force-push from a PR, public API endpoint, HTTP only, zero probes, no requests, single replicas, imperatively installed controllers, no security scanning.

---

## Review of your improvement list

Overall: **the list is accurate, well ordered, and strong on reliability and the
edge layer.** Every item is real, and the gotchas you noted (probe trap,
eviction vs ALB timing, CloudFront cert in us-east-1) are ones most courses
miss. The adjustments below are mostly about **missing prerequisites**, **one
security hole the list doesn't mention**, and the **DevSecOps pipeline being
almost absent**.

### Section 11 (edge)

| Item | Verdict | Notes |
|---|---|---|
| HTTPS + ACM | ✅ Correct; make it **P0** | A public domain over HTTP shouldn't exist even in a demo. Add `ssl-redirect: '443'` and a modern `ssl-policy` (e.g. `ELBSecurityPolicy-TLS13-1-2-2021-06`) |
| Readiness gates | ✅ Correct, but **blocked** | The manifests have **no readiness probes**. A readiness gate makes the pod wait for ALB health, but without a probe the pod is "Ready" the moment the process starts, so add probes first. Label: `elbv2.k8s.aws/pod-readiness-gate-inject=enabled` on the namespace |
| Security group chaining | ⚠️ Reword | With `target-type: ip` and no *Security Groups for Pods*, pods use the **node's** security group. The controller adds a rule letting the ALB's backend SG reach the node/ENI SG on the target port. So you verify "**nodes** accept app traffic only from the ALB SG". Pod-to-pod isolation inside the cluster is **NetworkPolicy's** job, not SGs' |
| WAF | ✅ | ~$5/mo per web ACL + ~$1 per rule + per-request. AWS managed Core + KnownBadInputs + a rate rule. Create and delete in the same session is right |
| CloudFront | ✅ optional | Add the missing half: **lock the ALB to CloudFront** (the CloudFront origin-facing managed prefix list in the ALB SG + a secret custom header checked by the ALB rule), otherwise attackers skip CloudFront and WAF by hitting the ALB directly |
| Shield / NLB / PrivateLink | ✅ | Correct to skip |

### Section 12 (platform)

| # | Item | Verdict | Notes |
|---|---|---|---|
| 1 | Pod Identity | ✅ | Most demo services never call AWS, so pick a **real** use: the **OTel collector exporting to CloudWatch / X-Ray** genuinely needs AWS permissions. Also **migrate the LB Controller from IRSA to Pod Identity**, which shows both mechanisms. Needs the `eks-pod-identity-agent` add-on |
| 2 | Run-it-well | ✅ Correct; **split it** | Order matters: **(a) probes + requests** (none exist today) → (b) replicas 3 → (c) PDB → (d) topology spread → (e) drain test. ⚠️ A PDB with `minAvailable: 1` on a **1-replica** Deployment **blocks node drains forever**, so scale before adding PDBs |
| 3 | NetworkPolicy | ✅ | Prerequisite: manage `vpc-cni` as an `aws_eks_addon` with `enableNetworkPolicy: "true"`. Expect the allow-list to be big: every service talks to the **otel-collector** and **flagd**, plus Kafka. Good realistic practice |
| 4 | VPC endpoints + system node group | ✅ | Cost: **S3 (and DynamoDB) gateway endpoints are free**; interface endpoints (ECR api/dkr, STS, Logs) are about $7/mo **per AZ each**, so ~$90/mo across 3 AZs. Fine for a session, not to leave running |
| 5 | More controllers | ✅ | Metrics Server is a **prerequisite for HPA**. Install controllers **via Argo CD** (app-of-apps), not `helm install`, and move the LB Controller there too |
| 6 | CI hardening | ⚠️ **Under-prioritised; move to P0/P1** | See "What the list misses" #2. SHA tags + Trivy are right, but the workflow has a much bigger problem |
| 7 | Light data layer | ✅ | Pair with ESO (item 5) and Pod Identity (item 1): RDS/DynamoDB + SG locked to the right pods + password via ESO from Secrets Manager |
| 8 | Separate envs/accounts | ✅ | Fine as P3 for a learner |
| 9 | Rollback + preStop | ✅ | Also lower the target group's **deregistration delay** (default 300 s) to ~30 s via annotation, and keep `terminationGracePeriodSeconds` > preStop sleep |
| 10 | Eviction vs ALB timing | ✅ | Correct: the default `tolerationSeconds: 300` for `not-ready`/`unreachable` taints vs ALB health checks failing in seconds |
| 11 | Alerting/SLOs, backups/DR, upgrades, cost | ✅ | The OTel demo already ships **Prometheus + Grafana + Jaeger**. Build SLOs on that telemetry; it's the point of this app |

### What the list misses

1. **The EKS API endpoint is public to the internet.** Restrict `public_access_cidrs` to your IP, or enable private access. Add **Access Entries** (`authentication_mode = "API"`) so the creator's admin is explicit. One-line Terraform changes, high value. **P0.**
2. **The CI force-pushes unreviewed PRs to `main`.** Fix: build/test on `pull_request`; build/push/update-manifest only on `push` to `main`; no `-f`; a `permissions:` block; branch protection; OIDC or a GHCR `GITHUB_TOKEN` instead of a Docker Hub token; actions pinned to SHAs; CI for all services (matrix), not one. **P0.**
3. **The whole DevSecOps pipeline (guide Layer 2).** Gitleaks, Checkov over `eks-install/` and `kubernetes/`, Trivy fs + image as **blocking** gates, SBOM, Cosign signing. Your item 6 is the seed of this.
4. **Admission and pod security (guide Layer 3).** PSA labels (`baseline` first, then `restricted`), consistent `securityContext` across all 19 services, Kyverno (verify signatures, require probes/requests, which also **enforces** your item 2), Falco.
5. **Add-ons and controllers as code.** `aws_eks_addon` for vpc-cni / coredns / kube-proxy / pod-identity-agent; LB Controller via Terraform or Argo CD rather than `eksctl` + `helm` + a hand-patched IAM policy.
6. **Argo CD hardening.** Rotate or delete the initial admin secret, don't expose the UI publicly, restrict `AppProject` destinations, auto-sync + self-heal.

### Suggested order

```
P0  (before anything is public)
    HTTPS + ACM · restrict API endpoint + Access Entries · fix CI (no force-push, push-only on main, permissions, branch protection)
P1  (reliability foundation)
    probes + requests → replicas → PDB → topology spread → drain test
    → readiness gates (+ deregistration delay, preStop)            ← needs probes
    vpc-cni add-on in TF → NetworkPolicy (deny-all → DNS → collector/flagd → real calls)
P1b (DevSecOps pipeline)
    Gitleaks · Checkov · Trivy fs/image as blocking gates · SHA tags · SBOM · Cosign
P2  Pod Identity (otel-collector → CloudWatch/X-Ray; LBC migration)
    Metrics Server → HPA · ESO · ExternalDNS · controllers via Argo CD
    VPC endpoints (gateway free; interface per session) · system node group
    SG chaining check · WAF · light data layer (RDS/DynamoDB + ESO)
P3  PSA restricted + Kyverno + Falco · SLOs on the built-in Prometheus/Grafana/Jaeger
    eviction timing · backups/DR · upgrades (1.30 → current) · cost · envs/accounts · CloudFront (+ ALB lock-down)
```
