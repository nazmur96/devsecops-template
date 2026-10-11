# 02: DevSecOps Mega Project | Multi-AZ Amazon EKS + Jenkins + Trivy + SonarQube + ECR + GitHub

| | |
|---|---|
| **Source** | https://www.youtube.com/watch?v=9LXZm0Fryfw |
| **Author** | Cloud With VarJosh (Varun Joshi) |
| **Length** | 3:39:14 |
| **Type** | Full build, with code (repo "Jenkins basics to production", project 1) |
| **App** | Simple Java "welcome" app (Maven) |
| **Stack** | **Jenkins** (controller + agent on EC2, over SSH) · **SonarQube** (+ external Postgres) · Trivy · Docker · **ECR** · **EKS** via `eksctl` (2 AZs, managed node group) · IAM + `aws-auth` + RBAC |

## What it builds

1. **Jenkins controller and a separate agent** (EC2, SSH): the controller orchestrates, the agent runs jobs.
2. **SonarQube** with an external PostgreSQL database; **quality gates** configured (vulnerabilities, coverage).
3. Jenkins pipeline:
   ```
   checkout (private repo, credentials) ─> trivy fs (SCA + OS + secrets, FAIL on HIGH/CRITICAL)
   ─> mvn build + test ─> SonarQube analysis ─> quality gate (FAIL)
   ─> ECR login + docker build ─> trivy image (FAIL on HIGH/CRITICAL) ─> push to ECR (build# tag + latest)
   ─> deploy to EKS (kubectl apply) ─> rollout status check ─> auto-rollback on failure ─> post actions
   ```
4. **EKS** with `eksctl`: 2 AZs, OIDC enabled, managed node group across AZs.
5. **Access for Jenkins**, the most instructive part:
   - IAM inline policy on the agent role (`eks:DescribeCluster`), then `aws eks update-kubeconfig`.
   - First deploy fails with **Unauthorized**: being authenticated by IAM isn't the same as being authorized in K8s.
   - **Authentication**: add the agent role to the `aws-auth` ConfigMap (`groups: system:authenticated`, *not* `system:masters`).
   - **Authorization**: ClusterRole `jenkins-deployer-role` (deployments, replicasets, pods, services, namespaces) + ClusterRoleBinding to that user.

## Coverage against the three-layer guide

| Layer | Coverage |
|---|---|
| **0: Planning** | Diagram of the pipeline and cluster; prerequisite knowledge listed |
| **1: Baseline** | ✅ controller/agent separation, IAM least privilege (he says so), credential bindings, multi-AZ nodes. ⚠️ **public** worker nodes, **NodePort**, **SSH allowed**, private IPs instead of DNS (all admitted shortcuts) |
| **2: Pipeline** | ✅✅ **Gates actually fail the build** (three: trivy fs, quality gate, trivy image); SAST; rollout check + rollback. ❌ no Checkov, SBOM, signing, DAST |
| **3: Enforcement** | ❌ None. Lists Kyverno, signed images, digests, Vault/SM, Argo CD, policy as code, observability as future work |
| **Runtime** | ❌ |

## Adopt

- **Blocking gates with per-app thresholds** ("70% coverage OK for app 1, 90% for app 2").
- `trivy fs` as stage 1: SCA, OS packages and secrets in one pass.
- Clear distinction between **SCA** (dependencies), **SAST** (your code) and **image scan** (base image), and why you need all three.
- **Rollout status + automatic rollback** as a deploy gate.
- **Controller/agent split** in CI.
- The explanation that **authentication and authorization are separate steps** (IAM → K8s identity → RBAC).
- Post actions: notifications, tickets, metrics on failure.
- His improvement list doubles as a checklist: Ingress, multibranch + protected branches, Argo CD, DNS names, **digests**, **signing + Kyverno**, **Vault/SM**, observability, policy as code, Argo Rollouts, cleanup and cost.

## Compromises (don't copy)

| Issue | Better |
|---|---|
| **`aws-auth` ConfigMap** | **EKS Access Entries** (guide §5.2) |
| **ClusterRoleBinding called "namespace-wide"** | It's **cluster-wide**; use a RoleBinding in the namespace (or a namespace-scoped access policy) |
| Says tags are "immutable" | They're mutable, which is why to deploy by **digest** |
| Pushes `latest` as well as the build number | Immutable tags (ECR setting) + digest |
| Public nodes, NodePort, SSH allowed | Private nodes, ALB Ingress + TLS, no SSH (SSM if needed) |
| CI holds cluster credentials and runs `kubectl apply` | GitOps (Argo CD pulls), so CI needs no cluster access |
| Long-lived Jenkins credentials | OIDC / instance role; Vault / Secrets Manager |
| Jenkins on EC2 you operate yourself | Fine for learning; managed CI cuts the CI attack surface you have to patch |
