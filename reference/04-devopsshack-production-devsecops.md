# 04: Production-Grade End-to-End DevSecOps Project (DevOpsShack)

| | |
|---|---|
| **Source** | Bilibili mirror: https://www.bilibili.com/video/BV1R7jv6vEJg (42 parts, ~9 h 11 min, all transcribed); repo: https://github.com/jaiswaladi246/3-Tier-User-Platform-Project |
| **Author** | Aditya Jaiswal (DevOpsShack); originally a Udemy course |
| **App** | 3-tier "user management" CRUD: React + webpack (client) · Node/Express (server) · MySQL 8 |
| **Stack** | GitHub Actions (QA CI/CD + prod CD) · Gitleaks · Checkov · Trivy fs/image · SonarQube · Syft SBOM · Docker Hub · EKS via **eksctl CLI** · GitHub OIDC → EKS Access Entries · ALB controller · ACM · Route 53 (Namecheap domain) · Secrets Manager + ESO · EBS CSI · MySQL StatefulSet · **self-hosted** Prometheus + Loki (S3) + Alloy + Grafana + kube-state-metrics; Tempo in a separate demo |

## What the repo actually does

**QA pipeline** (`qa-cicd.yml`, push to `qa`):
```
gitleaks → [checkov tf | checkov k8s | checkov dockerfile | trivy fs client | trivy fs server] (parallel)
→ lint client/server → test client/server → SonarQube + quality gate → client build
→ docker build (load) → trivy image → SBOM (source + image, Syft) → push (sha + latest)
→ update qa manifest (commit to qa) → deploy: OIDC → kubectl apply -n qa → rollout status
```

**Prod pipeline** (`prod-cd.yaml`, push to `main` touching the prod manifest): pull `:latest` → retag `prod-<sha>` → push → update prod manifest → OIDC (separate role) → `kubectl apply -n prod` → rollout status.

| Area | Finding |
|---|---|
| Gates | ❌❌ **Every gate disabled**: Gitleaks `--exit-code 0`, Checkov `|| true`, Trivy `--exit-code 0`, lint/tests/quality gate `continue-on-error: true` |
| Scan order | ✅ **Image scanned before push** (`load: true` → Trivy → push) |
| SBOM | ✅ Source + image SBOMs (Syft, SPDX), as artifacts (not attached/signed) |
| Parallelism | ✅ Five security jobs in parallel after Gitleaks |
| Tags | ✅ `github.sha`. ⚠️ also `latest` |
| **Promotion** | ❌ Prod pulls **`:latest`** and retags it (confirmed in part 39). If a newer QA build lands after sign-off, an **untested image** goes to prod. Should promote the QA-approved **digest** |
| Deploy | ⚠️ Push model: CI runs `kubectl apply` (no GitOps). ✅ GitHub OIDC, **separate IAM roles for QA and prod**, `kubectl auth can-i` checks before deploying |
| Environments | ⚠️ QA and prod are **namespaces in one cluster** (`my-cluster` in both workflows), although part 8 recommends a separate prod cluster |
| Ingress | ✅ ALB, `target-type: ip`, **HTTPS + ACM + ssl-redirect** from the start |
| Manifests | ❌ 1 replica, **no probes, no resources, no securityContext** (the videos *promise* probes and limits) |
| Dockerfile | ⚠️ non-root user, but **single-stage** (build tools ship); `npm install` not `npm ci`; `npm start` as PID 1 |
| Secrets | ✅ DB creds via `secretKeyRef` to an ESO-synced Secret. ❌ App falls back to `root`/`password` if env missing |
| Registry | ⚠️ Docker Hub + long-lived token; `imagePullSecrets: regcred` |
| CI hygiene | ⚠️ `contents: write` at workflow level; `gitleaks:latest`, `sonarqube-scan-action@master`, Trivy `version: latest`; Node 18 in CI (EOL) vs Node 20 in the image; Checkov-Terraform scans a repo with **no Terraform** |

## What the videos add

| Part(s) | Content |
|---|---|
| **1** | Full flow: environments, branching, promotion, monitoring (metrics, logs, traces), resume with quantified metrics |
| **2–5** ★ | Explains the **app** in DevOps terms (React vs Node vs MySQL, why the browser never talks to the DB, request flow across the 3 tiers). The best "understand the app first" model of all references |
| **8** ★ | **Five environments** (dev, qa, ppd, prod, dr) mapped onto **two clusters** (non-prod cluster with dev/qa/ppd namespaces + a prod cluster); **no always-on DR cluster** (not worth running 365 days for cluster-level failures). Adopted in the template's P0-6 |
| **9** | Branching: `main` + a long-lived `qa` branch; features branch from `qa`; bug-fix branches loop back to `qa`; QA sign-off → merge `qa` → `main` |
| **14** | CI/CD flow diagram: Jira ticket → feature branch → PR → QA pipeline (parallel scans) → testing feedback loop → prod CD by retag, "CD only" because artifacts were validated in QA |
| **28** | IAM user with **AdministratorAccess + static keys**; Windows tool install; cluster by **`eksctl create cluster` flags** (3 × t3a.xlarge, min 3 / max 4; notes min/max needs an autoscaler) |
| **29** ★ | **OIDC theory, the best segment of any project:** four scenarios from worst to best (admin kubeconfig → SA token → runtime token minted with AWS keys → **GitHub OIDC → IAM role → Access Entry → `AmazonEKSEditPolicy` scoped to the namespace**). Trust pinned to repo + branch; permission policy only `eks:DescribeCluster`. Slip: calls the JWT "encrypted" (it's **signed**) |
| **30** | OIDC practical, all imperative AWS CLI; distinguishes GitHub OIDC provider from the cluster (IRSA) one |
| **31** | ALB controller: IRSA via `eksctl` + Helm (imperative) |
| **32** | MySQL theory: StorageClass/EBS, StatefulSet vs Deployment, headless Service. ⚠️ Implies StatefulSet replicas **stay in sync with automatic failover**; they don't without MySQL replication / an operator |
| **33** | ESO with IRSA, **namespaced `SecretStore`** in `qa`, `ExternalSecret` (1 h refresh); EBS CSI via eksctl + add-on CLI |
| **34** | Domain from Namecheap → Route 53 hosted zone (NS delegation) → ACM certificate → Ingress |
| **35–39** | CD for QA and prod; prod OIDC role and prod DB setup mirror QA; prod promotion = pull `:latest`, retag `prod-<sha>` |
| **40** | **Monitoring on EKS, self-hosted**: Helm values per component, applied by a **PowerShell script** (not GitOps). Prometheus with kube-state-metrics, **2 Gi EBS** storage and the remote-write receiver enabled (so Alloy can push); **Loki storing logs in S3** (IAM policy for the bucket); **Alloy** reading pod logs via the Kubernetes API → Loki gateway; **Grafana** deployed separately, exposed through the ALB Ingress. No alerting, no retention/HA discussion |
| **41** | **Tracing demo, separate from the project**: an EC2 VM running Docker Compose (Alloy + Tempo + Prometheus + Loki + Grafana) with Grafana's QuickPizza demo app sending OTLP (port 4318). Explains telemetry, OTLP and Alloy as "collector + router" well, but the project's own app is **not instrumented** |
| **42** | Resume: quantified bullets (e.g. self-hosted runners cutting pipeline time 5 → 2.5 min, parallel jobs), DevSecOps tools, private registry, SHA tagging, secretless OIDC, promotion lower → upper environment |

## Coverage against the three-layer guide

| Layer | Coverage |
|---|---|
| **0: Planning** | ✅ environments (5 → 2 clusters), branching, app walkthrough; resume metrics |
| **1: Baseline** | ✅ HTTPS, ESO + Secrets Manager, OIDC per env, non-root image. ❌ no IaC (all CLI), admin keys, no probes/resources/securityContext, no NetworkPolicy, QA and prod share a cluster |
| **2: Pipeline** | ✅✅ widest of all (Gitleaks, Checkov ×3, Trivy fs + image, lint, tests, SonarQube, SBOM, scan-before-push, parallel jobs) but ❌❌ **nothing blocks** |
| **3: Enforcement** | ❌ no GitOps, no admission |
| **Runtime / observability** | ⚠️ self-hosted metrics + logs on EKS (Loki on S3 is a good touch); tracing only as a separate demo; no alerting or SLOs |

## Adopt

1. **The 4-scenario OIDC explanation** (p29) as the way to teach CI → cluster auth.
2. **Five environments on two clusters**, DR rebuilt on demand (p8).
3. **Scan the local image before pushing** + **SBOM for source and image**.
4. **Parallel security jobs** after a fast secret scan.
5. **Separate OIDC role per environment** with namespace-scoped access entries and `auth can-i` pre-checks.
6. **Namespaced `SecretStore`** (scoped) rather than one cluster-wide store.
7. **Explaining the app** before the infrastructure (p3–5).
8. **Loki on S3** and **Alloy as collector + router**: the shape of the template's local stack (P3-10) and later self-hosted track (P10-13).
9. **Quantified resume metrics** as a deliverable.

## Compromises (don't copy)

Gates disabled; promoting `:latest`; push-model deploys; QA and prod in one cluster (contradicting its own part 8); everything created by CLI (`eksctl`, `aws`, `helm`, a PowerShell install script) with no IaC or GitOps; admin IAM user with static keys; bare manifests; single-stage Node image; Docker Hub token; the "StatefulSet replicas stay in sync" claim; tracing never connected to the real app; no alerting.
