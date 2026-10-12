# Phase 7: Security pipeline (CI)

**Goal:** every change is checked cheapest-first; every image is built
**once**, scanned **before** it's pushed, given an SBOM, **signed**, pushed by
digest, and promoted through environments **by that same digest**. Every gate
**fails closed**.

**Why:** across the six reference projects:
- 01 pushed before scanning, and set `continue-on-error` on everything
- 04 had the widest pipeline (Gitleaks, Checkov, Trivy, SonarQube, SBOM), but **every gate was disabled** (`--exit-code 0`, `|| true`)
- 05 had no scanning, and a workflow that could force-push unreviewed PRs to `main`
- 06 had no scanning at all, and built every image twice

Only 02's gates actually blocked anything. This phase is 04's pipeline, made
real, plus signing and promotion by digest.

---

## The pipeline at a glance

```
PULL REQUEST (fast, no secrets, no push)
  gitleaks ─┐
  lint      ├─ parallel ─→ unit tests + coverage ─→ SAST + quality gate
  hadolint  │
  checkov   │
  trivy fs ─┘

MERGE TO main (per changed service, matrix)
  build (load locally) → trivy image (gate) → size budget (gate) → SBOM
     → push to ECR (OIDC) → record digest → cosign sign + attest SBOM/provenance
     → commit digest to gitops repo envs/dev   ──→ Argo CD deploys (P8)

PROMOTION (no rebuild)
  PR to gitops: copy dev's digest → envs/staging → (ZAP scan, P10) → envs/prod
  prod requires review + GitHub environment approval
```

---

## P7-1 Pipeline hygiene (applies to every workflow)

**Type:** Story

- [ ] **Third-party actions pinned to a commit SHA**, with the version as a comment: `uses: actions/checkout@<40-char-sha> # v4`. In March 2025 the `tj-actions/changed-files` action was compromised, and pipelines that referenced it by tag leaked their secrets. Let Renovate/Dependabot bump the SHAs
- [ ] Top-level `permissions: { contents: read }`; jobs add only what they need (`id-token: write` for OIDC)
- [ ] Tools installed from the repo's **`mise.toml`** (`jdx/mise-action`), so CI runs exactly the versions developers run
- [ ] **No `continue-on-error`, no `|| true`, no `--exit-code 0` on a gate.** A new scanner may run in report-only mode for at most 2 weeks, with a ticket to make it blocking
- [ ] `concurrency:` set so two runs on the same branch don't race
- [ ] **Caching**, so builds stay fast: dependency caches (`actions/setup-node` / `setup-java` / `setup-go` with `cache:`), Docker **BuildKit layer cache** (`--cache-from`/`--cache-to type=gha`, or a registry cache in ECR), and Trivy's vulnerability DB cached between runs. Target: an incremental build under 5 minutes
- [ ] All dependencies resolved through the **artifact repository** (P4-8: CodeArtifact or Nexus); base images through the ECR pull-through cache. CI gets the CodeArtifact token from its OIDC role at run time; nothing stored
- [ ] Workflows on `pull_request` never push images, never write to repos, and never get cloud credentials (forks can't be trusted)
- [ ] **Working on a fork (P1-0)?** Delete or replace the original project's `.github/workflows/*.yml` with yours **first**, then turn Actions back on (`gh api -X PUT repos/<you>/<repo>/actions/permissions -F enabled=true`). In the other order, the original author's release job runs in your repo

---

## P7-2 Pull-request checks

**Type:** Story

```yaml
# .github/workflows/pr.yml (excerpt)
name: pr
on: { pull_request: { branches: [main] } }
permissions: { contents: read }
concurrency: { group: pr-${{ github.ref }}, cancel-in-progress: true }

jobs:
  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha> # v4
        with: { fetch-depth: 0 }                      # full history for gitleaks
      - uses: jdx/mise-action@<sha> # v2
      - run: gitleaks git --redact --verbose .        # non-zero exit on findings

  static:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha> # v4
      - uses: jdx/mise-action@<sha> # v2
      - run: hadolint $(git ls-files '**/Dockerfile')
      - run: checkov -d . --framework dockerfile,kubernetes,helm,terraform,github_actions --quiet
      - run: trivy fs --scanners vuln,secret,misconfig --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 .

  test:
    runs-on: ubuntu-latest
    strategy: { matrix: { service: [catalogue, carts, front-end, user, payment, orders, shipping, queue-master] } }
    steps:
      - uses: actions/checkout@<sha> # v4
      - uses: jdx/mise-action@<sha> # v2
      - run: make -C services/${{ matrix.service }} lint test   # each service's own lint + tests + coverage

  sast:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@<sha> # v4
        with: { fetch-depth: 0 }
      - uses: jdx/mise-action@<sha> # v2
      - run: semgrep scan --config p/default --error  # or SonarQube/SonarCloud with qualitygate.wait=true
```

| Check | Tool | Gate | Catches |
|---|---|---|---|
| Secrets (history) | Gitleaks | any finding | keys, tokens, passwords |
| Dockerfile | hadolint | errors | root user, unpinned packages, `latest` |
| IaC | Checkov (Terraform, K8s, Helm, Dockerfile, Actions) | failed checks | public buckets, missing securityContext, wildcard IAM |
| Dependencies + misconfig | `trivy fs` | HIGH/CRITICAL with a fix | vulnerable libraries in lockfiles |
| Lint + unit tests | per language (golangci-lint, ESLint, Checkstyle) | errors; coverage threshold | bugs |
| SAST | Semgrep or SonarQube quality gate | new critical issues / coverage below threshold | SQL injection, XSS paths in **your** code |

> **Why secrets are scanned in the PR, not after merge** (reference 09): once a
> commit with a secret is merged to `main`, it's in the Git history, cloned
> by every developer and CI runner, and possibly forked. Deleting the file
> later doesn't remove it from history. Catch it in the PR. If one gets
> through anyway: **rotate the secret first** (assume it's stolen), then
> check its usage logs (e.g. CloudTrail for AWS keys), then rewrite history
> (`git filter-repo`) if you must. Rotation is the fix; history rewriting
> is only tidying up.

- [ ] Every job is a **required status check** in branch protection
- [ ] Path filters so a docs change doesn't build eight Java services
- [ ] **Prove each gate fails:** a PR with a fake key, an unpinned `apk add`, a public S3 bucket, a vulnerable dependency, and a deliberate SQL injection. All five are blocked

---

## P7-3 Build once, scan before push, push by digest

**Type:** Story

```yaml
# .github/workflows/release.yml (one service; run as a matrix over changed services)
name: release
on: { push: { branches: [main] } }
permissions: { contents: read }

jobs:
  build:
    runs-on: ubuntu-latest
    permissions: { contents: read, id-token: write }   # OIDC to AWS and Sigstore
    env:
      SERVICE: catalogue
      ECR: <account>.dkr.ecr.eu-central-1.amazonaws.com
    steps:
      - uses: actions/checkout@<sha> # v4
      - uses: jdx/mise-action@<sha> # v2

      - name: Build once, keep it local
        run: |
          docker build \
            --label org.opencontainers.image.revision=$GITHUB_SHA \
            --label org.opencontainers.image.source=$GITHUB_SERVER_URL/$GITHUB_REPOSITORY \
            -t ci/$SERVICE:$GITHUB_SHA services/$SERVICE

      - name: Scan before push (gate)
        run: trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ci/$SERVICE:$GITHUB_SHA

      - name: Size budget (gate)
        run: scripts/check-image-size.sh ci/$SERVICE:$GITHUB_SHA 30

      - name: SBOM
        run: syft ci/$SERVICE:$GITHUB_SHA -o cyclonedx-json > sbom.json

      - uses: aws-actions/configure-aws-credentials@<sha> # v4
        with:
          role-to-assume: arn:aws:iam::<account>:role/sockshop-ci-build
          aws-region: eu-central-1
      - uses: aws-actions/amazon-ecr-login@<sha> # v2

      - name: Push (immutable tag = git SHA) and capture the digest
        id: push
        run: |
          REF=$ECR/sockshop/$SERVICE:$GITHUB_SHA
          docker tag ci/$SERVICE:$GITHUB_SHA $REF
          docker push $REF
          DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' $REF)
          echo "image=$DIGEST" >> "$GITHUB_OUTPUT"

      - name: Sign + attest (keyless, OIDC identity of this workflow)
        run: |
          cosign sign --yes ${{ steps.push.outputs.image }}
          cosign attest --yes --type cyclonedx --predicate sbom.json ${{ steps.push.outputs.image }}
```

- [ ] One build per image; **no** second `docker build` for a second tag (reference 06 built twice)
- [ ] Trivy and the size budget run on the **local** image; nothing vulnerable reaches ECR (references 01/05 pushed first)
- [ ] Tags: `<git-sha>` (and `v<semver>` on releases). **No `latest`, ever** (references 04/06 pushed `latest`)
- [ ] The **digest** (`…@sha256:…`) is the deployable identity from here on
- [ ] **[OPT]** SLSA build provenance with `actions/attest-build-provenance`
- [ ] **Prove it:** `cosign verify $IMAGE --certificate-identity-regexp 'https://github.com/<org>/sockshop/.github/workflows/release.yml@.*' --certificate-oidc-issuer https://token.actions.githubusercontent.com` succeeds; `cosign verify-attestation --type cyclonedx …` shows the SBOM

---

## P7-4 Hand the digest to GitOps (CI never deploys)

**Type:** Story · *Reference 07 ADR-0009: CI builds, Argo CD deploys*

- [ ] Final release step commits the new **digest** to `sockshop-gitops/envs/dev/<service>.yaml` using a **GitHub App token scoped to the gitops repo only** (`actions/create-github-app-token`), not a personal token, and not `contents: write` on the app repo
- [ ] The CI workflows contain **zero** `kubectl` / `helm upgrade` calls (`grep` it)
- [ ] Alternative: Argo CD Image Updater watches ECR and writes back to Git (ADR if chosen)

---

## P7-5 Promotion by digest, never by tag

**Type:** Story · *Reference 04's idea, fixed*

Reference 04's prod workflow pulled **`:latest`** and re-tagged it, so if a
newer build landed after QA sign-off, an **untested image** went to production.

- [ ] Promotion = a PR to the gitops repo that **copies the exact digest** from `envs/dev` → `envs/staging` → `envs/prod`. Nothing is rebuilt or re-tagged
- [ ] `envs/prod` changes need a code-owner review **and** the GitHub `production` environment approval (P5-3)
- [ ] A small script or workflow (`promote.sh dev staging catalogue`) does the copy, so humans don't hand-edit digests
- [ ] **One `main` branch, no environment branches** in the gitops repo: environments are folders (`envs/dev`, `envs/staging`, `envs/prod`). Branches per environment drift apart and promotion turns into merge conflicts (reference 09)
- [ ] **Lower environments merge themselves; prod needs a human** (reference 09's ruleset trick):
  - a GitHub **ruleset** on `main`: pull request required, **0** required approvals by default, **"require review from code owners" on**, required status checks on
  - `CODEOWNERS` lists **only** `/envs/prod/` (owned by the platform leads)
  - result: the bot's dev/staging PRs **auto-merge** when checks pass (`gh pr merge --auto`); any PR touching `envs/prod/` waits for a lead
- [ ] So every change is: one feature PR in the app repo → a bot PR to staging (auto-merged) → a promotion PR to prod (approved)
- [ ] **Staging runs integration/API tests (P8-4c) and the ZAP scan (P10-6)** before anyone opens the prod promotion PR. Neither exists yet: build the promotion flow now and add them as required checks when you reach them
- [ ] Say "promotion" for moving between environments **through Git**, and "promote" for the Argo Rollouts command that advances a canary step. Interviewers notice when these get mixed up
- [ ] **Prove it now:** `promote.sh` opens a PR that changes **only** the digest lines, and the digest CI wrote to `envs/dev` is byte-for-byte the one in the promotion PR
- [ ] ⏸ **Prove it after P8 and P10-6:** the digest running in prod equals the digest that passed staging's tests, ZAP scan and CUJ (`kubectl get pod -o jsonpath='{..image}'`)

---

## P7-6 Infrastructure pipeline

**Type:** Story

- [ ] On PR to `platform/`: `terraform fmt -check`, `validate`, `tflint`, `checkov`, then `terraform plan` (read-only OIDC role) posted as a PR comment
- [ ] On merge: `terraform apply` for dev automatically; prod behind the `production` environment approval
- [ ] Nightly `terraform plan -detailed-exitcode` → alert on drift (P4-10)

---

## P7-7 Findings management

**Type:** Task

- [ ] Thresholds documented per gate (e.g. block HIGH/CRITICAL **with a fix available**)
- [ ] Suppressions are files in the repo with a reason and an **expiry date** (`.trivyignore`, `#checkov:skip=CKV_X:reason`, `.gitleaksignore`), reviewed in PRs
- [ ] SARIF uploaded to GitHub code scanning so findings are visible in one place
- [ ] Dependabot/Renovate for dependencies, base images and action SHAs

---

## Prove it

- [ ] Five deliberately bad PRs (secret, bad Dockerfile, insecure Terraform, vulnerable dependency, SQL injection) are all **blocked**
- [ ] A deliberately vulnerable base image fails at the image-scan gate and **never appears in ECR**
- [ ] Every running image is signed by your workflow, has an SBOM attestation, and its digest matches the gitops repo

## Common mistakes

| Mistake | Seen in |
|---|---|
| Every gate disabled (`--exit-code 0`, `|| true`, `continue-on-error`) | 01, 04 |
| Push first, scan after | 01, 05 |
| Building the image twice for two tags | 06 |
| Pushing `latest` / promoting `latest` | 04, 06 |
| CI with `kubectl apply` and cluster credentials | 02, 04 |
| `pull_request` workflow that pushes or force-pushes to `main` | 05 |
| Actions pinned to tags or `@master` | 04 (`sonarqube-scan-action@master`), most others |
| Only one of N services has CI | 05, 06 |
| Checkov told to scan Terraform that isn't in the repo | 04 |
