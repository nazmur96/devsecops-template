# 01: Day-7 | Ultimate End-to-End DevSecOps Project Implementation

| | |
|---|---|
| **Source** | https://www.youtube.com/watch?v=fuiTqI3noTo |
| **Author** | Abhishek Veeramalla |
| **Length** | 3:09:35 |
| **Type** | Full build, with code (repo "journey", `devops` branch) |
| **App** | 3-tier blog: React + nginx reverse proxy / Node.js + Express / PostgreSQL |
| **Stack** | Terraform (AWS VPC + EKS) · Docker multi-stage · Docker Compose · K8s manifests · **GitHub Actions** · **Argo CD** · GHCR |

## What it builds

1. Run the app locally from the developers' README.
2. **Terraform**: VPC, **private subnets**, EKS with the **API not exposed publicly**.
3. **Dockerfiles**: multi-stage, slim base, **non-root user** (old `node:20-alpine` used deliberately so Trivy has something to find).
4. **Docker Compose** for local runs, with secrets kept out of the file.
5. **K8s manifests**: single manifest set; **NetworkPolicies** (only the backend can reach the DB); DB password in a K8s `Secret`; PVC + StorageClass for Postgres.
6. **GitHub Actions** pipeline:
   ```
   lint ─> SCA (npm audit, matrix fe/be) ─> build + push to GHCR ─> Trivy image ─> Checkov (Terraform + K8s) ─> hadolint ─> update K8s manifest
   ```
7. **Argo CD** watches the manifest folder and syncs to EKS (CI = Actions, CD = Argo).

## Coverage against the three-layer guide

| Layer | Coverage |
|---|---|
| **0: Planning** | Partial: starts from the dev team's README, explains the 3-tier architecture |
| **1: Baseline** | ✅ Private VPC/subnets, private K8s API, multi-stage, non-root, NetworkPolicies. ⚠️ plain K8s Secret (base64) |
| **2: Pipeline** | ✅ lint, SCA, Trivy, Checkov, hadolint. ❌ no Gitleaks, no SAST, no SBOM, no signing, no DAST. ⚠️ **non-blocking** |
| **3: Enforcement** | ✅ GitOps (Argo CD). ❌ no admission control, no branch protection shown |
| **Runtime** | ❌ |

## Adopt

- Hardening at **every** layer: infra, image, compose, manifests.
- **GitOps split**: CI builds and updates the manifest; Argo CD deploys. CI never touches the cluster.
- Matrix jobs for multiple services in one repo.
- SCA as its own early stage (`npm audit`; `pip-audit` / equivalents for other languages).
- Checkov over **both** Terraform and K8s manifests.
- Deliberately vulnerable base image to *prove* the gate works (nice demo technique).

## Compromises (don't copy)

| Issue | Better |
|---|---|
| **Pushes the image before scanning** | Scan the local image; push only on pass |
| `continue-on-error: true` on **every** job → failing Trivy/Checkov/hadolint still updated the manifest | Remove it after onboarding; deploy only if all gates pass (he says so himself) |
| Plain K8s Secret | ESO + Secrets Manager / Vault, or SOPS |
| Deploys by tag | Deploy by digest |
| No SAST / secret scanning / SBOM / signing / DAST / runtime | See guide Layer 2 |
| Commits straight to `main` in the demo | PR + branch protection |
