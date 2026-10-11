# 09: "Explain your end-to-end CI/CD pipeline" (interview-answer video)

| | |
|---|---|
| **Source** | A YouTube video (transcript pasted by the user, 2026-10-11); the user rates it one of the best of its kind |
| **Type** | Talk: how to answer the most common DevOps interview question; no code |
| **Stack** | GitHub Actions · ECR (+ Inspector enhanced scanning) · EKS · Helm · Argo CD · Argo Rollouts · SonarQube · Gitleaks · Trivy · External Secrets + Secrets Manager · Prometheus · Fluentd → Elasticsearch/Kibana · Slack |

## The pipeline it describes

1. **Branching + PR checks:** feature branches, never push to `main`; Gitleaks pre-commit **and** in the PR (hooks can be skipped; after merge the secret is already in history); lint, unit tests, SonarQube SAST + quality gate (coverage on new code ≥ 80 %, no new criticals), dependency scan; approval required.
2. **Post-merge auth + caching:** GitHub Actions → AWS via **OIDC** (temporary credentials); dependency and Docker layer caching.
3. **Packaging:** multi-stage Dockerfile, distroless/Chainguard runtime; **Trivy scan on the runner before push**; ECR with **tag immutability**, tag = commit SHA; **ECR enhanced scanning** (Inspector) rescans continuously.
4. **GitOps:** CI never runs `kubectl`; a **separate GitOps repo**, one Helm chart + `values-staging.yaml` / `values-prod.yaml` on **one `main` branch** (no environment branches). **Argo CD renders the chart and applies it, so there's no Helm release; rollback = `git revert`.** **Build once, deploy many.** Staging: a bot PR **auto-merges**, Argo CD syncs to staging in a **separate AWS account**, integration/API tests run. Prod: a promotion PR copies the same tag, and **a ruleset (0 required approvals + code-owner review) + CODEOWNERS on `values-prod.yaml` only** makes just that PR need a lead.
5. **Prod deploy:** rolling updates are already zero-downtime with good readiness probes, but they ship a bug to everyone, so **Argo Rollouts canary** (10 %). Neither Argo CD nor Rollouts talks to AWS: Rollouts writes weights to the Ingress, and the **Load Balancer Controller** sets **weighted target groups** on the ALB, so the split is exact, per request. Secrets via ESO from Secrets Manager. DB: **expand/contract**, backward-compatible migrations.
6. **Rollback + monitoring:** Prometheus only collects; Rollouts queries it every minute (5xx rate, p99); above 2 % errors it aborts and resets weights to 100 % stable; failed canary pods kept a few minutes for inspection; logs in Elasticsearch/Kibana; Slack alert with commit details.

## Coverage against the template

| | Video | Template |
|---|---|---|
| Pre-commit + PR secret scanning | ✅ | P1-4, P7-2 |
| SAST + quality gate | ✅ SonarQube | P7-2 (Semgrep or SonarQube) |
| IaC scanning (Checkov) | ❌ (Trivy/SonarQube *could*, not mentioned) | P7-2 Checkov |
| DAST (ZAP) | ❌ nothing equivalent | P10-6 |
| OIDC, caching | ✅ | P5-2, P7-1 |
| Scan before push, immutable SHA tags, Inspector | ✅ | P7-3, P4-7 |
| SBOM, signing, admission verification | ❌ | P7-3, P8-6 |
| Separate GitOps repo, one branch, folders/values per env | ✅ | P1-1, P7-5 |
| Auto-merge staging, CODEOWNERS-gated prod | ✅✅ | P7-5 (added from this video) |
| Argo CD renders Helm (no release) | ✅ | P8-1 (added) |
| Canary with metric analysis, ALB weighted TGs | ✅✅ | P8-8 (Gateway API plugin) |
| Keep failed canary pods for inspection | ✅ | P8-8 (added) |
| Expand/contract migrations | ✅ (concept) | P8-3b (full ticket) |
| Staging in a separate AWS account | ✅ | P0-6 (accounts option) |
| Logs | Fluentd → Elasticsearch/Kibana | CloudWatch Logs (P10-2); EFK is another valid self-hosted option |

## Adopt

1. The **interview answer structure**: one stack, walked in order, hitting security → tagging → safe deploy → rollback (now in P11-5).
2. **Ruleset + CODEOWNERS** trick: lower environments auto-merge, prod needs approval, all on one branch.
3. **"Promotion" vs "promote"** wording.
4. **Who talks to AWS:** only the Load Balancer Controller (and other controllers via Pod Identity), never Argo CD.
5. **Prometheus doesn't roll back**; Rollouts' analysis does.

## Gaps

No IaC scanning, no DAST, no SBOM/signing/admission control, no runtime security, no mention of NetworkPolicy or reliability settings beyond readiness probes. That's fine for a 10-minute interview answer, but not a complete platform.
