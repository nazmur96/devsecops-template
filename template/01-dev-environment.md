# Phase 1: Workstation & dev environment

**Goal:** anyone (including future you) can clone the repo, run one command,
and get the exact same toolchain; you can run the application locally and
explain how a request flows through it.

**Why:** "works on my machine" starts here. Tools installed by hand on the
host drift, two installs of the same runtime silently fight over `PATH`, and
nobody can reproduce your setup. Reference 04 spends a whole video installing
tools into `C:\DevOps-tools` by hand; this phase makes that a single file.

**Rules** (from `~/orca/projects/gitops-workstation/TOOLS.md`):

1. **Nothing project-specific on the host.** No terraform, kubectl, helm, language runtimes or language servers in the OS config.
2. A repo has **`mise.toml` or `devenv.nix`, never both.**
3. A runtime is owned by the **base image or by mise, never both.**
4. Language servers live in the **dev container** (editor server), not on the host (editor client).
5. **If it isn't in a committed file, it doesn't exist.**

---

## P1-1 Decide the repository layout

**Type:** Decision (ADR)

| Repo | Contains | Who changes it |
|---|---|---|
| `<app>` | Service source, Dockerfiles, CI workflows, `mise.toml`, `devcontainer.json` | Developers + CI |
| `<app>-platform` | Terraform (state backend, VPC, EKS, IAM, ECR, data stores) | Platform engineers |
| `<app>-gitops` | Helm values / manifests per environment, Argo CD apps | CI (image digests) + platform engineers |

- [ ] Layout chosen and recorded as an ADR. App code and platform code may share a repo while learning, but the **GitOps repo is always separate** (below)
- [ ] Each repo has `README.md`, `.gitignore`, `.editorconfig`, `CODEOWNERS`, branch protection on `main` (PR required, status checks required, no force-push)

### Why Argo CD gets its own repository

Argo CD watches a Git repo and makes the cluster match it. **Which** repo it
watches matters:

| If Argo CD watched the app repo | With a separate `<app>-gitops` repo |
|---|---|
| Every code commit, README typo or test change wakes Argo CD up | Argo CD only reacts to real deployment changes |
| CI needs write access to the **source code** repo to bump image versions, a big blast radius if CI is compromised | CI can write **only** to the gitops repo, and only image digests (P7-4) |
| "What's running in prod, and since when?" is buried in code history | The gitops repo's history **is** the deployment log: every commit = a deploy, every revert = a rollback |
| App developers and prod approvers share one permission model | Different access: developers own app code; `envs/prod/` needs platform/code-owner review |
| CI commits trigger CI again (loops, `[skip ci]` hacks: reference 04) | No loops: CI writes to a repo that doesn't run the app's CI |

Folder layout of the gitops repo:

```
<app>-gitops/
├── README.md                   what this repo is, how to promote, break-glass
├── CODEOWNERS                  /envs/prod/  @platform-team
├── bootstrap/
│   └── root-app.yaml           the one Application you apply by hand, once (app-of-apps, P8-3)
├── apps/                       Argo CD Application / ApplicationSet / AppProject objects
│   ├── platform/               controllers, Kyverno, observability, Falco
│   └── sockshop/               one Application per environment
├── charts/
│   └── sockshop/               the umbrella Helm chart (P8-1); or pulled from ECR as OCI
├── platform/                   values for platform charts, per environment
│   └── kyverno/ policies/ …
└── envs/
    ├── dev/      values.yaml   image digests written by CI (P7-4)
    ├── staging/  values.yaml   digests promoted from dev (P7-5)
    └── prod/     values.yaml   digests promoted from staging, reviewed
```

- [ ] `<app>-gitops` repo created, private, with branch protection on `main` and `CODEOWNERS` requiring review for `envs/prod/`
- [ ] The only automated writer is CI's **GitHub App** token (P7-4), allowed to change `envs/dev/` digests
- [ ] **No secrets in this repo, ever**: only references (ExternalSecret objects, role names, paths)

`.gitignore` must include, from day one:

```gitignore
# Terraform: state holds secrets in plain text (see reference 06)
*.tfstate
*.tfstate.*
.terraform/
*.tfvars.json
crash.log
# local env
.env
.env.*
!.env.example
# build output
node_modules/
target/
dist/
```

---

## P1-2 Pin the toolchain with `mise.toml`

**Type:** Story

```toml
# mise.toml: tool versions for this repo. Works on the host and inside the devcontainer.
# Example pins; check current versions with `mise latest <tool>` and pin exactly.
[tools]
terraform = "1.9.8"
kubectl   = "1.33.5"     # keep within one minor version of the cluster
helm      = "3.16.2"
kind      = "0.27.0"
awscli    = "2.22.0"
trivy     = "0.57.1"
hadolint  = "2.12.0"
gitleaks  = "8.21.2"
cosign    = "2.4.1"
syft      = "1.17.0"
"pipx:checkov" = "3.2.300"
"pipx:pre-commit" = "4.0.1"

[env]
AWS_REGION  = "eu-central-1"
AWS_PROFILE = "sockshop-dev"     # SSO profile, never static keys (P5)
KUBECONFIG  = "{{config_root}}/.kube/config"   # per-repo kubeconfig; add .kube/ to .gitignore
```

- [ ] `mise install` gives every tool at the pinned version
- [ ] `mise doctor` clean; `which terraform` points into mise, not `/usr/bin`
- [ ] Language runtimes (Go, Java, Node) are owned by **either** the devcontainer image **or** `mise.toml`. Write down which in the README
- [ ] **No** tool from this list installed system-wide on the host

> **Nix-native alternative:** use `devenv.nix` instead. Pick this if you also want local **services** (Postgres, Redis) managed by the dev env, which mise can't do. Never use both in one repo.

---

## P1-3 Define the dev container

**Type:** Story

`devcontainer.json` defines the **userland** (base OS, system libs); `mise.toml`
defines **tool versions** inside it; **DevPod** decides **where** it runs
(local Podman now, a remote host later, with one flag).

```jsonc
// .devcontainer/devcontainer.json
{
  "name": "sockshop-platform",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu-24.04",
  "features": {
    // needed for docker build / kind inside the container
    "ghcr.io/devcontainers/features/docker-in-docker:2": {}
  },
  "remoteEnv": {
    "PATH": "${containerEnv:PATH}:/home/vscode/.local/bin:/home/vscode/.local/share/mise/shims"
  },
  "postCreateCommand": "curl -fsSL https://mise.run | sh && ~/.local/bin/mise trust && ~/.local/bin/mise install && pre-commit install",
  "customizations": {
    "vscode": {
      "extensions": [
        "hashicorp.terraform",
        "redhat.vscode-yaml",
        "ms-kubernetes-tools.vscode-kubernetes-tools",
        "golang.go",
        "vscjava.vscode-java-pack",
        "dbaeumer.vscode-eslint"
      ]
    }
  }
}
```

- [ ] `devpod up . --provider docker` (Podman-backed) starts the workspace and opens the editor
- [ ] Inside it: `terraform version`, `kubectl version --client`, `kind version` all match `mise.toml`
- [ ] Destroy and recreate (`devpod delete` → `devpod up`) gives the identical environment in under 10 minutes
- [ ] Neovim users: `devpod up . --ide none` then `ssh <workspace>.devpod`

> **Known friction:** KinD needs a container engine *inside* the dev container
> (docker-in-docker, privileged). If that's unreliable on rootless Podman,
> run KinD from the host's Podman with `KIND_EXPERIMENTAL_PROVIDER=podman`,
> using the **same `mise.toml`** on the host (mise is allowed on the host;
> the tools stay per-repo). Record which you chose in an ADR.

---

## P1-4 Install the commit-time safety net

**Type:** Story · **First line of secret-leak defence**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
        args: [--allow-multiple-documents]
      - id: check-json
      - id: check-added-large-files
      - id: detect-private-key
  - repo: https://github.com/hadolint/hadolint
    rev: v2.12.0
    hooks:
      - id: hadolint            # uses the hadolint binary from mise
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.96.1
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
```

- [ ] `pre-commit install` runs automatically in `postCreateCommand`
- [ ] **Prove it:** commit a file containing a fake AWS key (`AKIA` + 16 chars). The commit is **rejected**
- [ ] Hooks run in under 5 seconds on a typical commit
- [ ] `CONTRIBUTING.md` explains the hooks **and** that CI runs the same checks (P7), because `git commit --no-verify` skips hooks

---

## P1-5 Run the application locally, exactly as the developers do

**Type:** Story

Before containerising anything, run it the developers' way. You learn the real
dependencies, env vars and failure modes (reference 01: *"try it yourself;
only then do you know the steps are correct"*).

- [ ] Each service builds and starts following its README
- [ ] Every env var it reads is listed in `docs/inventory.md` (found by grepping, not by trusting the README)
- [ ] The CUJ (P0-3) works end to end locally
- [ ] Every step that didn't match the README is fixed in the README (a PR to the dev team)

---

## P1-6 Understand the app the way an operator needs to

**Type:** Story · *Reference 04 does this best: it explains React, Node,
MySQL and the request flow in DevOps terms before any infrastructure.*

> **Read first:** [How code goes from your laptop to the user](https://github.com/nazmur96/devops-learning-journal/blob/main/foundations/how-code-goes-from-laptop-to-user.md):
> every language has an entry point, a dependency manager, a build tool and a
> final output. Knowing those four for each service is most of this ticket.

For each service, answer and record in `docs/inventory.md`:

| Question | How to find out | Why it matters later |
|---|---|---|
| How does a request flow? | Follow one CUJ step through the code | NetworkPolicy allow-list (P8), tracing (P10) |
| Health endpoint? Does it check dependencies? | `curl :port/health`; read the handler | Probe design (P6): **liveness must not check the DB** |
| Logs to stdout as structured lines? | Run it and look | Log pipeline (P10) |
| Handles SIGTERM? | `time docker stop <c>`: ~10 s means it ignored SIGTERM and got SIGKILLed | `preStop`, grace period (P6) |
| Config via env vars? | grep | 12-factor; ConfigMaps/Secrets (P5) |
| Writes to local disk? | Run with a read-only filesystem | `readOnlyRootFilesystem` + `emptyDir` (P6) |
| Startup time? | Measure | `startupProbe` (P6) |
| Memory under load? | `docker stats` during the CUJ | Requests/limits, JVM flags (P2/P6) |

---

## P1-7 [OPT] Linux and Bash fundamentals on a throwaway VM

> **Use when:** you're new to Linux administration or want Ansible practice. Skip if you're already comfortable; it isn't needed for EKS.

**Type:** Story · *Preserves the skills from the original plan's E-0
without breaking rule 1: do this on a disposable VM (Proxmox, Multipass,
cloud), never on your workstation.*

- [ ] Idempotent provisioning script (root check, OS detect, retries with backoff, logging, second run = zero changes, `shellcheck` clean)
- [ ] SSH hardening (keys only, no root login, fail2ban, firewall) with **a second session kept open** so a mistake doesn't lock you out
- [ ] Backup script (tar + SHA-256 + retention + tested restore)
- [ ] logrotate + an `awk` filter for `ERROR`/`FATAL` lines
- [ ] Then: rewrite the same thing as Ansible roles with Molecule tests, and compare

---

## Not now: the inner loop (comes in P3-9)

These tools need running workloads to iterate on, so they can't be set up
yet. They're listed here only so you know what's coming;
**skip this section and do it in [P3-9](03-local-kubernetes.md#p3-9-set-up-the-inner-loop).**

| Tool | What it does | Pick |
|---|---|---|
| **Skaffold** | Watch code → rebuild image → redeploy | Pick **one** of Skaffold or Tilt |
| **Tilt** | Same, plus live file sync into running pods and a UI | |
| **mirrord** | Run one service **locally** while it receives the cluster's traffic, env and DNS. No rebuild at all | Add alongside either; best for debugging one service |
| **vCluster** | A virtual cluster per developer inside one namespace | *Where* the loop runs, not the loop. Ticket: [P8-10](08-gitops-and-enforcement.md#p8-10-opt-vcluster-virtual-clusters-inside-the-shared-cluster) |

Ticket in [P3-9](03-local-kubernetes.md#p3-9-set-up-the-inner-loop).

---

## Prove it

- [ ] Fresh machine (or fresh DevPod workspace): `git clone && devpod up .` → every tool works → CUJ runs locally, **with no other manual step**
- [ ] A fake secret is rejected at commit time
- [ ] `docs/inventory.md` answers every question in P1-6 for every service

## Common mistakes

| Mistake | Seen in |
|---|---|
| Tools installed by hand on the host / an EC2 "workstation" with admin keys | 04 (Windows PATH), 05 (EC2 + admin IAM user) |
| Same runtime from two sources; `PATH` order decides | user's workstation notes |
| `.gitignore` missing `*.tfstate`; state committed with passwords | 05 (instructor's state in repo), 06 (state backup with DB passwords) |
| Skipping the local run, then debugging app bugs inside Kubernetes | — |
| Pre-commit as the only secret scanner | guide |
