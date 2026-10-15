# Study bank — Phase 1: Workstation & dev environment

Interview practice for [Phase 1](../template/01-dev-environment.md). Each
**case** is how a scenario interview actually runs: the interviewer sets a
scene and asks an open question, then pushes with follow-ups to see how deep
your understanding goes.

**New to Phase 1?** Read [Phase 1 explained simply](p1-explained.md) first,
then come back here to practise.

**How to use it:** treat it as a mock interview. Read the scene, answer out
loud for about two minutes, *then* open the model answer. Do the follow-ups
the same way. There is rarely one right answer. The model answer shows the
reasoning a strong candidate gives, and **Weak answer** shows what makes an
interviewer stop listening.

**Evidence:** answers are general first. **In the Podinfo run** notes give
evidence from the [Podinfo fork](https://github.com/nazmur96/podinfo) (read
at `3e0ff8a`, tag `6.15.0`) and its platform repo. Use them as interview
stories.

`★` = came from a real finding or mistake in a run.

**Adding a question:** add a follow-up under the case it belongs to (next
number, e.g. `6.4`), or a new case at the end of its section's numbering.
Phrase it as something an interviewer would say, give the model answer, and
cite the file/line, ADR or RCA that proves it. Never renumber existing cases
or follow-ups: references like "study P1 6.2" must stay valid.

---

## Big picture

### 1. "Onboarding takes two days" ★

> **Interviewer:** New engineers on your team spend their first two days
> installing tools, and their setups never quite match. How would you fix
> that?

<details>
<summary><b>Model answer</b></summary>

Make the environment a set of **committed files** instead of a list of
instructions:

- `mise.toml` pins the exact version of every tool (Terraform, kubectl,
  Helm, scanners).
- `devcontainer.json` defines the box they run in: base OS, system
  libraries, editor extensions.
- **DevPod** creates that box and opens the editor in it, locally or on a
  remote host.
- Pre-commit hooks install automatically as the box is created.

Onboarding becomes `git clone && devpod up .`. Nothing project-specific is
installed on the laptop itself, so two projects needing different Terraform
versions never collide. And when someone changes a tool version, it goes
through a PR like any other change.

**Weak answer:** "I'd write a really good setup guide in the wiki." The wiki
is out of date the week after you write it, and nothing checks that anyone
followed it.
</details>

**Follow-ups**

<details>
<summary><b>1.1</b> "Why not just a setup script that installs everything?"</summary>

A script describes *steps*, not *state*. It runs once, and from then on the
machine drifts: someone upgrades one tool by hand, another project's script
installs a different version on top. A declared environment (versions in a
file, a box rebuilt from that file) can always be compared with what's
actually running, and recreated from scratch. If it isn't in a committed
file, it doesn't exist.
</details>

<details>
<summary><b>1.2</b> "How would you prove the setup really is reproducible?"</summary>

Destroy it and build it again: `devpod delete` then `devpod up`, and check
every tool version matches `mise.toml`, within about 10 minutes. The
stronger test is a fresh machine: clone, one command, then the CUJ runs
locally with **no other manual step**. "It worked when I built it" proves
nothing; "I threw it away and got it back" does.
</details>

<details>
<summary><b>1.3 ★</b> "A tool is installed twice on the same machine. Which one runs?"</summary>

Whichever directory comes first on `PATH`, and nothing tells you. The two
copies can be different versions, so behaviour depends on how your shell
happened to start. Worse, an install done by hand doesn't replace a declared
one; it silently *shadows* it. Rule: every tool has exactly one owner.

**In my own setup:** I once had three copies of Claude Code on one
workstation, from three install methods, each hiding the others. `which -a
<tool>` shows every copy on `PATH`.
</details>

---

## P1-0 — Get the code

### 2. "Practising on someone else's project"

> **Interviewer:** You want to practise building a CI/CD pipeline around an
> open-source app. Walk me through getting the code, step by step.

<details>
<summary><b>Model answer</b></summary>

I can't push to the original, so I **fork** it (my own copy on GitHub) and
**clone my fork**. That gives two remotes: `origin` (my fork, where I push)
and `upstream` (the original, where I only fetch updates). Then I make the
safe action the easy one:

```bash
gh repo fork <owner>/<app> --clone=false
gh repo clone <me>/<app>
gh repo set-default <me>/<app>               # gh commands target my fork
gh repo set-default --view                   # check: gh sometimes picks the original
git remote set-url --push upstream no-pushing   # pushing to the original fails safely
```

Before the first push I lock the fork down: Actions off, workflow token
read-only, secrets list confirmed empty, default branch noted (Podinfo uses
`master`). And I verify each setting by reading it back through the API.

**Weak answer:** "I'd just clone it." Then you can't push anywhere, or you
push your experiments at someone else's project.
</details>

**Follow-ups**

<details>
<summary><b>2.1</b> "In your next job, would you fork the company's repos too?"</summary>

Usually not. If you have write access to branches, you clone the company
repo directly, branch, and open a PR in the **same** repo. Fork only when you
can't push to the original. Some companies (often banks) require forks
anyway: then you fork *inside* the company org (it stays private) and PRs go
back to the company repo.
</details>

<details>
<summary><b>2.2</b> "<code>gh pr create</code> opened a pull request against the original project instead of your fork. How did that happen?"</summary>

With two remotes, `gh` has to guess which repo is "the" repo, and it often
picks the parent of a fork. That's why `gh repo set-default <me>/<app>`
comes right after cloning, followed by `--view` to confirm, because the
command succeeding doesn't prove the setting. Then close the stray PR
politely; maintainers see it.
</details>

<details>
<summary><b>2.3</b> "The original project released a new version. How do you bring it into your fork, and what can go wrong?"</summary>

`gh repo sync <me>/<app>`, or `git fetch upstream` then merge
`upstream/master`. Do it on purpose, not automatically. What goes wrong:
your own patches conflict, or an upstream change quietly reverts something
you rely on. So keep a short list of your patches in the repo, and have a
check that fails if a patch disappears.

**In the Podinfo run:** our build switches off `/env`, `/configs` and
`/panic` (ADR-0006, ADR-0008). A CUJ step probes those routes, so an upstream
sync that brings them back fails the CUJ instead of shipping silently.
</details>

### 3. "Day one at a company"

> **Interviewer:** It's your first day. You've been added to the company's
> GitHub org, but cloning the repo you were told to work on fails with a 404.
> The repo definitely exists. What's going on?

<details>
<summary><b>Model answer</b></summary>

Probably not the URL or the network: it's **identity**. Many companies sign
in to GitHub through single sign-on (SAML SSO). Being in the org isn't
enough; your `gh` token and SSH key must also be **authorised for the org's
SSO**. Until then, private repos answer 403 or 404, which looks exactly like
"the repo doesn't exist". Fix: `gh auth refresh` and authorise SSO, or
GitHub → Settings → SSH keys → *Configure SSO* → Authorize.

The general lesson: when access fails, check *who you are to that system*
before checking the address.
</details>

**Follow-ups**

<details>
<summary><b>3.1</b> "You're in. What do you do before changing a single line?"</summary>

Read `README.md`, `CONTRIBUTING.md` and `CODEOWNERS`, and run the app
locally the way the developers do. That tells me how the team works (branch
names, PR rules, who reviews what) and gives me a baseline: if something
breaks later, I know it worked before I touched it.
</details>

<details>
<summary><b>3.2</b> "Your push to <code>main</code> is rejected. Your PR can't merge either. Why?"</summary>

That's the protection working. `main` is protected: changes go through a
PR, with required reviews (often from `CODEOWNERS`), required status checks,
and sometimes **signed commits** (`git config commit.gpgsign true` with an
SSH or GPG key). The PR is blocked until every check is green and the right
people approve. Find out which rule blocks it from the PR's checks panel; don't
look for a way around it.
</details>

---

## P1-1 — Repository layout

### 4. "One repo or three?"

> **Interviewer:** Your app, its Terraform and its Kubernetes deployment
> config could all live in one repo. Why would you split them?

<details>
<summary><b>Model answer</b></summary>

Because they're three different jobs, with different people and different
risks:

| Repo | Answers | Changed by |
|---|---|---|
| **App** | How do we build and test it? | Developers and CI |
| **Platform** | What infrastructure does it run on? | Platform engineers |
| **GitOps** | Which version runs where, with what settings? | CI (image digests only) and platform engineers |

App and platform code *can* share a repo while learning. The GitOps repo is
the one I'd always keep separate: Argo CD watches it, its history is the
deploy log, and it lets me give CI write access to *deployment config only*,
not to the source code.

**Weak answer:** "Microservices best practice is one repo per thing." That's
a slogan, not a reason.
</details>

**Follow-ups**

<details>
<summary><b>4.1</b> "What actually goes wrong if Argo CD watches the app repo?"</summary>

- Every code commit or README typo wakes Argo CD up.
- To bump the image version, CI needs write access to the **source code**: if
  CI is compromised, so is the code.
- "What's in prod, and since when?" is buried in thousands of code commits.
- Developers and prod approvers share one set of permissions.
- CI commits the new version → that commit triggers CI again → loops, and
  `[skip ci]` hacks to break them.

With a separate GitOps repo, every commit is a deploy and every revert is a
rollback.
</details>

<details>
<summary><b>4.2</b> "CI needs to update the image version after each build. How much access does it get?"</summary>

As little as possible: a **GitHub App** token (short-lived, scoped to one
repo, not a person's token) that can write to the GitOps repo, and only the
image digests in `envs/dev/`. Promotion to staging and prod is a separate,
reviewed change. If CI is ever compromised, the worst it can do is propose a
dev image, which admission control later rejects if it isn't signed.
</details>

<details>
<summary><b>4.3 ★</b> "Your Terraform CI job fails: 'Not authorized to perform sts:AssumeRoleWithWebIdentity'. The trust policy allows <code>repo:you/platform</code>. The repo is called platform. What's wrong?"</summary>

It's called `platform` **on your disk**. On GitHub, names must be unique in
the account, so it's really `you/<app>-platform`, and the OIDC token's `sub`
claim carries the GitHub name. The trust policy never matches. Debug by
printing the token's claims in the job, not by loosening the policy. Rule:
when something has a local name and a remote name, identity checks always
match the remote one.

**In the Podinfo run:** the folder is `platform/`, the GitHub repo
`nazmur96/podinfo-platform`. The template used three names for it; fixed in
`d2b4295`, RCA `2026-10-14-p1-1-platform-repo-name.md`.
</details>

### 5. "The password was only there for one commit"

> **Interviewer:** A developer committed a database password to the GitOps
> repo, noticed straight away, and deleted it in the next commit. Are we fine?

<details>
<summary><b>Model answer</b></summary>

No. The password is still in Git history: anyone who cloned or forked the
repo in between has it, and anyone can still read the old commit. Deleting
the line hides it from the *current* files only. So:

1. **Rotate the password now.** Treat it as leaked. This is the only real fix.
2. Check the database's access logs for use since the commit.
3. Optionally rewrite history to remove it, but only *after* rotating; a
   rewrite doesn't undo copies already made.
4. Find out why the safety net missed it (no pre-commit hook? `--no-verify`?)
   and make sure CI scans too.

And the GitOps repo should never hold secret values at all, only
**references** to where the secret lives.

**Weak answer:** "Yes, it's deleted." Or: "Force-push to remove it," with no
mention of rotating.
</details>

**Follow-ups**

<details>
<summary><b>5.1</b> "So how <i>should</i> a secret appear in the GitOps repo?"</summary>

As a pointer, never a value: an `ExternalSecret` that names the store and the
path (`podinfo/dev/podinfo/jwt-secret`), and an IAM role name. External
Secrets Operator fetches the value from AWS Secrets Manager inside the
cluster. Reviewers, CI and Argo CD can all read the repo safely, because
there's nothing secret in it.
</details>

<details>
<summary><b>5.2</b> "You added <code>.env</code> to <code>.gitignore</code>, but Git still shows changes to it. Why?"</summary>

`.gitignore` only affects files Git isn't tracking yet. A file that was
already committed stays tracked. Stop tracking it with `git rm --cached .env`
and commit that, and if it ever held a secret, rotate the secret: it's in
history.
</details>

<details>
<summary><b>5.3</b> "Why is <code>*.tfstate</code> on the ignore list from day one? It's just Terraform's bookkeeping."</summary>

Terraform state stores every attribute of every resource, **including
secrets, in plain text**: database passwords, generated keys. Committed
state is a credentials dump. State belongs in a remote backend (an encrypted,
versioned S3 bucket, P4), and the ignore rule stops a local copy slipping
into Git before that exists.
</details>

---

## P1-2 — Pin the toolchain

### 6. "Works on my machine"

> **Interviewer:** You and a colleague run `terraform plan` on the same
> commit and get different results. Where do you start?

<details>
<summary><b>Model answer</b></summary>

First suspect: **different tool versions** (toolchain drift). Terraform,
its providers, kubectl and Helm all change behaviour between versions. I'd
compare `terraform version` on both machines, and the provider versions
locked in `.terraform.lock.hcl`.

The lasting fix is to make versions part of the repo: `mise.toml` pins the
exact version of every tool, and mise switches to them automatically in that
folder. Providers are pinned by the committed lock file. Then the same commit
means the same tools for everyone, including CI.

(If the versions match, the other suspects are different credentials or
different variable values: `AWS_PROFILE`, a local `.tfvars`.)
</details>

**Follow-ups**

<details>
<summary><b>6.1</b> "<code>mise install</code> succeeded, but <code>terraform version</code> still shows an old version. Why?"</summary>

Your shell is running a **different copy**, one installed elsewhere that
comes first on `PATH`. `which terraform` shows which file runs; it must point
into mise, not `/usr/bin`. Usually either mise isn't activated in that shell
(`mise activate` or its shims directory isn't on `PATH`), or an old
system-wide install shadows it. Remove the system copy: tools for a project
belong to the project. `mise doctor` reports activation problems.
</details>

<details>
<summary><b>6.2</b> "Pin exact versions, or allow ranges like '1.9.x'?"</summary>

Exact. A range means two people can still get different versions, which is
the problem you set out to fix. Upgrades then happen on purpose: bump the
version in a PR (a bot like Renovate can open those PRs), CI runs on it, and
everyone gets the new version on their next pull. For kubectl, also stay
within **one minor version** of the cluster, or some commands break.
</details>

<details>
<summary><b>6.3</b> "Your <code>mise.toml</code> sets <code>AWS_PROFILE</code>. Doesn't that put credentials in the repo?"</summary>

No. It only *names* a login profile; it holds no credentials and grants no
permissions. The profile itself uses short-lived credentials from a federated
login, never static access keys. You still have to sign in, and IAM still
decides what you can do. The benefit is that everyone working in this repo
uses the intended account and role by default.

**In the Podinfo run:** profiles `authentik` (CLI, via `aws login`) and
`authentik-tf` (Terraform), ADR-0004.
</details>

<details>
<summary><b>6.4</b> "Why does each repo get its own <code>KUBECONFIG</code>?"</summary>

So `kubectl` in this repo only knows this project's clusters. With one shared
kubeconfig, the "current context" might be prod while you think you're on
local kind, and a `kubectl delete` lands on the wrong cluster. A per-repo
file (`.kube/config`, git-ignored, because it holds cluster credentials)
makes the wrong cluster unreachable from here.
</details>

<details>
<summary><b>6.5</b> ★ "<code>mise install</code> passes on your laptop and fails in a clean container. Which one is lying?"</summary>

The laptop. Some mise backends need a helper: `pipx:` tools need `pipx` or
`uv`. If the helper isn't in `mise.toml`, mise quietly uses whatever the
host has, so the host passes because of something nobody pinned. The clean
container has nothing extra and tells the truth.

Pin the helper (`uv = "…"`) and treat the **clean-container run** as the
proof, not the host run. The same rule catches every "works on my machine"
leak: if a step only works because of something outside a committed file,
that thing doesn't exist yet.

**In the Podinfo run:** NixOS shipped `uv` system-wide; `pipx:checkov`
failed in the P1-3 container until `uv` was pinned. RCA
`platform/docs/rca/2026-10-15-p1-2-p1-3-devcontainer-on-nixos.md`.
</details>

<details>
<summary><b>6.6</b> ★ "Which kubectl version do you pin?"</summary>

The cluster's minor, so **choose the cluster version first**. kubectl is
supported within one minor of the API server. Three things must agree: the
kubectl pin, the local kind node image, and the EKS version. Decide them
together in one ADR and bump them together in one PR.

For EKS, pick the newest minor that is in **standard** support and that your
add-ons (Karpenter, load balancer controller, policy engine, GitOps) have
certified. Often that's newest-minus-one, because add-ons lag a brand-new
minor. Never start on a minor in **extended** support: it's billed at about
six times the normal control-plane rate.

**In the Podinfo run:** kubectl 1.33 was copied from the template example.
kind 0.33 created v1.37 clusters (four minors of skew), and EKS 1.33 was
already in extended support. ADR-0010 set 1.36 for all three.
</details>

### 7. "Two Go versions"

> **Interviewer:** The dev container image ships Go 1.22, and `mise.toml`
> also installs Go 1.27. A build passes in your terminal and fails in the
> editor. What's happening, and what's the rule?

<details>
<summary><b>Model answer</b></summary>

The terminal and the editor's language server found different Go binaries,
because they have different `PATH` orders. Same code, two compilers. The
rule: **each runtime has exactly one owner, the image or mise, never
both,** and the README says which.

Either the image is a plain OS and mise provides Go (easy to change per
repo, same file works on a host), or the image provides Go and mise handles
only tools (good when the image is shared across many repos). Both work; two
owners never does.

**In the Podinfo run:** `go.mod` says Go `1.27`. The owner gets decided and
written down in P1-2.
</details>

**Follow-ups**

<details>
<summary><b>7.1</b> "When would you use <code>devenv.nix</code> instead of <code>mise.toml</code>?"</summary>

When the dev environment should also **run services**, like Postgres or
Redis, which mise can't do, or when the team already works in Nix. It
describes the whole environment declaratively. One or the other per repo,
never both, or you're back to two owners.
</details>

---

## P1-3 — The dev container

### 8. "Three files for one environment?"

> **Interviewer:** Your setup uses `devcontainer.json`, `mise.toml` and
> DevPod. Why three things? Isn't that overkill?

<details>
<summary><b>Model answer</b></summary>

Each answers one question, so each can change without touching the others:

| Piece | Question |
|---|---|
| `devcontainer.json` | What's the box? Base OS, system libraries, editor extensions |
| `mise.toml` | Which tool versions inside it? |
| DevPod | Where does the box run? |

The payoff: moving from my laptop to a remote build server is **one flag**
in DevPod (`--provider ssh`), with the box and the tools unchanged. And
`mise.toml` works on a host too, so the same pins serve the container, a
plain laptop and CI.

**Weak answer:** "The tutorial used them." Or mixing the jobs up, like
installing tool versions in the Dockerfile and in mise.
</details>

**Follow-ups**

<details>
<summary><b>8.1</b> "<code>postCreateCommand</code> runs <code>curl https://mise.run | sh</code>. Any concerns?"</summary>

Yes: it downloads a script and runs it straight away, so I'm trusting that
URL and everything behind it, on every rebuild. In a company I'd use an
approved installer, a pinned and checksummed download, or bake the tool into
a reviewed base image. The same goes for `mise trust`: fine for my own repo,
but I'd read a stranger's config before trusting it, because it can run
commands.
</details>

<details>
<summary><b>8.2</b> "kind won't run reliably inside the dev container on your machine. What are your options?"</summary>

kind runs Kubernetes nodes as containers, so inside the box it needs
docker-in-docker in **privileged** mode, which is fragile on rootless Podman.
Options:

1. Make docker-in-docker work (privileged, the right cgroup setup, and
   `iptables-nft` on hosts without legacy iptables; see 8.5).
2. Run kind from the **host's** Podman with
   `KIND_EXPERIMENTAL_PROVIDER=podman`, using the same `mise.toml` on the
   host (mise on the host is allowed; the tools still belong to the repo).
3. Run the box on a remote host where privileged containers are fine.

Pick one, test it, and record it in an ADR, because the next person will hit
the same wall.
</details>

<details>
<summary><b>8.3</b> "Is the dev container a security boundary?"</summary>

Not one to rely on. A privileged container with docker-in-docker can do
nearly anything on the host, and anything that can reach the container
engine can start privileged containers itself. The box gives
*reproducibility*, not isolation. Treat code you run inside it as running on
your machine.
</details>

<details>
<summary><b>8.4</b> "Why do the language servers run inside the container instead of in your editor on the host?"</summary>

A language server needs the project's toolchain: the Go compiler, the
module cache, the exact versions. In the container, it sees the same
versions the build uses. On the host, it would need its own Go, which breaks
the "nothing project-specific on the host" rule and brings back two owners.
So the editor's *window* runs on the host, and its *brains* run in the box.
</details>

<details>
<summary><b>8.5</b> ★ "The dev container builds on your colleague's laptop, but on yours dockerd won't start: <code>can't initialize iptables table 'nat'</code>. What's going on?"</summary>

Docker programs the **host kernel's** firewall, even from inside a
container, because a container has no kernel of its own. Linux has two
firewall interfaces, legacy iptables and nftables. The docker-in-docker
feature switches the container to **legacy**; a host that only has
**nftables** (NixOS, and more distros every year) has no legacy `nat`
table, so dockerd dies. Your colleague's kernel still has the legacy
modules, so it works for them.

Fix it in a committed file, not by hand: a `postStartCommand` that switches
the container to `iptables-nft` and starts dockerd. It must run on every
start, because the feature's installer resets the choice in the image.
Then prove it: `docker info` and a `kind create cluster` inside the box.

**In the Podinfo run:** RCA
`platform/docs/rca/2026-10-15-p1-2-p1-3-devcontainer-on-nixos.md`.
</details>

---

## P1-4 — The commit-time safety net

### 9. "Keep secrets out of Git"

> **Interviewer:** How do you stop AWS keys and passwords from ever reaching
> your GitHub repos?

<details>
<summary><b>Model answer</b></summary>

In layers, because each one can fail:

1. **Pre-commit hook on every laptop** (Gitleaks, plus a private-key check):
   catches it before the commit even exists. Fast, but optional.
2. **The same scan in CI** on every PR: can't be skipped by a developer.
3. **The platform's own scanning** (GitHub secret scanning and push
   protection) as a backstop.
4. **Nothing worth leaking exists:** people and CI use short-lived
   federated credentials, not access keys, so there are few long-lived
   secrets to leak in the first place.
5. **A rotation runbook** for when one gets through anyway.

**Weak answer:** "We have a pre-commit hook." One layer that anyone can skip.
</details>

**Follow-ups**

<details>
<summary><b>9.1</b> "How do you know the hook actually works?"</summary>

Try to break it. Commit a throwaway file with a **fake** AWS key (`AKIA`
plus 16 characters), and the commit must be **rejected**. Then delete the
file and confirm `git status` is clean. A config file proves nothing on its
own: the hook might not be installed, or the rule might not match. Only a
blocked commit proves it.
</details>

<details>
<summary><b>9.2</b> "A developer commits with <code>--no-verify</code> to skip the hooks. Now what?"</summary>

Then CI catches it, because CI runs the same checks and nobody can skip
those. That's why `CONTRIBUTING.md` says both: hooks are for fast feedback,
CI is the gate. If it was a real secret, it's now in a pushed branch: rotate
it.
</details>

<details>
<summary><b>9.3</b> "The hooks take 40 seconds and people are starting to skip them. What do you do?"</summary>

Take it seriously: a slow hook trains people to use `--no-verify`, which is
worse than no hook. Aim for under **5 seconds** on a normal commit. Keep the
fast, local checks (secrets, formatting, YAML/JSON, Dockerfile lint) in the
hook, run only on changed files, and move slow checks (full Terraform
validation, image scans) to CI.
</details>

<details>
<summary><b>9.4</b> "Gitleaks blocks a commit, but it's a fake key in a test fixture. How do you get past it?"</summary>

Allow that one finding, not the whole check: add its fingerprint to
`.gitleaksignore`, or put a `gitleaks:allow` comment on the line, in a
reviewed commit. Never turn the rule off or teach people `--no-verify`. And
double-check it really is fake before allowing it.
</details>

<details>
<summary><b>9.5</b> "Why pin versions in <code>.pre-commit-config.yaml</code>? It's just linters."</summary>

Scanners change too: a new version adds rules or changes defaults. Unpinned,
the same commit could pass today and fail tomorrow, or pass on one machine
and fail on another. Pinned (`rev:`), the checks are as reproducible as the
tools, and upgrades happen in a PR.
</details>

<details>
<summary><b>9.6</b> ★ "Your secret-leak drill was rejected. Is the hook working?"</summary>

Not necessarily. **Read why it was rejected.** A commit can fail because
the hook *crashed*: a missing config file, or pre-commit failing to download
a toolchain. That looks exactly like "secret caught" if you only check the
exit code. The opposite also happens: the drill passes because the hook
never ran, for example after a wrong `pre-commit install` flag.

A real pass needs three things: the output names **gitleaks** and a rule
(`RuleID: aws-access-token`); a clean commit right after **succeeds**; and
the fake key is **random**. Gitleaks deliberately ignores AWS's
documentation key `AKIAIOSFODNN7EXAMPLE`, so a drill using it gets
committed.

**In the Podinfo run:** the first drill was rejected three times, every time
for the wrong reason (an SSL error while pre-commit downloaded Go, and a
branch without a config). The hooks had never been installed. RCA
`platform/docs/rca/2026-10-15-p1-4-hook-drill-false-pass.md`.
</details>

---

## P1-5 — Run it locally first

### 10. "Why not straight to Kubernetes?"

> **Interviewer:** The app will run in Kubernetes anyway. Why waste time
> running it locally first?

<details>
<summary><b>Model answer</b></summary>

Because when it fails in Kubernetes, the list of suspects is huge: a code
bug, a missing env var, the database, the image, the Service, a network
policy, a missing secret, a probe. If I've already seen it work locally,
following the developers' own README, half of those are ruled out. It's a
**known-good baseline**.

Running it also teaches me what reading can't: which env vars it really
needs, how it fails without its dependencies, and whether the README is even
right. And I run the **CUJ** locally, because "it started" isn't the same as
"it works".
</details>

**Follow-ups**

<details>
<summary><b>10.1</b> "The README says the app reads <code>DATABASE_URL</code>. The code reads <code>DB_HOST</code>, <code>DB_USER</code> and <code>DB_PASSWORD</code>. What do you do?"</summary>

Believe the code, and record the real variables in the inventory, found by
searching the code. Then fix the README with a PR to the developers, so the
next person doesn't lose an hour. Every step where the README didn't match
reality gets fixed the same way.
</details>

---

## P1-6 — Understand it as an operator

### 11. "Before you write the manifests" ★

> **Interviewer:** You're handed a service you didn't write. Before you write
> a single Kubernetes manifest, what do you need to know about it, and how do
> you find out?

<details>
<summary><b>Model answer</b></summary>

First the four basics: its **entry point**, **dependency manager**, **build
tool** and **final output**. Then eight operator questions, each answered by
*doing*, not by reading the README:

| Question | How I find out | What it decides later |
|---|---|---|
| How does a request flow? | Follow one CUJ step through the code | Network policy, tracing |
| What does the health check check? | `curl` it, read the handler | Probe design |
| Logs: structured, to the console? | Run it and look | Log collection |
| Handles SIGTERM? | `time docker stop` | Grace period, `preStop` |
| Config from env vars? | Search the code | ConfigMaps, Secrets |
| Writes to local disk? | Run with a read-only filesystem | `readOnlyRootFilesystem`, `emptyDir` |
| Startup time? | Measure it | Startup probe |
| Memory under load? | `docker stats` during the CUJ | Requests and limits |

Each answer goes into `docs/inventory.md`, because each one becomes a
setting in Phase 6.

**In the Podinfo run:** entry point `cmd/podinfo/main.go`, Go modules,
`go build` via the `Makefile`, one static binary.

**Weak answer:** "The port and the image name."
</details>

**Follow-ups**

<details>
<summary><b>11.1</b> "<code>docker stop</code> on the container takes almost exactly 10 seconds. What does that tell you?"</summary>

That it **ignored SIGTERM** and was killed. Docker sends SIGTERM, waits 10
seconds by default, then sends SIGKILL. A well-behaved app stops in well
under that. In Kubernetes this means every deploy kills in-flight requests.
Common causes: the app doesn't handle the signal, or a shell wrapper in the
Dockerfile (`CMD` in shell form) runs as PID 1 and never passes the signal
on.
</details>

<details>
<summary><b>11.2 ★</b> "The app logs JSON, but to stderr, not stdout. Is that a problem?"</summary>

Not for collection: the container runtime captures both streams, and log
collectors read both. But write it down. Some tools and dashboards treat
stderr as "errors", and someone filtering on stdout will see nothing. The
detail that matters more: check whether the logger **samples**.

**In the Podinfo run:** zap is configured with `OutputPaths: stderr` and
sampling `Initial: 100, Thereafter: 100` (`cmd/podinfo/main.go`). After 100
identical messages in a second, it keeps only every 100th. During an
incident, log counts undercount what happened, so count with metrics, not
log lines.
</details>

<details>
<summary><b>11.3</b> "How do you find out whether an app writes to local disk, quickly?"</summary>

Run it with a read-only filesystem and use it: `docker run --read-only …`,
then drive the CUJ. Whatever it tries to write fails loudly, with the path in
the error. Then give it exactly those paths as small writable volumes
(`--tmpfs /tmp` locally, `emptyDir` in Kubernetes) and decide whether any of
that data matters if the pod dies.

**In the Podinfo run:** expect `/data` (the `/store` endpoint,
`--data-path`).
</details>

<details>
<summary><b>11.4</b> "A Java service takes 90 seconds to start. In Kubernetes it gets restarted over and over. Why, and what's the fix?"</summary>

The liveness probe starts checking before the app is up, fails, and
Kubernetes restarts it, forever. It looks like an app bug but it's a timing
bug. Fix: a **startup probe** that allows enough time (measured, plus
margin); liveness and readiness only begin after it passes. That's why
startup time is measured in Phase 1.
</details>

<details>
<summary><b>11.5</b> "<code>docker stats</code> shows a Java service at 1.8 GB. You planned a 1 GB memory limit. What do you check?"</summary>

How the JVM sizes its heap. Modern JVMs read the container's memory limit
and size the heap as a percentage of it (`-XX:MaxRAMPercentage`), but off
the shelf they may claim more than you expect, and heap isn't the only
memory (threads, metaspace, buffers). Measure under the CUJ load, set the
limit above the real peak, and set the JVM's percentage so heap plus
everything else fits. Otherwise the container gets OOMKilled.
</details>

<details>
<summary><b>11.6</b> ★ "The app stops 3 seconds after SIGTERM and exits 0. So shutdown is graceful, right?"</summary>

You've only proven it handles the signal **when idle**. Graceful means
in-flight requests finish. Check *when the shutdown timeout starts*: if
the clock starts at the signal and the app then sleeps to let the load
balancer catch up, the sleep eats the drain budget. Test it with a request
in flight that takes longer than the budget left after the sleep.

Also don't trust the exit code: many servers log "shutdown failed:
deadline exceeded" and still exit 0, so a cut request looks like a clean
stop to every restart metric.

**In the Podinfo run:** 5 s timeout created at SIGTERM, then a 3 s sleep,
so ~2 s for in-flight work. A request needing 6.5 s after the signal was
cut at 5.0 s, and the process exited 0 (`pkg/signals/shutdown.go:43,61`;
platform `docs/inventory.md`, P1-6).
</details>

---

## P1-7 — Linux fundamentals (optional)

### 12. "Run it twice"

> **Interviewer:** You wrote a script that sets up a Linux server. What does
> it mean for it to be idempotent, and how do you prove it is?

<details>
<summary><b>Model answer</b></summary>

Idempotent means running it again leaves the system in the same state, with
no extra changes and no errors. Each step checks before acting: "install the
package *if missing*", "create the folder *if absent*", "write the config
*if different*". Proof: run it twice, and the **second run reports zero
changes**. I'd also expect a root check, OS detection, logging, retries with
backoff for network steps, and a clean `shellcheck`.

**Weak answer:** "It doesn't fail the second time." Not failing isn't the
same as not changing things.
</details>

**Follow-ups**

<details>
<summary><b>12.1</b> "You're hardening SSH on a remote server. How do you avoid locking yourself out?"</summary>

Keep a **second SSH session open** while changing the config, validate the
config before restarting (`sshd -t`), then test a *new* login from a third
terminal before closing anything. If the new login fails, the open session
fixes it. And do this practice on a throwaway VM, never on a machine you
need.
</details>

<details>
<summary><b>12.2</b> "Your backup script runs every night without errors. Is your data safe?"</summary>

Not proven. A backup is only real once you've **restored** from it: the
archive opens, the checksum matches, the files you need are there, and you
know how long a restore takes. Test the restore, on a schedule.
</details>

<details>
<summary><b>12.3</b> "You rewrote your Bash setup script as Ansible. When is that worth it?"</summary>

When you manage more than a couple of machines, or the setup must be
repeated and reviewed by others. Ansible's modules are idempotent by
default, roles are reusable, and Molecule can test them. Bash is fine for
one small, simple job. The point of the exercise is to feel the difference,
not to pick a winner.
</details>

---

## The inner loop

### 13. "Eight minutes per change"

> **Interviewer:** A developer complains that every code change takes eight
> minutes to see in the Kubernetes dev environment: build the image, push,
> deploy, wait. How would you help?

<details>
<summary><b>Model answer</b></summary>

That's the **inner loop**: change → rebuild → see the result. Tools built
for it:

- **Skaffold** or **Tilt** (pick one): watch the code, rebuild and redeploy
  automatically. Tilt can also sync files straight into a running pod,
  skipping the image build.
- **mirrord**: run the one service on the laptop while it receives the
  cluster's traffic, config and DNS. No rebuild at all; best for debugging
  one service.
- **vCluster**: a virtual cluster per developer, so people stop stepping on
  each other. That's *where* the loop runs, not how fast it is.

I'd add them only once there are running workloads to iterate on (P3-9).
Before that, they automate a process nobody understands yet.
</details>

---

## How we work

### 14. "Prove it"

> **Interviewer:** How do you know your dev environment work is actually
> done, and not just 'works for me'?

<details>
<summary><b>Model answer</b></summary>

Three checks, each by doing rather than reading:

1. **Fresh workspace:** `git clone && devpod up .`, every tool at its pinned
   version, the CUJ runs locally, with **no other manual step**.
2. **A fake secret is rejected** at commit time.
3. **The inventory answers all eight operator questions** for every service,
   from measurements and the code, not from the README.

If any of those needs "oh, and you also have to…", that step isn't in a
committed file yet, so it doesn't exist.
</details>
