# Phase 1 explained simply

A plain-English walk through [Phase 1: Workstation & dev environment](../template/01-dev-environment.md).
Read it before you start the phase. Then test yourself with the
[Phase 1 interview cases](p1-dev-environment.md).

Examples use the **Podinfo** run (fork
[`nazmur96/podinfo`](https://github.com/nazmur96/podinfo), a small Go app).
Phase 1 has now been run on Podinfo: lines marked **"In the Podinfo run"** are
what actually happened, with the measured numbers and the mistakes we hit.

---

## The idea in one paragraph

Phase 0 was planning on paper. Phase 1 builds the **workshop**: a safe copy
of the code, a set of tools that's the same for everyone, checks that stop
mistakes before they reach Git, and a first real run of the app, so you know
how it behaves. You're not deploying to AWS or Kubernetes yet. The test at
the end is simple: **on a fresh machine, clone the repo, run one command, and
get the exact same tools, with the app running locally and no other manual
steps.**

A beginner installs Terraform, kubectl, Helm and the AWS CLI straight onto
their laptop and starts building. An experienced engineer asks first:

- How do we get the same tool versions on every machine?
- How do we change code safely, without touching someone else's project?
- How do we stop secrets reaching Git?
- How does this app actually start, log, fail and stop?

## The whole phase on one page

```mermaid
flowchart TD
    A["1. Get the code safely<br/>clone or fork, lock the fork down<br/>(P1-0)"]
    B["2. Organise the repos<br/>app · platform · gitops<br/>(P1-1)"]
    C["3. Pin the tools<br/>mise.toml<br/>(P1-2)"]
    D["4. Make the environment reproducible<br/>devcontainer.json + DevPod<br/>(P1-3)"]
    E["5. Add the safety net<br/>pre-commit hooks, Gitleaks<br/>(P1-4)"]
    F["6. Run the app and understand it<br/>the developers' way, then as an operator<br/>(P1-5, P1-6)"]
    G["Phase 2: build the container image"]
    A --> B --> C --> D --> E --> F --> G
```

### Five rules behind the whole phase

1. **Nothing project-specific on your host machine.** No Terraform, kubectl,
   Helm, language runtimes or language servers in the operating system.
   They belong to the project's dev environment.
2. A repo has **`mise.toml` or `devenv.nix`, never both.**
3. Each language runtime (Go, Java, Node, Python) comes from **the container
   image or from mise, never both.**
4. Language servers (the editor's code intelligence) run **inside the dev
   container**; only the editor window runs on the host.
5. **If it isn't in a committed file, it doesn't exist.**

Why so strict? Tools installed by hand drift apart between machines, and two
installs of the same tool quietly fight over which one runs. Your workstation
already learned this the hard way: three copies of Claude Code, each hiding
the others.

---

## 1. Get the code the right way

*Ticket P1-0*

### Clone or fork?

- **Clone** = download a repo to your computer.
- **Fork** = make your own copy *on GitHub*, under your account.

They solve different problems. The rule: **fork only when you can't push to
the original.**

| Situation | What you do | Remotes you end up with |
|---|---|---|
| Your company's repo (you can push branches) | Clone it, branch, open a PR in the same repo | `origin` only |
| Open source or a practice app (Podinfo) | Fork it, then clone **your fork** | `origin` = your fork, `upstream` = the original |
| A company that requires forks (some banks) | Fork inside the company org, clone it, PR to the company repo | `origin` = your fork, `upstream` = company repo |

### origin and upstream

A **remote** is just a name Git uses for a repo somewhere else.

```mermaid
flowchart TD
    U["stefanprodan/podinfo<br/>the original = upstream"] -->|fork on GitHub| O["nazmur96/podinfo<br/>your fork = origin"]
    O -->|clone| L["your computer"]
    L -->|push your work| O
    U -.->|fetch updates only| L
```

The setup for a practice app:

```bash
gh repo fork stefanprodan/podinfo --clone=false   # make your copy on GitHub
gh repo clone nazmur96/podinfo                    # download YOUR copy
cd podinfo
gh repo set-default nazmur96/podinfo              # gh commands target your fork
gh repo set-default --view                        # check it: gh sometimes picks the original
git remote set-url --push upstream no-pushing     # pushing to the original now fails safely
git remote -v                                     # check what you got
```

`git remote -v` should show something like:

```
origin    git@github.com:nazmur96/podinfo.git (fetch)
origin    git@github.com:nazmur96/podinfo.git (push)
upstream  git@github.com:stefanprodan/podinfo.git (fetch)
upstream  no-pushing (push)
```

The idea: **make the safe action the easy one.** A stray `git push upstream`
or `gh pr create` aimed at the original project now fails or targets your
fork instead.

### Lock the fork down before your first push

A fork copies the original project's **workflow files** (`.github/workflows/`)
but **not its secrets**. Those workflows still run *in your repo*. Podinfo's
`release.yml` would try to sign and publish images. So:

| Check | Why |
|---|---|
| **Turn Actions off** until you write your own workflows (P7) | The original author's pipeline shouldn't run in your repo |
| **Workflow token read-only** | If a workflow is ever compromised, it can't write anything. This is *least privilege*: give something only the access it needs. |
| **Secrets list is empty** (`gh secret list`) | Confirms nothing was inherited |
| **Remember it's public** | A fork of a public repo can't be made private. Never commit anything you wouldn't publish. |
| **Note the default branch** | Podinfo uses `master`, not `main`. Workflows and branch protection must use the right name. |
| **Stay current on purpose** | `gh repo sync` when *you* want the original's updates |

```bash
gh api -X PUT repos/nazmur96/podinfo/actions/permissions -F enabled=false
gh api -X PUT repos/nazmur96/podinfo/actions/permissions/workflow \
  -f default_workflow_permissions=read -F can_approve_pull_request_reviews=false
```

Check each setting by reading it back (`gh api repos/…/actions/permissions`),
not by trusting that the command ran.

### In a real job

```bash
gh auth login
gh repo clone company/payments-api
cd payments-api
git switch -c feat/PAY-123-add-healthcheck   # branch name from the team's conventions
# … edit, commit …
git push -u origin HEAD && gh pr create      # PR into the same repo
```

What's different:

- **You never push to `main`.** It's protected: PRs, required reviews and
  checks, code owners, sometimes signed commits.
- **Access goes through the company,** usually with single sign-on (SAML
  SSO). If a clone fails with **403 or 404 on a repo you know exists**, your
  token or SSH key probably isn't authorised for the company's SSO yet. The
  company knows who you are, but your Git credentials haven't been approved
  for its repos. Fix: `gh auth refresh`, then authorise SSO (or GitHub →
  Settings → SSH keys → *Configure SSO*). Lesson: an access error isn't always
  a URL or network problem; it can be an identity problem.
- **It may not be GitHub.com:** GitHub Enterprise, GitLab, Bitbucket or Azure
  DevOps. Git commands stay the same; only the login and CLI change.
- **The org manages secrets and runners.** You use them; you usually can't see
  them.
- **First day:** read `README.md`, `CONTRIBUTING.md` and `CODEOWNERS`, and run
  the app locally *before* changing anything.

### The project folder

All three repos sit side by side in one project folder, with a **symlink** to
this template so Claude and Cursor both see it next to your code:

```
~/Downloads/Projects/podinfo/
├── podinfo/              the app (your fork)
├── podinfo-gitops/       deployment config (created in P1-1)
├── platform/             Terraform and the Phase 0 docs
└── devsecops-template -> ~/orca/projects/devsecops
```

A link, never a copy: there's one template, and a fix made through the link
lands in the real repo. Keep the link in the project folder, never inside a
repo, or it gets committed.

---

## 2. Decide where each kind of work lives

*Ticket P1-1*

There are three different jobs, so there are three repos:

| Repo | Holds | Question it answers | Who changes it |
|---|---|---|---|
| **App** (`podinfo`) | Source code, Dockerfile, CI workflows, `mise.toml`, `devcontainer.json` | How do we build and test it? | Developers and CI |
| **Platform** (`platform`) | Terraform: VPC, EKS, IAM, ECR | What infrastructure does it run on? | Platform engineers |
| **GitOps** (`podinfo-gitops`) | Helm values per environment, Argo CD apps | Which version runs where, with what settings? | CI (image digests only) and platform engineers |

The platform repo is always called `platform` in the project folder. On
GitHub its name must be unique in your account, so it's `<app>-platform`
(`nazmur96/podinfo-platform`), cloned with
`gh repo clone nazmur96/podinfo-platform platform`. Anything that matches on
the GitHub name, like the AWS trust policy for CI (P5), uses
`podinfo-platform`.

While learning, app and platform code *may* share a repo. **The GitOps repo is
always separate.** Record the layout in an ADR.

Every repo gets a `README.md`, `.gitignore`, `.editorconfig`, `CODEOWNERS`,
and branch protection on the default branch (PRs required, checks required,
no force-push).

> **In the Podinfo run:** GitHub refused branch protection: private repos on
> a free plan can't have it (a free *organisation* can't either). So it's a
> deferred ticket, to close before CI writes to the GitOps repo (P7). Until
> then every change goes branch → PR → squash-merge by habit, and a local
> hook (`no-commit-to-branch`) refuses commits on `main`. A habit isn't
> enforcement: writing the gap down is what keeps it honest.

### What GitOps means

Think of the cluster as a kitchen and **Argo CD** as the kitchen manager. The
GitOps repo holds the approved recipe: *"run three copies of this exact
image, with these settings."* Argo CD keeps checking the kitchen and fixes
anything that doesn't match the recipe.

```mermaid
flowchart LR
    G["GitOps repo<br/>what should run"] --> A[Argo CD] --> K["Cluster<br/>what actually runs"]
    K -.->|differs? Argo CD fixes it| A
```

### Why Argo CD gets its own repo

| If Argo CD watched the app repo | With a separate GitOps repo |
|---|---|
| Every code commit or README typo wakes it up | It reacts only to real deployment changes |
| CI needs write access to the **source code** to bump versions: a big risk if CI is hacked | CI can write **only** to the GitOps repo, and only image digests |
| "What's in prod, since when?" is buried in code history | The GitOps history **is** the deploy log: every commit a deploy, every revert a rollback |
| Developers and prod approvers share one set of permissions | `envs/prod/` gets its own reviewers |
| CI commits trigger CI again (loops, `[skip ci]` hacks) | No loops |

### Inside the GitOps repo

```
podinfo-gitops/
├── bootstrap/root-app.yaml   the one Argo CD app you apply by hand, once
├── apps/                     Argo CD application definitions
├── charts/                   the Helm chart
├── platform/                 settings for platform tools
└── envs/
    ├── dev/values.yaml       image digest written by CI
    ├── staging/values.yaml   digest promoted from dev
    └── prod/values.yaml      digest promoted from staging, reviewed
```

A release moves like this:

```mermaid
flowchart LR
    B["CI builds the image<br/>sha256:abc…"] --> D["envs/dev<br/>CI writes the digest"] -->|tests pass| S["envs/staging<br/>same digest"] -->|reviewed| P["envs/prod<br/>same digest"]
```

**Promote** means moving an image you already built and tested to the next
environment. You don't rebuild for prod; you ship the exact image staging
tested, identified by its **digest** (an ID calculated from the image's
contents, so it can't secretly change like a `latest` tag can).

The only automated writer is CI's **GitHub App** token, and it may only change
the image digests in `envs/dev/`. Prod changes need a review (on a solo
project: the GitHub Environment `prod` reviewer, see the
[Phase 0 explainer](p0-explained.md#10-agree-who-owns-what)).

### No secrets in the GitOps repo. Ever.

Deploy config may say **where** a secret lives, never **what** it is:

```yaml
# Wrong: the password is now in Git history forever, even if you delete the line
password: my-real-password

# Right: a reference; External Secrets fetches the value from AWS Secrets Manager
secretStoreRef: aws-secrets-manager
remoteRef: podinfo/dev/podinfo/jwt-secret
```

(The second block is illustrative, not a complete ExternalSecret.)

### `.gitignore` from day one

```gitignore
*.tfstate          # Terraform state stores secrets in plain text
*.tfstate.*
.terraform/        # downloaded providers and working files
*.tfvars.json
crash.log
.env               # local passwords, tokens, connection strings
.env.*
!.env.example      # …but the safe example file IS committed
node_modules/      # dependencies and build output: regenerated, never committed
target/
dist/
```

> **Trap:** `.gitignore` only affects files Git isn't tracking yet. If a
> secret is already committed, ignoring it now does nothing. Remove it from
> tracking and **rotate the secret**: assume it's leaked.

---

## 3. Pin the tools with `mise.toml`

*Ticket P1-2*

### The problem: "works on my machine"

You have Terraform 1.9, a colleague 1.6, someone else 1.8. Same code, three
different results. This is **toolchain drift**, and it happens to kubectl,
Helm, runtimes and scanners too.

### The fix: write the versions in the repo

**mise** is a tool-version manager. You list the tools and versions in
`mise.toml`; mise installs exactly those, and switches to them whenever
you're in that folder.

```toml
[tools]
terraform = "1.9.8"
kubectl   = "1.36.5"     # the SAME minor as your cluster: choose that first
helm      = "3.16.2"
kind      = "0.27.0"
awscli    = "2.22.0"
trivy     = "0.57.1"
hadolint  = "2.12.0"
gitleaks  = "8.21.2"
cosign    = "2.4.1"
syft      = "1.17.0"
uv        = "0.13.1"     # needed by the two pipx: tools below
"pipx:checkov"    = "3.2.300"
"pipx:pre-commit" = "4.0.1"
```

These are the template's **example** versions. Check current ones with
`mise latest <tool>` and pin exact versions. (Podinfo's `platform/mise.toml`
pins the AWS CLI at `2.38.1`, and kubectl at `1.36.5` to match the cluster,
ADR-0010.)

What each tool is for:

| Tool | In one line | Used in |
|---|---|---|
| terraform | Creates AWS infrastructure from code | P4 |
| kubectl | Talks to a Kubernetes cluster | P3 on |
| helm | Installs apps into Kubernetes from packaged "charts" | P3, P8 |
| kind | Runs a whole Kubernetes cluster inside containers on your machine | P3 |
| awscli | Talks to AWS from the terminal | everywhere |
| trivy | Scans images and files for known vulnerabilities | P2, P7 |
| hadolint | Checks Dockerfiles for mistakes | P1-4, P2 |
| gitleaks | Finds secrets before they're committed | P1-4 |
| cosign | Signs container images and checks signatures | P7, P8 |
| syft | Lists everything inside an image (an SBOM: software bill of materials) | P7 |
| checkov | Checks Terraform for security mistakes | P4 |
| pre-commit | Runs checks automatically when you commit | P1-4 |

### Check it worked

```bash
mise install          # install every tool at its pinned version
mise doctor           # report setup problems
which terraform       # must point into mise, NOT /usr/bin/terraform
```

The `which` check catches a sneaky problem: the right version is installed,
but your shell runs a different copy that was installed somewhere else.

> **Trap: your laptop can lie.** Some tools need a helper to install: the
> `pipx:` ones need `uv` or `pipx`. If the helper isn't pinned in `mise.toml`,
> mise quietly borrows whatever your laptop has, and `mise install` passes.
> A clean container has nothing to borrow, so it fails. **In the Podinfo
> run** NixOS had `uv` installed, so the laptop passed and the container
> failed until `uv` was pinned. Lesson: the clean container is the real test,
> not your laptop.

### The `[env]` section

```toml
[env]
AWS_REGION  = "eu-central-1"                     # Frankfurt
AWS_PROFILE = "sockshop-dev"                     # which AWS login profile to use
KUBECONFIG  = "{{config_root}}/.kube/config"     # this project's own cluster list
```

- **AWS_PROFILE** only *picks* a profile. It doesn't log you in or give you
  permissions, and it must never hold static keys. For Podinfo the profiles
  are `authentik` (CLI, via `aws login`) and `authentik-tf` (Terraform), from
  ADR-0004.
- **KUBECONFIG** per project means `kubectl` here only knows this project's
  clusters, so a command meant for local kind can't hit a real EKS cluster by
  accident. That file holds cluster credentials: add `.kube/` to `.gitignore`.

### "A runtime comes from the image or from mise, never both"

A **runtime** is what runs code in a given language: Go, Java, Node, Python.
If the container image ships Java 21 and `mise.toml` installs Java 17, which
one runs? Whichever comes first on `PATH`, silently. That's a bug waiting to
happen.

Pick one owner per runtime and write it in the README. Either:

- the image provides the runtime, and mise handles only tools; or
- the image is a plain OS, and mise provides the runtime too.

> **In the Podinfo run:** Go `1.27.0` (from `go.mod`) comes from **mise**;
> the container image is plain Ubuntu with no Go. The README says so, and
> inside the box `which go` points into mise.

### The alternative: `devenv.nix`

`devenv.nix` is a Nix-based way to describe the whole dev environment. Pick it
when you also want the dev environment to run **services** like Postgres or
Redis, which mise can't do. **One or the other per repo, never both.**

---

## 4. Make the whole environment reproducible

*Ticket P1-3*

### Why a dev container?

Install everything on your laptop, and six months later another project needs
different versions, `PATH` gets edited, something breaks, and a new laptop
means doing it all again. A **dev container** is a workshop in a box: its
contents are described in files, so anyone can rebuild the same box.

### Three files, three jobs

| Piece | Its one job | Question it answers |
|---|---|---|
| `devcontainer.json` | The box: base operating system, system libraries, editor extensions | What's the environment? |
| `mise.toml` | Tool versions inside the box | Which versions? |
| **DevPod** | Creates the box and opens your editor in it | Where does it run? |

Because the jobs are separate, you can move the box (your laptop today, a
remote server later) by changing one flag, without touching the other files.
On your workstation, DevPod's `docker` provider is backed by **Podman**.

Think of it as layers, each owned by one file:

```
your laptop (NixOS)          the editor window, Podman. Nothing project-specific
└─ DevPod                    WHERE the box runs (local, or a server over ssh)
   └─ the box                OS + system libs + Docker-in-Docker   ← devcontainer.json
      └─ the tools           go, terraform, kubectl… exact versions ← mise.toml
         └─ the Git hooks    checks on every commit                ← .pre-commit-config.yaml
```

Change one layer and the ones above it don't notice.

### The example `devcontainer.json`, line by line

```jsonc
{
  "name": "sockshop-platform",                                         // just a label
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu-24.04",       // the base OS: Ubuntu 24.04
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {}           // a container engine inside the box
  },
  "remoteEnv": {
    "PATH": "${containerEnv:PATH}:/home/vscode/.local/bin:/home/vscode/.local/share/mise/shims"
  },                                                                   // so the shell finds mise's tools
  "postStartCommand": "…switch to iptables-nft, start dockerd…",       // runs on EVERY start (see the trap below)
  "postCreateCommand": "curl -fsSL https://mise.run | MISE_VERSION=v2026.10.9 sh && ~/.local/bin/mise trust && ~/.local/bin/mise install && if [ -f .pre-commit-config.yaml ]; then ~/.local/bin/mise exec -- pre-commit install; fi"
}
```

Why a container engine *inside* the container? To run `docker build` and
**kind**, which runs Kubernetes nodes as containers.

There are two hooks, and the difference matters:

- **`postCreateCommand` runs once**, when the box is first built. Use it
  for installs.
- **`postStartCommand` runs on every start.** Use it for anything that must
  be true each time the box comes up.

Read the `&&` in `postCreateCommand` as "and then, only if that worked":

```mermaid
flowchart LR
    A[Box created] --> B[Install mise, pinned version] --> C[Trust this repo's mise.toml] --> D[Install the pinned tools] --> E[Install the Git hooks, if configured]
```

Things to stay aware of:

- `curl … | sh` downloads a script and runs it straight away. That's a trust
  decision; a company may require an approved installer instead. Without
  `MISE_VERSION`, it also installs *whatever is newest today*: **in the
  Podinfo run** the box got mise 2026.10.9 while the laptop had 2026.8.6.
- `mise trust` lets that repo's config run. Fine for your own repo; read a
  stranger's config before trusting it.

### Start it, check it, rebuild it

```bash
devpod up . --provider docker        # create the box and open the editor
# inside the box:
terraform version
kubectl version --client
kind version                         # all must match mise.toml
devpod delete . && devpod up .       # throw it away and rebuild it
```

The rebuild must give the **same** environment in **under 10 minutes**.
"I built it once and it worked" isn't reproducible; "I deleted it and got it
back" is. Neovim users: `devpod up . --ide none`, then
`ssh <workspace>.devpod`.

> **Known problem on rootless Podman:** kind inside the box needs
> docker-in-docker in *privileged* mode, which can be unreliable. The
> fallback: run kind from the host's Podman with
> `KIND_EXPERIMENTAL_PROVIDER=podman kind create cluster`, using the **same
> `mise.toml`** on the host (mise on the host is allowed; the tools still
> belong to the repo). Pick one, test it, record it in an ADR.

> **Trap on NixOS (nftables-only hosts):** Docker inside the box died with
> `can't initialize iptables table 'nat'`. A container has no kernel of its
> own, so Docker programs your *laptop's* firewall, and the Docker-in-Docker
> add-on picks the old firewall interface (legacy iptables) that NixOS
> doesn't have. The `postStartCommand` switches it to the new one
> (`iptables-nft`) and starts Docker. It's in `postStartCommand`, not
> `postCreateCommand`, because the add-on resets the choice every time.

A container isn't automatically secure. Privileged mode, docker-in-docker and
access to the container engine are all real permissions. Know what you've
granted.

---

## 5. Catch mistakes before they're committed

*Ticket P1-4*

### What a pre-commit hook is

A check that runs **every time you commit**, on your machine. If it fails,
the commit doesn't happen. It's the cheapest place to catch a mistake,
especially a leaked password: once a secret is pushed, it's in Git history and
may already be copied, so you must assume it's leaked and rotate it.

| Check | What it catches |
|---|---|
| **Gitleaks** | Text that looks like a password, token or cloud key |
| detect-private-key | Private keys |
| check-added-large-files | Big files added by accident |
| check-yaml, check-json | Broken YAML or JSON |
| trailing-whitespace, end-of-file-fixer | Messy whitespace (fixed for you) |
| **hadolint** | Dockerfile mistakes |
| terraform_fmt, terraform_validate | Unformatted or invalid Terraform |
| conventional-pre-commit | Commit messages that don't follow `type: summary` (runs on the *message*, not the files) |
| no-commit-to-branch | Commits made directly on `main` |

They're listed in `.pre-commit-config.yaml`. The shape is simple: `repos`
says where each check comes from, `rev` **pins its version** (scanners change
too), and `hooks` picks which checks to run. `pre-commit install` (already in
`postCreateCommand`) wires it into Git. Put
`default_install_hook_types: [pre-commit, commit-msg]` at the top of the
config, so that one plain `pre-commit install` sets up both the file checks
and the message check.

Use the hook `gitleaks-system`, which runs the gitleaks that mise pinned. The
plain `gitleaks` hook downloads Go and builds gitleaks from source on first
use; **in the Podinfo run** that download failed on the NixOS laptop.

### Prove it works, don't assume it

1. Create a throwaway file containing a **fake** AWS key: `AKIA` followed by
   16 **random** characters. Never a real key, and not AWS's documentation
   key `AKIAIOSFODNN7EXAMPLE`: gitleaks deliberately ignores that one.
2. Try to commit it.
3. The commit must be **rejected, and the output must say why**: the
   gitleaks hook `Failed`, with `RuleID: aws-access-token`.
4. Remove the file, make a normal commit, and check that it **passes**.

A config file can exist and still not work. And a rejection can be a
*crash*, not a catch: **in the Podinfo run** the first drill was rejected
three times for the wrong reasons (a failed download, a missing config) while
the hooks weren't even installed. Read the output, not just the result.

The hooks must also be **fast**: under 5 seconds on a normal commit, or
people start skipping them.

### It's the first layer, not the whole wall

Anyone can skip local hooks with `git commit --no-verify`, and secret
scanners don't catch everything. So CI runs the same checks again (P7), where
a developer can't skip them. `CONTRIBUTING.md` says both.

```mermaid
flowchart TD
    A["Your machine: pre-commit hooks<br/>fast, but can be skipped"] --> B[Push and open a PR]
    B --> C["CI runs the same checks<br/>can't be skipped"]
    C --> D[Review and merge]
```

---

## 6. Run the app exactly as the developers do

*Ticket P1-5*

### Why before Kubernetes?

Imagine the app fails to start on EKS. Is it a code bug, a missing env var,
the database, the image, the Service, a network policy, a missing secret?
Too many suspects. If you've already run it locally, half of them are ruled
out. **Get a known-good baseline before adding complexity.**

So, before containerising anything:

- Build and start each service **following its README**.
- List **every env var it reads** in `docs/inventory.md`, found by searching
  the code, not by trusting the README. (The README might say `DATABASE_URL`
  while the code reads `DB_HOST`.)
- Run the **CUJ** (the Critical User Journey from P0-3) end to end, locally.
  Starting isn't the same as working: a process can start fine and still fail
  when a user does something real.
- Wherever the README was wrong, **fix the README** (a PR to the developers).

> **In the Podinfo run:** upstream's README has no local-run section, but its
> `Makefile` does: `make run`, inside the app's dev container (`make` isn't
> on the laptop, by design). Port 9898, and CUJ steps 1–6 from
> `platform/docs/cuj.md` all passed. The app README now documents it.

### How does a program know its own version?

The compiled program can't look at Git. The build has to **write the commit
into it**, in Go with a linker flag (`-ldflags "-X …REVISION=<sha>"`), in a
Dockerfile with a build argument. Nothing written means `unknown`.

| How it was built | What `/version` says |
|---|---|
| `go run ./cmd/podinfo` | `unknown` |
| `make run` | `6.15.0-11-g6f667bcd` (`git describe`: the short SHA is inside) |
| CI, from P7 | must be the **full commit SHA**, passed as `--build-arg REVISION` |

Why care: "which code is running right now?" is the first question in every
incident and rollback. An image that can't answer it leaves you guessing.

---

## 7. Understand the app the way an operator needs to

*Ticket P1-6*

You don't need to be the app's developer. You do need to know how it
**starts, takes requests, fails and stops**. First, for each service, find
four things: its **entry point** (where it starts), its **dependency
manager**, its **build tool**, and its **final output**. For Podinfo:
`cmd/podinfo/main.go`, Go modules (`go.mod`), `go build` via the `Makefile`,
and a single static binary.

Then answer eight questions and record them in `docs/inventory.md`:

| Question | How to find out | Why it matters later |
|---|---|---|
| How does a request flow? | Follow one CUJ step through the code | Network rules (P8), tracing (P10) |
| What does the health check check? | `curl` it, then read its code | Probe design (P6) |
| Logs: structured, to the console? | Run it and look | Log collection (P10) |
| Handles SIGTERM? | `time docker stop <container>` | Draining and grace period (P6) |
| Config from env vars? | Search the code | ConfigMaps and Secrets (P5) |
| Writes to local disk? | Run it with a **read-only** filesystem | Read-only root filesystem + `emptyDir` (P6) |
| How long to start? | Time it | Startup probe (P6) |
| Memory under load? | `docker stats` while the CUJ runs | Requests and limits (P2, P6) |

### 1. Request flow

Trace one real request through the components:

```mermaid
flowchart LR
    User --> Frontend --> Backend --> Database
```

Later, network policies allow only these paths. Get one wrong and you block
the app, or allow far too much.

### 2. Health checks: three different questions

| Probe | Question | Fails → Kubernetes… |
|---|---|---|
| **Liveness** | Is it stuck? | Restarts the container |
| **Readiness** | Should it get traffic right now? | Stops sending it traffic |
| **Startup** | Has it finished starting? | Waits before running the other two |

**The classic mistake: a liveness check that tests the database.** The
database blips → liveness fails → Kubernetes restarts the app → the database
is still down → restart again, and again. A database problem has become a
restart storm. **Liveness must never check the database.** Readiness *may*
check a dependency the app truly can't work without.

> **In the Podinfo run:** confirmed. `/healthz` and `/readyz` are simple
> in-memory flags that check no dependencies, which is right for liveness.
> Both switch to failing the moment SIGTERM arrives.

### 3. Logs

Containers should write logs to the console (**stdout or stderr**), not to a
file inside the container, so the platform can collect them:

```mermaid
flowchart LR
    A["App writes to stdout/stderr"] --> B[Container runtime] --> C[Log collector] --> D[Central log search]
```

**Structured** logs are machine-readable, usually JSON, so you can filter by
field:

```json
{"level":"error","service":"checkout","msg":"payment request failed"}
```

> **In the Podinfo run:** it logs JSON, but to **stderr**, not stdout
> (`cmd/podinfo/main.go`, `OutputPaths: stderr`). Collectors read both, so
> that's fine, but write down which one. Its logger also **samples**: after
> 100 identical messages in a second, it keeps only every 100th. Remember
> that before trusting a log count during an incident.

### 4. SIGTERM: stopping politely

When Kubernetes replaces a pod (every deploy), it asks politely first:

```mermaid
flowchart TD
    A["Kubernetes sends SIGTERM"] --> B[App stops accepting new requests]
    B --> C[App finishes the requests it already has]
    C --> D[App exits]
    A -.->|"no exit before the grace period ends"| E["SIGKILL: killed mid-request"]
```

The quick test: `time docker stop <container>`. Docker waits 10 seconds
before forcing it. **If stop takes about 10 seconds, the app ignored SIGTERM
and was killed.** A fast stop is a good sign, but only half the test.

The other half: stop it **while a slow request is still running**, and check
that request finishes. An idle stop can't show whether the app gives
in-flight work enough time.

> **In the Podinfo run:** an idle stop took 3 seconds: it marks itself
> unready, waits 3 s for the load balancer to notice, then shuts down. Looks
> perfect. But its 5-second shutdown timer **starts at SIGTERM, before that
> wait**, so in-flight requests only get about **2 seconds**:
>
> ```
> 0 s ── SIGTERM: timer starts, app marks itself unready
> 3 s ── wait over: stop taking new connections, finish open ones
> 5 s ── timer runs out: any request still open is cut off
> ```
>
> A request needing 6.5 s more was cut at 5.0 s, **and the app still exited
> with code 0**, so nothing looked wrong. (At debug log level the 3 s wait is
> skipped entirely, which is why `make run` isn't how you test draining.)

### 5. Config from environment variables

Settings should come from outside the code, so **the same image** runs in dev
and prod with different settings. Plain values go in a ConfigMap later;
secrets go through Secrets Manager (P5). Search the code for what the app
*really* reads.

### 6. Writing to disk

Container filesystems are temporary: replace the container and anything it
wrote is gone. For security we'll also make the filesystem **read-only**
(P6). So find out *now* where it writes: run it with a read-only filesystem
and see what breaks. Any folder it truly needs gets a small temporary volume
(`emptyDir`).

> **In the Podinfo run:** `/store` writes to `/data` (`--data-path`), so it
> needs an `emptyDir` there. That's half the answer: the read-only run that
> proves *nothing else* writes needs a container, so it moved to P2-4.

### 7. Startup time

If a probe expects an answer before the app is ready, Kubernetes thinks the
app is broken and kills it, over and over: a crash loop that looks like an
app bug. Measure startup, and add a startup probe if it's slow. (Big Java apps
can take a minute or more; **Podinfo measured 50–100 ms**, so no startup
probe.)

### 8. Memory under load

Run the CUJ and watch `docker stats`. Too low a memory limit later means the
app gets killed (OOMKilled); too low a request means Kubernetes packs too much
onto one node. Measure, don't guess. (Java needs extra care: the JVM must be
told how much of the container's memory it may use.)

> **In the Podinfo run:** 38 MiB idle, 42 MiB after the CUJ. The upstream
> Helm chart asks for only 16 Mi, less than the app uses doing nothing. Real
> load testing comes in P2-9.

---

## 8. Optional: Linux and Bash practice

*Ticket P1-7 [OPT]: skip it if you're already comfortable with Linux.*

Do it on a **throwaway VM** (Proxmox, Multipass or cloud), never on your
workstation, because automation mistakes can lock you out or break the
system.

| Exercise | What it teaches |
|---|---|
| **Idempotent** provisioning script | Running it twice gives the same result; the second run changes nothing |
| SSH hardening, with **a second session kept open** | If you break SSH, the open session lets you fix it |
| Backup with a checksum and a **tested restore** | A backup you've never restored is only a hope |
| logrotate and an `awk` filter for `ERROR`/`FATAL` | Logs fill disks; real troubleshooting starts with filtering logs |
| The same thing again as **Ansible** roles with Molecule tests | When a structured tool beats a script |

**Idempotent** in plain words:

```
First run:   install package · create folder · apply config
Second run:  already installed · already exists · already correct → no changes
```

The script also needs a root check, OS detection, logging, a clean
`shellcheck`, and **retries with backoff**: if a download fails, wait 1 s,
then 2 s, then 4 s, instead of retrying non-stop.

---

## 9. Not yet: inner-loop tools

The **inner loop** is the developer's cycle: change code → rebuild → see the
result → repeat. These tools speed it up, but they need running workloads, so
they come in **P3-9**:

| Tool | What it does | Pick |
|---|---|---|
| Skaffold | Watches code, rebuilds the image, redeploys | Skaffold **or** Tilt, not both |
| Tilt | The same, plus live file sync and a web UI | |
| mirrord | Runs one service on your laptop while it gets the cluster's traffic and config | Alongside either |
| vCluster | A virtual cluster per developer inside a shared one | A different problem: *where* the loop runs (P8-10) |

---

## How you know Phase 1 is done

1. **Fresh machine or fresh workspace:** `git clone && devpod up .` → every
   tool works at its pinned version → the CUJ runs locally, **with no other
   manual step.**
2. **A fake secret is rejected** at commit time.
3. **`docs/inventory.md` answers all eight questions** for every service.

**In the Podinfo run** all three passed from a fresh clone, in a brand-new
workspace, with no manual step.

## Working alongside an AI agent

A repo folder has **one current branch and one staging area**, shared by
everyone working in it. When two agents (or you and an agent) use the same
folder, one switching branches redirects the other's next commit. **In the
Podinfo run** that put a commit, plus someone else's unfinished files, on
the wrong branch. The fix is `git worktree`: a second folder on the same
repository, with its own branch.

```bash
git worktree add ../wt-mine -b my-branch origin/main   # my own folder and branch
git -C ../wt-mine add docs/x.md                        # stage by name, never `git add -A`
git worktree remove ../wt-mine                          # clean up after the merge
```

## Common mistakes

| Mistake | Why it hurts |
|---|---|
| Tools installed by hand on the host | Drift: nobody can reproduce your setup |
| The same runtime from two places | `PATH` order silently decides which runs |
| `*.tfstate` missing from `.gitignore` | State holds passwords in plain text |
| Skipping the local run | You debug app bugs inside Kubernetes, the hardest place to do it |
| Pre-commit as the only secret scanner | `--no-verify` skips it; CI must check again |
| Trusting the laptop's `mise install` | It can borrow unpinned tools; the clean container is the test |
| Judging a drill by "it failed" | A crash looks like a catch; read the reason |
| Testing SIGTERM only when idle | In-flight requests can still be cut, with exit code 0 |
| Pinning kubectl from an example | It must match the cluster's version; choose that first |
| Two agents in one checkout | Commits land on the wrong branch; use a worktree each |

## What to remember

You don't need to memorise every line of these files. Remember what each one
is **for**:

| File or tool | Its job |
|---|---|
| Three repos | Where code, infrastructure and "what runs where" each live |
| `mise.toml` | Which tool versions this project uses |
| `devcontainer.json` + DevPod | The environment, and where it runs |
| Pre-commit hooks | Catch secrets and mistakes before they're committed |
| Local run + inventory | Proof the app works, and how it behaves |

**The big idea: reproducibility plus understanding.** Another engineer, or
you on a new machine, can rebuild everything from committed files. And you
know what the app does, what it depends on, and how it behaves when things go
wrong, before Kubernetes adds its own problems.

**Next:** practise with the [Phase 1 interview cases](p1-dev-environment.md), then
[Phase 2: build the container image](../template/02-containerise.md).
