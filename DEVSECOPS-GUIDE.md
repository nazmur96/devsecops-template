# DevSecOps Field Guide

A guide to follow on every project, a study plan, and a script for explaining
the setup in an interview. Each control is listed with **what** it is, **why**
it exists, and **how** to implement it.

**Contents**

1. [The mental model](#1-the-mental-model)
2. [Layer 1: Baseline (secure by design)](#2-layer-1-baseline-secure-by-design)
3. [Layer 2: Pipeline checks](#3-layer-2-pipeline-checks)
4. [Layer 3: Enforcement](#4-layer-3-enforcement)
5. [Deep dives](#5-deep-dives)
6. [Project checklist (follow this every time)](#6-project-checklist-follow-this-every-time)
7. [Learning roadmap](#7-learning-roadmap)
8. [Interview playbook](#8-interview-playbook)
9. [Lessons from the reference videos](#9-lessons-from-the-reference-videos)
10. [Cheat sheets](#10-cheat-sheets)

---

## 1. The mental model

### Scanners don't make anything secure

Every scanner (Gitleaks, Trivy, Checkov, ZAP) **reports** problems. None of
them **fixes** anything. Security comes from how the system is built, and
scanners confirm that it stayed that way.

> The building inspector doesn't build the house. You need a builder (the
> baseline) and an inspector (the pipeline). You also need a locked gate so
> nobody moves in without the inspector's sign-off (enforcement).

### Three layers

| Layer | Question it answers | What happens without it |
|---|---|---|
| **1. Baseline**: secure by design | Is the system built securely? | Scanners produce endless failures; nothing is safe by default |
| **2. Pipeline**: verify every change | Did this change introduce a problem? | Hardening quietly erodes: someone deletes `USER` "to debug" and nobody notices |
| **3. Enforcement**: gatekeep | Can anything get around layers 1 and 2? | One `kubectl apply` from a laptop bypasses everything |

```
 LAYER 1 ─ baseline built into the infra, images, manifests and identities
 ──────────────────────────────────────────────────────────────────────────────
 LAYER 2 ─ the pipeline
 Commit/PR ─> Build ─> Scan ─> Sign ─> Registry ─> Staging ─> Production
   │           │        │       │         │           │           │
 Gitleaks    SAST     Syft    Cosign   scheduled    ZAP       Falco /
 hadolint    (Semgrep/ Trivy/  (+SBOM   rescans     (DAST)    Tracee
 Checkov     Sonar)   Grype    attest)                        (runtime)
 SCA
 ──────────────────────────────────────────────────────────────────────────────
 LAYER 3 ─ branch protection · admission control (Kyverno) · GitOps (Argo CD)
```

### One rule, traced through all three layers

Take **"containers must not run as root"**:

1. **Baseline:** the Dockerfile sets `USER 10001`, and the manifest sets `runAsNonRoot: true`.
2. **Pipeline:** hadolint and Checkov fail the PR if either is missing.
3. **Enforcement:** Kyverno or Pod Security Admission rejects a root pod, even one applied by hand.

If you can explain this in an interview, you've shown you understand
defence in depth.

### Principles that come up again and again

- **Shift left:** catch problems where they're cheapest to fix (in the editor, then the PR).
- **Defence in depth:** assume every single control will eventually fail.
- **Least privilege:** every identity (human, CI, pod) gets only what it needs.
- **Immutability:** build once, deploy the same bytes everywhere, identified by digest.
- **Everything as code:** if it isn't in git, it doesn't exist. Manual changes are drift.
- **Fail closed:** a security gate that only reports is decoration.

---

## 2. Layer 1: Baseline (secure by design)

These belong in **every** production system, whether or not you run any scanners.

### 2.1 Network

| Control | Why |
|---|---|
| Worker nodes in **private subnets**; only load balancers are public | Nodes can't be reached from the internet at all |
| Kubernetes **API endpoint private**, or restricted to known CIDRs | The API is the cluster's front door |
| Expose apps through **Ingress / ALB with TLS** (cert from ACM or cert-manager), never `NodePort` on public nodes | One controlled, encrypted, logged entry point |
| **NetworkPolicies, deny by default**, then allow frontend → backend → DB | A compromised frontend can't reach the database |
| A **CNI that enforces NetworkPolicy** (EKS VPC CNI with network policy enabled, Calico, or Cilium) | ⚠️ Without one, NetworkPolicy objects are accepted and **silently ignored**. Classic interview gotcha |
| Egress control (NAT plus egress policies, or an egress proxy) | Stops a compromised pod from sending data out or downloading tools |

### 2.2 Container images

| Control | Why |
|---|---|
| **Multi-stage build**: compile in a full image, ship only the output | No compilers, package managers or shells for an attacker to use |
| **Minimal base** (distroless, `-alpine`, `-slim`, Chainguard) | Fewer packages means fewer CVEs and a smaller attack surface |
| **Non-root `USER`** | A container breakout starts unprivileged |
| **Pin versions**: base image, and OS packages (`apk add tini=0.19.0-r3`) | Same input, same output. Unpinned packages make builds non-reproducible |
| `.dockerignore` excludes `.git`, `.env`, keys | Secrets never get copied into a layer |
| Never put secrets in `ENV`/`ARG`; use BuildKit `--mount=type=secret` | Layers are permanent; anyone with the image can read `ENV` |

### 2.3 Pod security context

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities: { drop: ["ALL"] }
  seccompProfile: { type: RuntimeDefault }
```

Also set **resource requests and limits**, otherwise one pod can starve the
node (a denial-of-service risk), and `automountServiceAccountToken: false`
unless the pod actually talks to the Kubernetes API.

### 2.4 Secrets

**Rule: never in git, never in plain YAML, never in an image.** For the deep
dive, see [5.1](#51-secrets-management-vault-vs-aws-secrets-manager-vs-sops).

| Option | Best for |
|---|---|
| **AWS Secrets Manager** | AWS-native teams; built-in rotation |
| **HashiCorp Vault** | Multi-cloud or on-prem; **dynamic secrets** (short-lived DB credentials) |
| **SOPS + age/KMS** | GitOps: the secret lives *encrypted* in git |
| Plain Kubernetes `Secret` | ❌ Only base64-encoded, not encrypted. Acceptable as the *delivery mechanism* fed by one of the options above |

### 2.5 Identity (least privilege)

| Who | How |
|---|---|
| **CI → cloud** | **OIDC federation** (GitHub Actions → `AssumeRoleWithWebIdentity`). No long-lived access keys stored as CI secrets |
| **Humans/CI → EKS** | **EKS Access Entries**, not the legacy `aws-auth` ConfigMap. See [5.2](#52-eks-authentication-aws-auth-vs-access-entries) |
| **Pod → AWS** | **EKS Pod Identity** (newer, simpler) or **IRSA**. Never the node's instance role |
| **Inside Kubernetes** | Namespaced `Role` + `RoleBinding`. Use `ClusterRole` + `ClusterRoleBinding` only for truly cluster-wide needs |
| **CI → Kubernetes** | Ideally none: with GitOps, Argo CD pulls changes from git, so CI never needs cluster credentials |

> ⚠️ A `ClusterRoleBinding` is **cluster-wide**, always. For namespace-scoped
> access, bind a ClusterRole with a **RoleBinding** inside the namespace.

### 2.6 The CI system itself

The pipeline holds the keys to production, so treat it as production.

- Separate the controller from the agents/runners (Jenkins controller vs agent; ephemeral GitHub runners).
- **Pin third-party actions to a commit SHA**, not a tag. In March 2025 the `tj-actions/changed-files` action was compromised, and pipelines that referenced it by tag leaked their secrets.
- Minimal `GITHUB_TOKEN` permissions (`permissions: contents: read` by default).
- Protected `main`: changes only via PR with required checks and a review.
- Don't run untrusted fork PRs with access to secrets.

### 2.7 Host and cluster OS

- Use an immutable, minimal node OS: **Talos** (no SSH, no shell, API-only) or Bottlerocket.
- Keep nodes patched, and replace them rather than patching in place.
- Turn on audit logs: Kubernetes API audit logs and CloudTrail.
- Encrypt etcd at rest. EKS 1.28+ encrypts Kubernetes API data by default with an AWS-owned key; use your own KMS key if you need to control and audit it.

---

## 3. Layer 2: Pipeline checks

**Golden rules:**

1. Every check **fails the build** above a defined threshold.
2. **Cheapest checks first**: fail fast, so nobody waits 10 minutes for a typo.
3. **Scan before push**: a vulnerable image never reaches the registry.
4. Build **once**, then promote the same digest through every environment.

### Phase 1: Commit / PR, before any build

| Tool | Checks | How to run / gate |
|---|---|---|
| **Gitleaks** | Secrets in diffs and history | Pre-commit hook **and** in CI. Hooks are skippable with `--no-verify`; CI is not |
| **Linters**: eslint, ruff, **hadolint** | Code style; Dockerfile practice (unpinned packages, `latest`, root) | Fail on error |
| **Checkov** (or Trivy config) | Terraform, K8s manifests, Dockerfiles: public buckets, missing `securityContext`, wildcard IAM | `checkov -d . --soft-fail false` |
| **SCA on lockfiles**: `trivy fs`, `npm audit`, `pip-audit`, OWASP Dependency-Check | Known CVEs in dependencies | `trivy fs --scanners vuln,secret --severity HIGH,CRITICAL --exit-code 1 .` |

> `trivy fs` covers SCA **and** secret scanning in one pass, which is why one
> of the reference videos used it as the first stage.

### Phase 2: Build and test

| Tool | Checks | Gate |
|---|---|---|
| **SAST**: Semgrep, CodeQL, SonarQube | *Your own* code: SQL injection, XSS, unsafe deserialisation | SonarQube **quality gate**: "no new vulnerabilities, coverage ≥ 80% on new code" |
| Unit tests | Behaviour | Feeds coverage into the gate |

- **SCA** = vulnerabilities in code you *imported*. **SAST** = vulnerabilities in code you *wrote*. **Image scan** = vulnerabilities in the *OS layer* underneath. Three different things.
- SonarQube is mainly a code-quality tool with SAST rules. For Java it needs compiled bytecode, so it runs **after** compiling.

### Phase 3: Package, scan, sign, push

```
docker build ─> Syft (SBOM) ─> Trivy/Grype (gate) ─> push by digest ─> Cosign sign + attest SBOM
```

| Step | Tool | Notes |
|---|---|---|
| SBOM | **Syft** / CycloneDX | Generate from the **built image**, so OS packages are included. See [5.5](#55-sbom) |
| Image CVE scan | **Trivy** / **Grype** | `trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 <img>`. Grype can scan the SBOM: `grype sbom:sbom.json` |
| Push | registry (ECR, GHCR) | Turn on **tag immutability** in ECR; record the `sha256` digest |
| Sign + attest | **Cosign** | Signs *the digest*; attaches SBOM and SLSA provenance. See [5.4](#54-image-signing-and-provenance) |

> Signing usually happens right after the push, because Cosign stores the
> signature *in the registry* next to the image. That's still safe, as long as
> the scan ran before the push and Layer 3 rejects anything unsigned.

### Phase 4: Registry (continuous)

- **Rescan stored images on a schedule** (e.g. ECR enhanced scanning via Amazon Inspector, or a nightly Trivy job). An image that's clean today gets a critical CVE next month, and the image hasn't changed; the CVE database has.
- Feed SBOMs into **Dependency-Track** so "are we affected by CVE-X?" is a search, not a project.

### Phase 5: Staging and DAST

| Tool | Checks | Notes |
|---|---|---|
| **OWASP ZAP** | The *running* app: headers, cookies, injection, auth flaws | See the table below |

| ZAP script | Behaviour | Use |
|---|---|---|
| `zap-baseline.py` | **Passive only**: spiders and observes, sends no attacks | Every PR/deploy; safe and fast |
| `zap-full-scan.py` | **Active attacks**: injects payloads | Nightly against staging. **Never production** |
| `zap-api-scan.py` | Active scan driven by an OpenAPI/GraphQL spec | APIs |

Authenticated scans need login configuration, otherwise ZAP only tests the login page.

### Phase 6: Deploy and verify

- Deploy by **digest** via GitOps (CI commits the new digest to the manifests repo; Argo CD syncs it).
- **Health-gated rollout**: `kubectl rollout status`, or Argo CD health checks. Roll back automatically on failure.
- Progressive delivery (**Argo Rollouts**: canary or blue/green) for real traffic.
- Promotion to production: automatic only if every gate passed. Many teams add a manual approval.

### Phase 7: Runtime (production)

| Tool | What | Notes |
|---|---|---|
| **Falco** (CNCF) | eBPF syscall monitoring: a shell in a container, writes to `/etc`, unexpected outbound connections | **Detects, doesn't block.** Route alerts via Falcosidekick to Slack/SIEM. See [5.7](#57-runtime-security-falco) |
| **Tracee** (Aqua) | Same category | Alternative |
| **Tetragon** (Cilium) | eBPF, **can enforce** (kill the process) | When you need prevention |
| Observability | Logs, metrics, traces, audit logs | Without it, an incident is invisible |

### Managing findings (what makes a pipeline usable)

- **Thresholds per app**: e.g. block on CRITICAL/HIGH *with a fix available* (`--ignore-unfixed`).
- **Explicit, reviewable suppressions** with a reason and an expiry date: `.trivyignore`, `#checkov:skip=CKV_AWS_20:public website bucket`, `.gitleaksignore`.
- `continue-on-error: true` turns a gate into a report. Use it only while **onboarding** a new scanner (report-only for two weeks, then enforce).
- Track mean time to remediate; a backlog of findings nobody owns is not a security program.

---

## 4. Layer 3: Enforcement

Layer 2 checks what goes **through** the pipeline. Layer 3 makes sure nothing
goes **around** it.

| Control | Enforces |
|---|---|
| **Branch protection** | No merge to `main` without passing checks and a review |
| **Admission control**: **Kyverno** / OPA Gatekeeper / built-in ValidatingAdmissionPolicy | Rejects bad objects *at apply time*, whoever applies them. See [5.6](#56-admission-control-kyverno) |
| **Pod Security Admission** (built in) | Namespace label `pod-security.kubernetes.io/enforce: restricted` blocks privileged pods, host mounts and root |
| **GitOps** (Argo CD / Flux) | The cluster matches git; manual edits are **reverted** (self-heal). Humans get read-only cluster access |
| **Policy as code** | The policies above live in git, reviewed like any other code |

Typical admission policies to start with:

1. Images must be **signed** by our CI identity (Cosign `verifyImages`).
2. Images only from **our registry**.
3. **No `:latest`**; ideally require a digest.
4. `runAsNonRoot: true`, no privileged containers, no `hostPath`.
5. CPU/memory **limits required**.
6. Every namespace must have a **NetworkPolicy** (Kyverno can *generate* a default-deny one automatically).

---

## 5. Deep dives

### 5.1 Secrets management: Vault vs AWS Secrets Manager vs SOPS

**The problem:** apps need credentials, and every copy of a credential is a
place it can leak (git, CI logs, image layers, `kubectl describe`).

| | AWS Secrets Manager | HashiCorp Vault | SOPS + age/KMS |
|---|---|---|---|
| Where the secret lives | AWS-managed service | Your Vault cluster (or HCP) | **Encrypted in git** |
| Rotation | Built in (Lambda rotation; native for RDS) | **Dynamic secrets**: a new DB user per request, auto-expiring | Manual (edit, re-encrypt, commit) |
| Audit | CloudTrail | Vault audit log | git history |
| Operational cost | Low | **High** (HA, unsealing, upgrades) | Very low |
| Best when | You're on AWS | Multi-cloud, on-prem, or you need dynamic credentials | GitOps homelab or small team |

**Getting it into a pod.** Never bake it into the image:

- **External Secrets Operator (ESO)**: syncs from SM/Vault into a Kubernetes `Secret`. Most common.
- **Secrets Store CSI Driver**: mounts the secret as a file and can skip the `Secret` object entirely.
- **Vault Agent Injector**: a sidecar writes the secret to a shared volume.
- **SOPS**: Argo CD (via KSOPS or helm-secrets) decrypts at sync time.

**Interview line:** *"Kubernetes Secrets are base64, not encryption. I keep
the source of truth in Secrets Manager, sync it with External Secrets Operator,
enable envelope encryption on etcd, and lock down RBAC so only the app's
ServiceAccount can read that Secret. For databases I'd prefer Vault dynamic
credentials, so a leaked password expires on its own."*

**If a secret leaks:** **rotate it first**, then clean up history. Removing it
from git doesn't un-leak it; forks, clones and caches already have it.

### 5.2 EKS authentication: `aws-auth` vs Access Entries

**How EKS auth works:** you authenticate with **IAM**, and Kubernetes then has
to map that IAM identity to a Kubernetes user and groups for **RBAC**
authorization. These are two separate steps, which is why you can be
*authenticated* and still get "Unauthorized".

**The old way, the `aws-auth` ConfigMap** (in `kube-system`):

```yaml
mapRoles: |
  - rolearn: arn:aws:iam::111122223333:role/jenkins-agent
    username: jenkins-agent
    groups: [jenkins-deployers]
```

Why it was replaced:

| Problem | Effect |
|---|---|
| A YAML blob **inside** the cluster | One bad indent and everyone is locked out, including you |
| Needs cluster access to edit it | Chicken-and-egg: to grant access, you already need access |
| **Invisible creator admin** | The IAM identity that created the cluster was permanently `system:masters`, not shown anywhere, impossible to remove |
| Awkward for IaC | Terraform had to patch a ConfigMap inside the cluster it had just created |
| Weak audit | Changes are a kubectl edit, not a CloudTrail API event |

**The new way, EKS Access Entries** (since Dec 2023):

- Managed through the **EKS API** (`aws eks create-access-entry`, `associate-access-policy`), so it's IAM-native, auditable in CloudTrail, and a first-class Terraform resource (`aws_eks_access_entry`, `aws_eks_access_policy_association`).
- Grant **AWS-managed access policies** (`AmazonEKSClusterAdminPolicy`, `AmazonEKSAdminPolicy`, `AmazonEKSEditPolicy`, `AmazonEKSViewPolicy`), scoped to the **whole cluster or specific namespaces**, or map the principal to Kubernetes **groups** and use your own RBAC.
- Cluster-creator admin is an **explicit, removable** entry (`bootstrapClusterCreatorAdminPermissions`).
- Works without any cluster access, so no lock-out.
- Authentication modes: `CONFIG_MAP` → `API_AND_CONFIG_MAP` (migrate) → `API`. ⚠️ Switching is **one-way**.

```bash
aws eks create-access-entry --cluster-name prod \
  --principal-arn arn:aws:iam::111122223333:role/ci-deployer
aws eks associate-access-policy --cluster-name prod \
  --principal-arn arn:aws:iam::111122223333:role/ci-deployer \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy \
  --access-scope type=namespace,namespaces=app
```

**Don't confuse** these with how **pods** get AWS permissions (IRSA / Pod
Identity). Access Entries are about *who can call the Kubernetes API*.

**Interview line:** *"aws-auth was a ConfigMap inside the cluster, so a typo
could lock everyone out, and the cluster creator had invisible permanent admin.
Access Entries move the mapping to the EKS API, so it's auditable in
CloudTrail, managed in Terraform, scoped by namespace, and the creator's admin
is explicit and removable."*

### 5.3 Tags vs digests

- A **tag** (`app:1.4`, `app:latest`) is a **mutable pointer**. Anyone with push access can move it to different bytes.
- A **digest** (`app@sha256:9f86d0…`) is the **hash of the content**. Different content means a different digest. It cannot change.

Why it matters:

1. **Integrity:** what you scanned and signed is exactly what runs. With tags, you might scan image A and deploy image B under the same name.
2. **Reproducibility:** rollback goes to exactly the previous bytes.
3. **Signing:** Cosign signs the digest, so a signature on a tag means nothing.

How: enable **ECR tag immutability** (it stops tag overwrites but is still a
weaker guarantee than a digest), have CI write the digest into the manifest
(`kustomize edit set image app=repo@sha256:…`), and have Kyverno require
digests.

> Correction from one of the videos: tags are **not** immutable. That's the
> whole reason to use digests.

### 5.4 Image signing and provenance

**The problem:** the registry is a target. If someone pushes a malicious image
(stolen CI token, compromised registry), how does the cluster know it isn't
yours?

**Cosign (Sigstore):**

- **Key-based:** sign with a private key, ideally stored in AWS KMS (`cosign sign --key awskms:///alias/cosign <img>@sha256:…`).
- **Keyless:** CI proves its identity via OIDC. **Fulcio** issues a short-lived certificate saying "this was GitHub Actions, repo X, workflow Y", and the signature is recorded in the **Rekor** public transparency log. There are no keys to leak.
- **Attestations:** signed statements *about* the image. `cosign attest --type cyclonedx --predicate sbom.json` attaches the SBOM. **SLSA provenance** records *how* it was built (repo, commit, builder).
- **Verification** happens at admission (Kyverno `verifyImages`): "only run images signed by `github.com/me/repo/.github/workflows/release.yml`".

Alternatives: **Notation** (Notary v2) and **AWS Signer**, which integrate with ECR.

**Interview line:** *"Scanning proves an image was clean when built. Signing
proves it's the image we built. Kyverno verifies the signature at admission,
so a pushed-but-unsigned image never runs, even with stolen registry
credentials."*

### 5.5 SBOM

- A **Software Bill of Materials** lists every package in the artifact, with versions and licenses. Formats: **CycloneDX** and **SPDX**.
- **Why:** when Log4Shell hit (Dec 2021), teams without SBOMs spent days grepping; teams with SBOMs answered "are we affected?" in minutes. The same applies to the xz-utils backdoor (2024).
- **How:** Syft on the built image → attach it as a Cosign attestation → send it to Dependency-Track for continuous matching against new CVEs.
- Increasingly required by regulators and customers (US EO 14028, EU Cyber Resilience Act).

### 5.6 Admission control: Kyverno

An **admission controller** intercepts every create/update request to the
Kubernetes API and can **validate** (reject), **mutate** (fix) or **generate**
(create companion objects). It's the last gate before anything runs.

| | Kyverno | OPA Gatekeeper | ValidatingAdmissionPolicy |
|---|---|---|---|
| Policy language | YAML | Rego | CEL |
| Validate | ✅ | ✅ | ✅ |
| Mutate / Generate | ✅ / ✅ | mutate only | ❌ |
| Verify image signatures | ✅ built in | needs extras | ❌ |
| Install | Helm chart | Helm chart | **Built into K8s** (GA 1.30) |

Example: require signed images from our CI.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: verify-signed-images }
spec:
  validationFailureAction: Enforce
  webhookTimeoutSeconds: 30
  rules:
    - name: cosign-keyless
      match: { any: [ { resources: { kinds: [Pod] } } ] }
      verifyImages:
        - imageReferences: ["ghcr.io/nazmur96/*"]
          mutateDigest: true          # rewrite tag -> digest
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/nazmur96/*/.github/workflows/*"
                    issuer: "https://token.actions.githubusercontent.com"
```

**Rollout tip:** start every policy in **Audit** mode, read the policy
reports, fix the violations, *then* switch to **Enforce**. Going straight to
Enforce breaks the cluster on day one.

### 5.7 Runtime security: Falco

- Hooks into the kernel via **eBPF** and watches **syscalls** from every container.
- Default rules catch, among others: a shell spawned in a container, reading `/etc/shadow`, writing to binary directories, unexpected outbound connections, `kubectl exec` into pods.
- **Detection only.** It alerts but doesn't stop anything. Pair it with response: Falcosidekick → Slack, PagerDuty, SIEM, or Falco Talon to auto-isolate the pod. **Tetragon** can enforce directly.
- Why it's needed even with everything else: zero-days and stolen credentials don't show up in any scan. Runtime is where you see the attack *happening*.
- Tune it: default rules are noisy. Too many alerts and people stop reading them.

### 5.8 Pipeline ordering mistakes to avoid

| Mistake | Why it's wrong | Fix |
|---|---|---|
| Push, *then* scan | A vulnerable image is already in the registry and deployable | Scan the local image, push only on pass |
| `continue-on-error: true` on gates | Every job "passes", and deploy still happens | Remove it after onboarding |
| SBOM from source only | Misses OS packages in the base image | Generate from the built image |
| Pre-commit hooks as the only secret check | `--no-verify` skips them | Also run in CI |
| Scanning only once at build time | New CVEs appear daily | Scheduled registry rescans |
| ZAP full scan against production | Active attacks can corrupt data | Staging only |

---

## 6. Project checklist (follow this every time)

Work top to bottom. Each item is a commit you can show in a portfolio.

### Phase A: Understand the app
- [ ] Run it locally by following the developers' README. Note the stack, ports, env vars and dependencies.
- [ ] Draw the architecture (frontend → backend → DB) and the trust boundaries.
- [ ] Threat-model it in one page: what's valuable, who might attack, which entry points exist (STRIDE helps).

### Phase B: Containerise securely (Layer 1)
- [ ] Multi-stage Dockerfile, minimal base, pinned versions, non-root `USER`, `.dockerignore`.
- [ ] Docker Compose for local runs; secrets via `.env` (git-ignored) or Compose secrets.
- [ ] `hadolint` and `trivy image` locally. Get them clean before writing CI.

### Phase C: Infrastructure as code (Layer 1)
- [ ] Terraform: VPC, **private subnets**, NAT, private (or CIDR-restricted) cluster endpoint.
- [ ] EKS with **Access Entries** (`authentication_mode = "API"`), managed node groups in private subnets across 2+ AZs.
- [ ] Remote state with locking, encrypted.
- [ ] `checkov -d terraform/` clean (or documented skips).

### Phase D: Kubernetes manifests (Layer 1)
- [ ] Namespace with Pod Security label `restricted`.
- [ ] Deployment with the full `securityContext`, requests/limits, probes, and pinned image **digest**.
- [ ] Dedicated ServiceAccount; token not mounted unless needed.
- [ ] NetworkPolicies: default-deny, then the explicit allows. **Verify** the CNI enforces them (try a blocked connection and confirm it fails).
- [ ] Ingress + TLS; no NodePort/LoadBalancer per service.
- [ ] Secrets via ESO from Secrets Manager/Vault, or SOPS.
- [ ] `checkov -d k8s/` clean.

### Phase E: CI pipeline (Layer 2)
- [ ] OIDC to the cloud, no stored keys. Actions pinned to SHAs. Minimal token permissions.
- [ ] Stages: Gitleaks → lint/hadolint → Checkov → SCA → build/test → SAST + quality gate → build image → SBOM → Trivy gate → push → Cosign sign + attest.
- [ ] Every gate **blocks**. Suppressions are explicit and justified.
- [ ] CI updates the manifest with the new **digest** (a commit to the GitOps repo).

### Phase F: Deploy and enforce (Layer 3)
- [ ] Argo CD with auto-sync + self-heal; humans read-only in the cluster.
- [ ] Kyverno: signed images, registry allow-list, no root, limits required, digest required. Audit first, then Enforce.
- [ ] Branch protection on `main`.
- [ ] Prove it: try `kubectl apply` with an unsigned image, show it's rejected, screenshot it.

### Phase G: Staging, DAST, runtime
- [ ] Staging environment; ZAP baseline on every deploy; full scan nightly.
- [ ] Falco with alerts to Slack; trigger `kubectl exec ... sh` and show the alert.
- [ ] Scheduled registry rescans.
- [ ] Basic observability (Prometheus + Grafana, or CloudWatch).

### Phase H: Document it
- [ ] README with an architecture diagram, the three-layer table, and the **trade-offs you made** ("NodePort to save cost in the demo; in production I'd use ALB + ACM").
- [ ] Screenshots: failing gate, rejected admission, Falco alert. **Evidence beats claims.**

---

## 7. Learning roadmap

Learn in this order; each step depends on the one before.

| # | Topic | Practise by |
|---|---|---|
| 1 | Linux, networking, git | Daily use |
| 2 | **Docker**: layers, multi-stage, users | Harden one Dockerfile until `trivy image` and hadolint are clean |
| 3 | **CI** (GitHub Actions) | Lint + test pipeline; then add Gitleaks and Trivy as blocking gates |
| 4 | **Kubernetes** basics, then **RBAC, NetworkPolicy, securityContext, PSA** | Local cluster (kind/k3d, or Talos on Proxmox) |
| 5 | **Terraform** + Checkov | VPC + cluster, Checkov-clean |
| 6 | **Cloud IAM**: OIDC, roles, EKS Access Entries, Pod Identity | GitHub Actions → AWS with zero stored keys |
| 7 | **Secrets**: SOPS → ESO → Vault | Same app, three secret backends |
| 8 | **GitOps** (Argo CD) | CI writes the digest, Argo deploys it |
| 9 | **Supply chain**: SBOM, Cosign, SLSA | Sign in CI, verify locally with `cosign verify` |
| 10 | **Admission** (Kyverno) | Audit → Enforce; prove an unsigned image is rejected |
| 11 | **DAST + runtime** (ZAP, Falco) | Trigger and capture alerts |
| 12 | Threat modelling, compliance (CIS Benchmarks, SOC 2 basics) | kube-bench, Prowler |

**On your own stack:** Proxmox → Talos (immutable, no SSH; a Layer 1 point you
can talk about) → Argo CD (Layer 3) → SOPS + age (secrets) → Kyverno + Falco.
That's a full three-layer homelab, then repeat it on EKS for the cloud version.

---

## 8. Interview playbook

### 8.1 The 60-second pitch: "Walk me through your DevSecOps setup"

> "I think of it in three layers. **First, the baseline**: the system is
> secure by design, with private subnets and a private API endpoint, minimal
> non-root images, deny-by-default NetworkPolicies, secrets in Secrets Manager
> synced by External Secrets, and least-privilege identity: OIDC from CI with
> no stored keys, EKS Access Entries, Pod Identity for workloads.
>
> **Second, the pipeline** verifies every change, cheapest checks first:
> Gitleaks, hadolint and Checkov on the PR, SCA on lockfiles, SAST with a
> quality gate, then build, generate an SBOM with Syft, and gate on Trivy
> before pushing. Every gate blocks. We push by digest and sign with Cosign
> keyless, attaching the SBOM as an attestation.
>
> **Third, enforcement**, so nothing bypasses the pipeline: Argo CD deploys
> from git with self-heal, and Kyverno only admits images signed by our CI
> workflow, by digest, running as non-root. Staging gets a ZAP scan, and in
> production Falco watches syscalls for anything the scanners couldn't
> predict."

Then stop and let them choose what to dig into. Each sentence above is a door
to a deep dive you've prepared in [section 5](#5-deep-dives).

### 8.2 How to answer like a senior

1. **Lead with the why**, then the tool. "To stop a pushed-but-unauthorised image from running, we verify signatures at admission, using Kyverno."
2. **Name the trade-off.** "Vault gives dynamic credentials but it's heavy to operate; for a small AWS team, Secrets Manager plus rotation is the right call."
3. **Say what a control does *not* do.** "Falco detects, it doesn't prevent." "Pre-commit hooks can be skipped." "NetworkPolicy needs a CNI that enforces it." This signals real experience.
4. **Give a failure story.** "The pipeline flagged an unpinned `apk add`, and here's why that mattered…"
5. **Use the right words**: shift left, defence in depth, least privilege, blast radius, immutable artifacts, fail closed, drift.

### 8.3 Likely questions, short answers

| Question | Answer in one breath |
|---|---|
| SAST vs DAST vs SCA? | SAST reads your source code; SCA checks the libraries you imported; DAST attacks the running app from outside. They find different bugs, so you need all three |
| Why scan the image if you already ran SCA? | SCA covers app dependencies; the image scan also covers the **base OS packages** (openssl, glibc), which SCA never sees |
| Tag vs digest? | Tags are mutable pointers; a digest is a content hash. Sign and deploy by digest so what you scanned is what runs |
| Why sign images if you scan them? | A scan says "clean when built"; a signature says "built by us". It protects against registry tampering and stolen push credentials |
| Are Kubernetes Secrets secure? | They're base64, not encrypted. Enable etcd encryption, restrict RBAC, and keep the source of truth in SM/Vault via ESO |
| aws-auth vs Access Entries? | See [5.2](#52-eks-authentication-aws-auth-vs-access-entries): in-cluster ConfigMap with lock-out risk and invisible creator admin, vs an IAM-native API that's auditable, scoped and Terraform-friendly |
| IRSA vs Pod Identity vs Access Entries? | Access Entries: who can call the **Kubernetes API**. IRSA/Pod Identity: how a **pod** gets **AWS** permissions. Pod Identity is the newer, simpler option |
| Kyverno vs Gatekeeper? | Both are admission controllers. Kyverno is YAML, can mutate/generate, and has built-in signature verification; Gatekeeper uses Rego and is more expressive but harder to learn |
| How do you roll out a new policy safely? | Audit mode, read the reports, fix violations, then Enforce, one namespace at a time |
| A developer committed an AWS key. What do you do? | **Rotate the key immediately**, check CloudTrail for misuse, then purge history. Rotation first, because the leak already happened |
| Critical CVE in a base image, no fix available? | Assess exploitability (is the vulnerable code even reachable?), document a time-boxed exception, add compensating controls, track the upstream fix. Don't block every deploy forever |
| What does Falco catch that scanners miss? | Behaviour at runtime: zero-days, stolen-credential abuse, someone getting a shell in a pod |
| How do you secure the pipeline itself? | OIDC instead of stored keys, actions pinned to SHAs (tj-actions, March 2025), minimal token permissions, ephemeral runners, protected branches |
| Pushing to registry before scanning: problem? | Yes. The image is deployable before it's vetted. Scan locally, push on pass, and have admission reject anything unsigned |
| Your pipeline is too slow and devs bypass it. What now? | Cheapest checks first, run in parallel, cache scanner DBs, scan only changed paths on PRs, run full/active scans nightly |

### 8.4 Turn demo shortcuts into talking points

Interviewers *like* hearing that you know where your demo cuts corners:

> "In my demo the app is exposed via NodePort to save cost. In production I'd
> use private nodes behind an ALB with an ACM certificate and WAF, because
> NodePort on public nodes exposes every node's IP and skips TLS."

---

## 9. Lessons from the reference videos

| | **Video A**: Abhishek Veeramalla, *Day-7 End-to-End DevSecOps* (GitHub Actions + Argo CD, Node/React/Postgres) | **Video B**: Cloud With VarJosh, *DevSecOps Mega Project* (Jenkins + SonarQube + Trivy + ECR + EKS, Java) |
|---|---|---|
| **Layer 1** | ✅ Private VPC/subnets, private K8s API, multi-stage builds, non-root, NetworkPolicies. ⚠️ Plain K8s Secrets | ✅ Jenkins controller/agent split, IAM least privilege, Jenkins credential bindings. ⚠️ **Public** worker nodes, NodePort, SSH allowed (he admits this) |
| **Layer 2** | Lint → `npm audit` → build + push → Trivy → Checkov → hadolint → update manifest | `trivy fs` (SCA + secrets) → Maven build → SonarQube + **quality gate** → Trivy image → push to ECR → deploy → rollout check + auto-rollback |
| **Layer 3** | Argo CD (GitOps) | None; lists Kyverno, signing, digests, Vault and Argo CD as future work |
| **Does well** | Hardening at every layer; GitOps | Gates **actually block**; SAST with quality gate; a clear explanation that IAM authentication and RBAC authorization are separate steps |
| **Don't copy** | Pushes the image **before** scanning; `continue-on-error: true` on every job, so failing scans still deployed | `aws-auth` instead of **Access Entries**; `ClusterRoleBinding` described as namespace-wide (it's **cluster-wide**); "tags are immutable" (they're **mutable**); CI pushes to the cluster directly (GitOps removes that need) |
| **Missing** | Gitleaks, SAST, SBOM, signing, DAST, runtime | Checkov, SBOM, signing, DAST, runtime, admission |

**Takeaway:** neither video is the whole picture. Video A is strongest on
Layer 1, Video B on Layer 2 gates, and neither does Layer 3 properly. This
guide combines both and adds the rest.

---

## 10. Cheat sheets

### Tool → layer → stage

| Category | Tools | Layer · stage |
|---|---|---|
| Secret scanning | Gitleaks, TruffleHog, `trivy fs` | 2 · commit + CI |
| Dockerfile lint | hadolint | 2 · commit |
| IaC misconfig | Checkov, Trivy config, (tfsec, now part of Trivy) | 2 · commit |
| SCA | `trivy fs`, Grype, npm audit, pip-audit, OWASP Dependency-Check, Dependabot/Renovate (auto-update PRs) | 2 · commit |
| SAST | Semgrep, CodeQL, SonarQube | 2 · build |
| SBOM | Syft, CycloneDX; Dependency-Track to consume | 2 · post-build |
| Image scan | Trivy, Grype, ECR/Inspector | 2 · pre-push + scheduled |
| Signing / provenance | Cosign (Sigstore: Fulcio, Rekor), SLSA, Notation, AWS Signer | 2 sign → 3 verify |
| DAST | OWASP ZAP (baseline / full / api), Nuclei | 2 · staging |
| Admission | Kyverno, OPA Gatekeeper, ValidatingAdmissionPolicy, Pod Security Admission | 3 · deploy |
| GitOps | Argo CD, Flux | 3 · deploy |
| Progressive delivery | Argo Rollouts, Flagger | 2 · deploy |
| Secrets | AWS Secrets Manager, Vault, SOPS + age; ESO / CSI driver to deliver | 1 |
| Cloud identity | OIDC federation, EKS Access Entries, Pod Identity / IRSA | 1 |
| Runtime | Falco, Tracee, Tetragon | 2 · production |
| Compliance / posture | kube-bench (CIS), Prowler, Kubescape | 1 audit |

### Commands worth memorising

```bash
gitleaks git -v                                         # scan repo history
hadolint Dockerfile
checkov -d . --framework terraform,kubernetes
trivy fs --scanners vuln,secret --severity HIGH,CRITICAL --exit-code 1 .
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 app:dev
syft app:dev -o cyclonedx-json > sbom.json
grype sbom:sbom.json --fail-on high
cosign sign  ghcr.io/me/app@sha256:<digest>             # keyless in CI
cosign attest --type cyclonedx --predicate sbom.json ghcr.io/me/app@sha256:<digest>
cosign verify ghcr.io/me/app@sha256:<digest> \
  --certificate-identity-regexp 'https://github.com/me/.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
docker run -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t https://staging.example.com
aws eks list-access-entries --cluster-name prod
kubectl auth can-i create deployments -n app --as=system:serviceaccount:app:ci
```

### Glossary

| Term | Meaning |
|---|---|
| **Shift left** | Move checks earlier, where fixes are cheaper |
| **Blast radius** | How much damage one compromised component can do |
| **Drift** | Live state differing from what's declared in git |
| **Fail closed** | When a check fails or errors, block rather than allow |
| **Attestation** | A signed statement about an artifact (SBOM, provenance, scan result) |
| **Provenance** | Verifiable record of how and where an artifact was built (SLSA) |
| **CVE / CVSS** | A public vulnerability ID / its severity score (0–10) |
| **Admission controller** | Kubernetes API plugin that validates/mutates requests before they're stored |
| **IRSA / Pod Identity** | Ways for a pod to get scoped AWS credentials |
| **Access Entry** | IAM-native mapping of an AWS principal to EKS cluster permissions |
