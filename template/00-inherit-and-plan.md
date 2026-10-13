# Phase 0: Inherit & plan

**Goal:** before touching any infrastructure, know what the application is,
who owns which part, what matters most, what it needs, what it may cost, what
could go wrong, and write it all down where others can review it.

**Why:** every demo project jumps straight to "create the cluster". In a real
job, the cluster you create on day 1 is sized by guesswork, the namespaces
don't match the teams, nobody agreed to the budget, and the first incident
reveals a data-protection requirement nobody asked about. Reference 03's whole
point: *"designing Kubernetes is not a single-day activity"*.

**Output of this phase** (all committed to a `docs/` folder in the platform repo):

```
docs/
├── questions.md          P0-1 gaps: answered-by-code + open (why / how we answer)
├── inventory.md          services, ports, datastores, dependencies, owners
├── cuj.md                the Critical User Journey (your universal test)
├── sizing-and-cost.md    measured or estimated resources, monthly cost
├── environments.md       which environments, how they're separated
├── ownership.md          who owns what between devs and platform (P0-11b)
├── conventions.md        names, tags, labels (P0-12)
├── threat-model.md       assets, entry points, top risks
├── nfr.md                SLO targets, RTO/RPO, compliance needs
├── adr/                  0001-*.md, 0002-*.md …
└── backlog.md            the tickets from this template, adapted

and in each app repo: MANIFEST.md, the service manifest (P0-2b)
```

---

## P0-1 Get access and read everything

**Type:** Task · **Output:** `questions.md` (scratch in the project folder; move into `platform/docs/` when that repo exists at P0-2)

- [ ] Access to every repo, the issue tracker, the existing cloud account(s), any monitoring, and the docs wiki. **Get the code the right way first:** clone (company repo) or fork-then-clone (open source); the steps and the fork safety checklist are in [P1-0](01-dev-environment.md#p1-0-get-the-code-clone-fork-or-both). For a practice-app fork, **re-verify** remotes, `gh` default repo, Actions off, and workflow-token read-only before you trust the tree
- [ ] Read each service's README, dependency manifest (`go.mod` / `package.json` / `pom.xml` / `build.gradle*` / `Cargo.toml` / `requirements*.txt` / `pyproject.toml` / `Gemfile` / whatever the stack uses), Dockerfiles (if any), Helm/Kustomize/Timoni values (if any), CI files, and **all config/secret entry points** (quote globs in zsh). Search for whatever this stack actually uses, not only `getenv` — e.g. `os.Getenv` / `viper` / `flag.` (Go), `process.env` / `dotenv` (Node), `System.getenv` / `System.getProperty` / `@Value` / `application*.yml` / `.properties` (Java/Kotlin), `os.environ` / `getenv` / `pydantic` `BaseSettings` / `django.conf` (Python), `std::env` / `dotenvy` (Rust), `ENV[` / `Rails.application.credentials` (Ruby), plus hardcoded defaults (`SetDefault`, `default:`, `fallback`), mounted paths (`/var/run/secrets`, `config` volumes), and cloud SDKs that pull secrets at runtime
- [ ] Write **`questions.md`** with two tables (below). Do not stop at "I don't know" — every open row needs a *How we answer it*
- [ ] **Company repo:** book 30 minutes with the dev lead for the rows that need a human. **Open-source / practice app:** there is no dev lead — answer each row by **code**, **experiment** (later phase), or **ADR** (decision you own), and say which in the table
- [ ] **If the app doesn't document how to run it locally, ask** (or, for OSS, treat that as a finding and write the run steps yourself in P1). Reference 01: *"you have every right to question the developer"*

### What `questions.md` must contain

```markdown
# <app> — P0-1 questions

## Access / fork state (verified <date>)
| Check | Result |
|---|---|
| remotes / default repo / Actions / token / default branch | … |

## Already answered by the code
| Question | Answer | Source (file:line) |
|---|---|---|

## Open questions
| # | Question | Why it matters | How we answer it |
|---|---|---|---|
| 1 | … | which later phase/control breaks if we guess | code | experiment in P… | ADR | ask <person> |
```

**Why the "Why it matters" column exists:** a beginner lists gaps; a platform engineer lists gaps *that change the design*. Port mismatch → 502s (P9). Hardcoded secret → P5 shape. `USER name` not UID → PSA/Kyverno reject (P8). Drain shorter than ALB deregistration → 502s on every rollout (P6/P9).

**Worked example (Sock Shop):** the upstream deploy repo is **deprecated** and
its manifests use drifted `:latest` images. That's a finding, not a blocker:
record it as a constraint in the open-questions table.

### Optional: use an AI assistant as a second reader

You are still responsible for checking the answers against the code. The assistant's job is to **surface gaps you would not think to ask**, not to invent the inventory.

Copy-paste prompt (adapt the paths):

```text
We're on template P0-1. Do not create AWS resources or new repos.

1. Verify remotes / gh default repo / Actions / workflow-token permissions
   (fork safety from P1-0).
2. Walk README; the language dependency manifest(s); Dockerfile(s);
   Helm / Kustomize / Timoni values if present; and .github/workflows
   (or the CI this repo actually uses).
3. Find every config and secret entry point for THIS stack — do not stop
   at getenv. Cover, as applicable:
   - Go: os.Getenv, viper, flag, cobra, SetDefault
   - Node: process.env, dotenv, config packages
   - Java/Kotlin: System.getenv/getProperty, @Value, application*.yml/.properties
   - Python: os.environ, getenv, pydantic BaseSettings, django.conf
   - Rust: std::env, dotenvy/clap env
   - Ruby: ENV[], Rails credentials/secrets
   - Also: hardcoded defaults/fallbacks, files under /var/run/secrets or
     config volumes, and cloud SDKs that fetch secrets at runtime
   Quote shell globs in zsh.
4. Draft questions.md with: fork/access state; "already answered by the
   code" (with file:line); "open questions" each with Why it matters and
   How we answer it (code | experiment in Pn | ADR | ask a human).
5. Flag anything that contradicts this DevSecOps template (branch name
   main vs master, upstream Dockerfile vs "write your own", etc.).
```

Feed the result into P0-2 (inventory constraints) and P0-2b (manifest). Rows that are decisions become ADRs in P0-8.

---

## P0-2 Build the application inventory

**Type:** Story · **Output:** `docs/inventory.md`

- [ ] One row per component: name, language/runtime, port, health endpoint, datastore, synchronous callers, asynchronous (queue) links, owning team
- [ ] Known constraints: hardcoded credentials, missing health endpoints, no SIGTERM handling, EOL runtimes, licences
- [ ] A dependency diagram (who calls whom); this becomes your NetworkPolicy allow-list in P8

**Worked example:**

| Service | Lang | Port | Datastore | Calls | Owner |
|---|---|---|---|---|---|
| front-end | Node | 8079 | — | catalogue, carts, orders, user | web team |
| catalogue | Go | 80 | catalogue-db (MySQL 5.7) | — | catalog team |
| carts | Java | 80 | carts-db (MongoDB) | — | checkout team |
| orders | Java | 80 | orders-db (MongoDB) | carts, user, payment, shipping | checkout team |
| user | Go | 80 | user-db (MongoDB) | — | identity team |
| payment | Go | 80 | — | — | checkout team |
| shipping | Java | 80 | RabbitMQ (publish) | — | fulfilment team |
| queue-master | Java | 80 | RabbitMQ (consume) | — | fulfilment team |

Constraints: `catalogue-db` credentials baked into the image; MySQL 5.7 EOL;
some services ignore SIGTERM; no tracing instrumentation.

---

## P0-2b Discovery session and the service manifest

**Type:** Story · **Output:** one `MANIFEST.md` per service, **in the app repo** next to the code

**Why:** the inventory (P0-2) is what *you* can read from the code. The rest
you have to **ask the developers**, and the biggest beginner mistake is not
knowing what to ask, then accepting vague answers ("it's stateless", "port
3000, I think"). Every vague answer turns into a 502, a crash loop or dropped
requests later. You build the highway; developers build the cars. They owe you
structured answers; you owe them a safe, automated place to run.

### The question bank: ask what breaks if you don't

| Area | Ask | What it protects you from |
|---|---|---|
| Packaging | "Dockerfile, binary, or raw source?" · "Base image, multi-arch?" · "Exact runtime version you tested with?" | The wrong pipeline; no Graviton later; "works on my machine" |
| Network | "Which port, from the code, not from memory?" · "Public or internal only?" · "HTTP/1.1, HTTP/2, gRPC, TLS, raw TCP?" · "Does any client need a fixed IP or mutual TLS?" | Target-group mismatch (**the #1 cause of 502s**); over-exposure; choosing ALB when you needed NLB (P9) |
| **State** | "If I kill this container mid-request, what does the user lose?" · "Does it write **anything** to local disk: sessions, uploads, temp files, caches?" | Data lost on every scale-in. **Ask the disk question even after they say "stateless"**: developers sincerely believe it while writing uploads to `/tmp` |
| Startup / shutdown | "How long from start to serving traffic?" · "Does it trap SIGTERM and finish in-flight work?" · "How long does draining take?" | A 90 s cold start killed by a 30 s health check = an endless crash loop that looks like an app bug. No SIGTERM handling = dropped requests on **every** deploy |
| Data | "Engine and version?" · "Real customer data, or disposable?" · "**Does it pool connections, and how many at peak?**" · "What does it call, what calls it, and what happens when a dependency is down?" | "The DB fell over" that was really 500 unpooled connections hitting `max_connections`; retry storms; no blast-radius map |
| Scaling | "Steady or spiky, and what *causes* the spikes?" · "What signals overload: CPU, requests or queue depth?" · "**Can it process the same message twice safely?**" | Scaling on the wrong metric (P6-2); data corrupted by at-least-once redelivery |
| Secrets | "Which credentials, and does it reload them **without a restart**?" · "Are any in the repo right now?" · "How will you debug it in prod?" | Rotation causing an unplanned outage; an inherited leak; the bastion-and-static-keys request arriving as an emergency (offer SSM, P5-1) |

### Running the 30-minute session

1. **Scope (5 min):** "I want the infrastructure to match exactly what your code needs."
2. **App (10 min):** packaging, port, protocol, state, startup and shutdown.
3. **Data (10 min):** datastore, criticality, connections, secrets, dependencies.
4. **Scale (5 min):** traffic shape, scaling signal, health path, owner.

**Rules that make it work:**
- Never ask "just tell me what you want". Give an **either/or with the consequence attached**: *"single-AZ to save money, or Multi-AZ because it's real customer data?"*
- Test statelessness by **behaviour**, not definition.
- "I don't know" is a valid answer. Set conservative bounds, instrument it, and book a revisit in two weeks. Then ask "what would *cause* a spike?"
- Under deadline pressure, ask only the **four load-bearing questions** (port and protocol, disk, SIGTERM, which database and is the data real). Write down the deferred risk as a shared decision.
- Write the answers into the manifest **during** the meeting.

### The service manifest (copy into each app repo)

```markdown
# SERVICE MANIFEST: <service>
## 1. Identity       runtime+version · artifact (Dockerfile/binary) · multi-arch?
## 2. Network        public|internal · port · protocol · static IP / mTLS needed?
## 3. Compute        CPU/RAM · stateless|stateful · min/max replicas · scaling signal
## 4. Data           engine+version · single|Multi-AZ · peak connections, pooled? · buckets · queues · idempotent?
## 5. Observability  health path (one that can FAIL) · logs to stdout? structured? · traced?
## 6. Config         key | plain|secret | Secrets Manager path · reloads secrets without restart?
## 7. Start/stop     cold start (s) · traps SIGTERM? · drain time (s)
## 8. Dependencies   calls · called by · on dependency failure: retry | circuit-break | fail fast
## 9. Ownership      dev on-call · team channel · runbook link · manifest owner
```

Sections 7–9 are the ones usually missing, and they're the ones you need at
3 a.m. **During an incident the manifest is your diff:** "it says HTTP/2 over
TLS on 8443; the target group says HTTP" is a 5-minute fix instead of an hour
of guessing. That's why **who keeps it current** matters more than how
complete it is.

- [ ] A manifest per service, filled in **with** a developer, committed in the app repo
- [ ] Every "I think" replaced by a value checked in the code
- [ ] Known gaps (no SIGTERM, health check on `/`, unpooled connections) turned into tickets for the dev team
- [ ] A manifest owner named per service

<details>
<summary><b>Exercise: spot the gaps in this manifest</b></summary>

`inventory-sync` · Production · Python 3.11 · internal · port 8000 · HTTP/1.1 ·
0.5 vCPU / 1 GB · "stateless" · min 1 / max 10 on CPU > 70 % · PostgreSQL 15
single instance · health check `/` · logs to stdout · `DB_PASSWORD` in Secrets Manager.

**Check yourself:**
1. Sections 7–9 are missing entirely: no cold start, SIGTERM, drain, dependencies or on-call. That is the most serious gap.
2. `min 1` in prod: every deploy is a brief outage and there's no AZ redundancy.
3. A health check on `/` returns 200 while the database is dead. A check that can't fail does nothing.
4. A single-instance database contradicts "Production". That's a business decision disguised as a field.
5. Nothing about connection pooling, and up to 10 replicas.
6. CPU may be the wrong signal for a "sync" job. It is probably queue- or schedule-driven.
7. Does it reload `DB_PASSWORD` on rotation? And is it idempotent?
</details>

<details>
<summary><b>Exercise: manifest vs live config (aim for under a minute each)</b></summary>

| Manifest says | Live config | Symptom |
|---|---|---|
| HTTP/2 over TLS, 8443 | Target group: HTTP, 8443 | **502**: unencrypted bytes hit a TLS socket; pods look healthy |
| Cold start 75 s | Health check fails after 30 s | **Endless crash loop** that looks like an app bug |
| Drain 30 s | Deregistration delay 5 s | **Dropped requests on every deploy** |
| Read-write one bucket | Role allows `s3:*` | No symptom: an audit finding and a wider blast radius |
| Internal only | SG allows `0.0.0.0/0` | Reachable from far more than intended; use SG-as-source (P4-3) |
| Scale on queue depth | HPA on CPU, min 1 | Scales late or never; never reaches zero (KEDA, P6-2) |
</details>

---

## P0-3 Define the Critical User Journey (CUJ)

**Type:** Story · **Output:** `docs/cuj.md`

The single end-to-end path that proves the system works. Every "verify" step
in later phases runs it; the load test (P10) automates it.

- [ ] Written as user steps, with the services each step touches
- [ ] **At least one step proves the app serves requests, not just that it's
      healthy.** Health endpoints are often exempt from the app's own failure
      paths (fault injection, middleware, dependency calls), so a sick pod can
      answer 200 on `/healthz` and 500 on every real path. Assert response
      bodies, not only status codes
- [ ] **One step checks build identity**: a version/commit endpoint (or
      header) whose value must equal what CI built. Without it, "the CUJ
      passed" can't tell the new release from the old one still running.
      Watch for defaults like `commit: "unknown"`, which still return 200
- [ ] **A negative step for every secret that has a default in the code**
      (JWT signing key, admin password, API key). For example, a token signed
      with the upstream default must be rejected. The happy path passes with
      the default as well, so only a negative step catches it. Add the step once P5
      wires the real secret
- [ ] A scripted version (curl / Playwright / k6) planned for P2

**Worked example:** browse catalogue → register / log in → add to cart →
checkout → order persisted → shipping task enqueued → queue consumed.

---

## P0-4 Classify by criticality and plan the rollout order

**Type:** Story

From reference 03: don't move everything at once.

- [ ] Each service tagged **critical / important / medium / low** by business impact (ask the product owner, not engineering)
- [ ] Rollout waves: **low first**, critical last. For a greenfield build, this decides which service you containerise and harden first as the pattern-setter
- [ ] Pick a **pilot sample** spanning tiers and including at least one stateful component

**Worked example:** pilot = `catalogue` (Go, simple) + `carts` (Java, slow start, MongoDB) + `rabbitmq`. Java goes early on purpose: its slow start teaches the `startupProbe` lesson (P6).

---

## P0-5 Size from data, not guesses

**Type:** Story · **Output:** `docs/sizing-and-cost.md`

- [ ] **Existing system:** pull CPU / memory / disk per service from Prometheus, CloudWatch, or the VM bill (reference 03)
- [ ] **Greenfield:** run the Compose stack in P2 under the CUJ load and record `docker stats`; revisit this ticket then
- [ ] Sum requests per environment, add headroom (×1.3), add the platform overhead (observability, Argo CD, Kyverno, Falco ≈ 2–3 GiB)
- [ ] Choose node size: **fewer, larger nodes** (reference 03), minimum 3 for spread. On EKS also check **max pods per node**: with the VPC CNI and no prefix delegation it is `ENIs × (IPv4 per ENI − 1) + 2` (`t3.medium` = 17, `t3.large` = 35). Subtract the DaemonSet pods per node, then compare with the platform's Deployment pods (Argo CD alone is ~7). On a small cluster the **pod count** often runs out before memory: pods sit `Pending` with "Too many pods" while memory is free

**Worked example:** four Java services make `t3.medium` (4 GiB) too small once
observability is co-resident; `t3.large` (8 GiB) ×3 is the practical dev floor.

---

## P0-6 Decide the environment strategy

**Type:** Decision (ADR) · **Output:** `docs/environments.md`

| Option | Pros | Cons | Use when |
|---|---|---|---|
| One cluster, namespace per env | Cheapest | Shared blast radius; strict RBAC needed; noisy neighbours | Learning, tiny teams |
| **Separate prod cluster**, shared lower envs | Isolated prod | 2× control plane | **Default for this template** |
| Separate **accounts** per env | Strongest isolation; IAM boundary | Most setup | Real companies |

**How many environments, and how many clusters?** Companies use different
names for the same ideas (reference 04, part 8, describes five):

| Environment | Purpose | Other names | Stable? |
|---|---|---|---|
| **dev** | Developers try changes; breaks often, by design | development, sandbox | No |
| **qa** | Testers verify features and log bugs (bug-fix branches loop back here) | test, SIT | Mostly |
| **ppd** | Pre-production: a copy of prod (sizes, config, data shape) to reproduce prod issues and rehearse releases | staging, pre-prod, UAT | **Yes**, nobody experiments here |
| **prod** | Real users | production, live | Yes |
| **dr** | Disaster recovery: takes over if prod's cluster or region is lost | standby, failover | Built on demand |

**Five environments doesn't mean five clusters.** Reference 04's layout, which
this template adopts:

```
Cluster 1: non-prod          Cluster 2: prod
├── namespace dev            └── namespace prod (or one namespace per team)
├── namespace qa
└── namespace staging      (= ppd)
```

- **Non-prod environments share one cluster**, separated by namespaces with quotas, RBAC per team, NetworkPolicy, and separate Argo CD Applications and gitops folders (`envs/dev`, `envs/qa`, `envs/staging`)
- **Prod gets its own cluster**: a bad change, a noisy test or a compromised dev workload can't touch it
- **No always-on DR cluster.** Running a second prod 365 days a year for a rare event is rarely worth the cost. Instead, DR is **recreated on demand** from Terraform + the gitops repo + backups (P10-9), and that recreation is rehearsed. Only build a warm standby in a second region (P10-11) when the business's RTO is shorter than the time it takes to recreate. Write the decision down with the RTO it gives you
- `ppd` is what this template calls **staging**; pick one name and use it everywhere (P0-12)

- [ ] Environments named (`dev`, `staging`, `prod`, plus `qa` if you have a test team), each with a purpose (reference 03: staging must be *stable*, unlike dev)
- [ ] Cluster layout decided (default above) and the **DR approach written down**: on-demand rebuild (default) or warm standby, with the RTO each gives

**One AWS account or several?** This template starts with **one account**
(a house with locked rooms: the locks are your IAM policies, names and tags).
Real companies usually give each environment its **own account** (separate
houses: nothing crosses unless you build a door on purpose). Learn in one
account first; move to several once you've done a few projects. Everything
that changes is written down in [Phase 12](12-multi-account.md).

- [ ] ADR: single account for now, with the trigger that would make you move (real customer data, a second team, compliance)
- [ ] Promotion rule: **the same image digest** moves dev → staging → prod; nothing is rebuilt (reference 04's retag idea, done correctly)
- [ ] Branching model chosen: **trunk-based** (short feature branches → `main`, environments via GitOps folders) is the default; reference 04's `feature → qa → main` works but duplicates pipelines
- [ ] ADR written

> For learning on a budget: build `dev` end to end; define `prod` values in the spec and build it once as a final exercise (reference 07's convention).

---

## P0-7 Set the budget and cost guardrails

**Type:** Task

- [ ] AWS Budget with email alerts at 50 %, 80 %, 100 % of your monthly limit **before creating anything**
- [ ] Cost model per phase in `sizing-and-cost.md` (EKS control plane, nodes, NAT gateways, ALB, interface endpoints, observability storage)
- [ ] Teardown discipline: every infra phase ends with `terraform destroy` (or a stop schedule for dev)
- [ ] Decide the cost-driven trade-offs explicitly, as ADRs (e.g. *no NAT gateway, public subnets, compensated by tight SGs*: reference 07 ADR-0005)

| Cost trap | Typical | Mitigation |
|---|---|---|
| NAT gateway per AZ | ~$100/mo for 3 | Single NAT for dev, or public subnets + strict SGs (ADR), or destroy |
| Interface VPC endpoints | ~$7/mo **per AZ each** | Gateway endpoints (S3/DynamoDB) are free; interface only when needed |
| Idle EKS | ~$73/mo control plane | Destroy between sessions |
| Observability storage | grows silently | Retention limits from day one |
| EKS **extended support** | control plane **$0.60/h** (6×) once the Kubernetes version leaves standard support | Create clusters on a version in standard support; check the EKS version calendar each run |
| T3/T4g **unlimited** credits | load tests above baseline CPU are billed as surplus credits | Short load tests, `standard` credit mode, or non-burstable nodes for load runs |

---

## P0-8 Start the ADR log

**Type:** Task · **Output:** `docs/adr/`

An **Architecture Decision Record** is a short file per decision. It's what
lets you answer "why did you do it this way?" in a review or an interview.

```markdown
# ADR-0005: Public subnets without a NAT gateway (dev only)

Status: Accepted · Date: 2026-10-11

## Context
NAT gateways cost ~$32/month each; dev runs a few hours a week.

## Decision
Dev uses public subnets with no NAT. Security groups are the boundary;
EKS public endpoint restricted to my IP; no SSH; IMDSv2 hop limit 1.

## Consequences
+ Saves ~$100/month.
− Nodes have public IPs; one misconfigured SG exposes them.
Prod uses private subnets + NAT (ADR-0011).

## When we'd revisit
Any real data in dev, or a second team member.
```

- [ ] First ADRs written: application choice, environment strategy, network design, secrets approach, CI/CD split

---

## P0-9 Threat model (one page)

**Type:** Story · **Output:** `docs/threat-model.md`

- [ ] **Assets:** customer data (users, orders), credentials, the CI pipeline, the container registry, Terraform state
- [ ] **Entry points:** public ALB, GitHub (PRs, Actions), the registry, cluster API, your laptop
- [ ] **Top risks** using STRIDE, each mapped to the phase that mitigates it:

| Risk | Example | Mitigated in |
|---|---|---|
| Spoofing | Stolen CI credential deploys malware | P5 (OIDC), P7 (signing), P8 (verify) |
| Tampering | Image swapped in registry | P4 (immutable tags), P7/P8 (Cosign + Kyverno) |
| Repudiation | Who changed prod? | P8 (GitOps = audit trail), P4 (CloudTrail) |
| Information disclosure | Secret in git / state / image layer | P1 (Gitleaks), P4 (state), P5 (secrets), P2 (layers) |
| Denial of service | Traffic spike, noisy neighbour | P6 (limits, HPA), P9 (WAF, rate limits) |
| Elevation of privilege | Container breakout, over-broad IAM | P2/P6 (non-root), P5 (least privilege), P8 (PSA), P10 (Falco) |

---

## P0-10 Non-functional requirements

**Type:** Story · **Output:** `docs/nfr.md`

- [ ] Draft **SLOs** per CUJ step (e.g. front-end availability 99.9 %, p95 < 500 ms; checkout success 99.5 %); finalised in P10
- [ ] **RTO / RPO** targets (e.g. RTO < 1 h, RPO < 24 h for cluster state, < 5 min for the order database)
- [ ] Compliance / data rules (PII in `user-db`? retention? region?)
- [ ] Supported browsers / clients, expected traffic (normal and peak)

---

## P0-11 Write the backlog and get sign-off

**Type:** Task

- [ ] Copy the tickets from phases 1–11 of this template into your tracker; delete what doesn't apply, add app-specific ones
- [ ] Mark optional tickets `[OPT]` so skipping them is a visible choice
- [ ] Share inventory + sizing + cost + environments + NFRs with the dev lead and manager. **Reference 03:** *"a lot of people prepare this document but keep it to themselves"*
- [ ] Agreement recorded (comment, email, or meeting note linked in `docs/`)

---

## P0-11b The ownership contract

**Type:** Task · **Output:** `docs/ownership.md`

**Why:** some failures have no technical cause. A developer assumes "tests
passed, so it's live"; you assume a human approves prod; a fix sits unshipped
for two days. Or a bad release runs at 100 % for 40 minutes because each side
thought the other would roll back. No Terraform fixes an unstated assumption.

| Developers own | Platform owns | **Agree explicitly, in writing** |
|---|---|---|
| Dockerfile / build definition | Pipeline automation | **Image tags:** git SHA (+ semver releases), **never `latest`** (P0-12) |
| Tests and what "passing" means | The deployment gate | **What blocks a deploy:** e.g. CRITICAL/HIGH findings block, MEDIUM warns; who may grant an override and for how long |
| Branch protection on their repo | Rollback mechanics | **Who approves prod:** auto on green to staging; prod needs a code-owner review (P7-5) |
| App-level error handling | Infrastructure alerting | **Who triggers rollback:** automatic on canary failure (P8-8); either side may `git revert` without asking |
| Versioning | Environment config | **Who owns each service manifest** (P0-2b) as the app changes |

- [ ] The five agreements written down, linked from each app repo's README
- [ ] **During incidents:** pages are routed **by layer, not seniority**: load balancer 5xx, node and pod failures page platform; app errors page the dev team; unclear ones page both. **One incident commander**; two people fixing in opposite directions is how incidents get worse (P10-8c)
- [ ] Console or `kubectl` changes made during an incident are reconciled back into Git within 24 hours
- [ ] Developers are told **before** a secret rotation if their service needs a restart to pick it up

---

## P0-12 Naming, tagging and labelling conventions

**Type:** Task · *Decide once, write it in `docs/conventions.md`, never debate it again*

**Why:** names are how you find things at 3 a.m., how costs get split, and
how policies match resources. If every engineer invents their own, IAM
policies need wildcards, cost reports are useless, and Kyverno can't tell
what's what. Pick a pattern now, before the first resource exists.

| Thing | Pattern | Example | Enforced by |
|---|---|---|---|
| AWS resources | `{project}-{env}-{component}` | `sockshop-dev-vpc`, `sockshop-prod-eks` | Terraform locals; review |
| AWS tags (every resource) | `Project`, `Environment`, `ManagedBy`, `Owner` (+ `Service` where per-service) | `Project=sockshop`, `Environment=dev`, `ManagedBy=terraform`, `Owner=platform-team` | provider `default_tags` (P4-1); Checkov custom check |
| IAM roles | `{project}-{env}-{who}` | `sockshop-dev-carts`, `sockshop-ci-build` | review |
| Secrets Manager paths | `{project}/{env}/{service}/{name}` | `sockshop/dev/catalogue/db` | IAM policies match the prefix (P5-5) |
| ECR repositories | `{project}/{service}` | `sockshop/carts` | Terraform `for_each` (P4-7) |
| Image tags | `{git-sha}`; releases `v{semver}`; **never `latest`** | `sockshop/carts:3f2a9c1…`, `v1.4.0` | ECR immutability; Kyverno (P8-6) |
| Kubernetes namespaces | `{team}` or `{app}`; platform ones by tool | `sock-shop`, `observability`, `argocd` | review |
| ServiceAccounts | `{service}` for the app; `{service}-{job}` for extra roles | `carts`, `carts-migrate` | Kyverno: no pod uses `default` (P5-4b) |
| Kubernetes labels | `app.kubernetes.io/{name,instance,component,part-of,managed-by}` | `name: carts`, `part-of: sockshop` | Kyverno `require-labels` (P8-6) |
| Terraform | modules `snake_case` folders; resources named by role (`this`, `main`), not by env | `module "vpc"`, `aws_eks_cluster.this` | `tflint` |
| Git branches | `{type}/{ticket}-{short}` | `feat/P5-3-github-envs`, `fix/P3-8-prestop` | branch-name check (optional) |
| Commits | **Conventional Commits** | `feat(carts): add startupProbe` | commitlint pre-commit hook (optional) |
| Environments | exactly `dev`, `qa` (only if you have a test team), `staging`, `prod`; not `development`, `production`, `prd`, and pick `staging` **or** `ppd`, never both | | everywhere |
| DNS | `{service}.{env}.{domain}`, prod without `{env}` | `shop.dev.example.com`, `shop.example.com` | review |

- [ ] `docs/conventions.md` committed with this table, adapted
- [ ] Linked from every repo's `CONTRIBUTING.md`
- [ ] Each "enforced by" item turned on when its phase arrives

---

## P0-13 Why EKS? The compute decision

**Type:** Decision (ADR: "why EKS") · *Do this once you have the inventory (P0-2) and manifests (P0-2b)*

This template is EKS-only, on purpose. But "why Kubernetes?" is one of the
most common interview questions, and what it really asks is **"do you know
when *not* to use it?"** In many real AWS shops the right answer is ECS or
Lambda. Know the framework even though you'll build on EKS.

| | **Lambda** | **ECS + Fargate** | **EKS** |
|---|---|---|---|
| You manage | Code | Task definitions | Cluster add-ons, node lifecycle, upgrades every few months |
| Fits | Event-driven work, glue between AWS services, spiky APIs, scheduled jobs | Long-running services, one team, roughly 5–20 services, no multi-tenancy need | You need the ecosystem (GitOps, admission policy, operators, mesh), several teams on shared infrastructure, or portability |
| Hard limits | 15 min per run, 10 GB memory, cold starts, no persistent connections | No Helm, Argo CD, Kyverno, CRDs or operators; isolation means separate accounts | ~$73/month control plane per cluster plus a real Day-2 workload ([DAY-2-OPERATIONS.md](DAY-2-OPERATIONS.md)) |
| Cost shape | Free when idle; **expensive at sustained high throughput** | Per task-second; no control-plane fee | Cheapest per pod at good node utilisation |

**Walk down the list and stop at the first honest "yes":**

1. Event-driven, stateless, under 15 minutes? → **Lambda.** Don't containerise it out of habit.
2. Under ~20 services, one team, no multi-tenancy, no need for the Kubernetes ecosystem? → **ECS + Fargate.**
3. Need GitOps, admission policy, operators or a mesh, several isolated teams, or portability? → **EKS.**
4. Choosing Kubernetes because it's on job descriptions? That's a valid **learning** reason (it's this template's reason). It isn't an **architecture** reason. Never mix the two up in an interview.

**Real systems mix them:** EKS for the core services, Lambda for event glue
and scheduled jobs (like the auto-remediation in P10-5c), and Fargate for a
few isolated workloads. "Which platform" is a per-workload question.

**What usually decides the shape** (from six common company types):

| Situation | Typical answer |
|---|---|
| Seed startup: 3 services, 2 engineers | ECS/Fargate or Lambda; CloudWatch; no GitOps, mesh or policy engine yet |
| Growing SaaS: ~15 services, 3 teams | EKS, prod + non-prod clusters, everything managed: **this template's shape** |
| Payments (PCI) | EKS; the card-data service gets **its own cluster and account** however small; NLB for partner banks (P9) |
| Healthcare (HIPAA) | EKS or ECS (the controls matter more than the orchestrator); encryption everywhere; no PHI in logs |
| Enterprise: 50+ services | Clusters per domain, hub-spoke GitOps, policy as default, golden paths |
| Global consumer app | One cluster per region (forced by data residency), warm standby before active-active |

The pattern: **compliance decides topology, client requirements decide the
edge, and what you leave out is the real decision.**

- [ ] The "why EKS" ADR written: the options, the choice, **the case against your own choice**, the ongoing cost (upgrades, patching, on-call), and the **reversal cost** (images are portable; manifests → task definitions is weeks, not a rewrite)
- [ ] The ADR names what would change your mind

---

## Prove it

- [ ] A person who hasn't seen the project can read `docs/` in 20 minutes and explain: what the app does, what you'll build first, what it costs, and the top 3 risks
- [ ] Every later-phase decision you make links back to an ADR or a requirement here

## Common mistakes

| Mistake | Seen in |
|---|---|
| Creating a cluster before knowing the workload | 01, 02, 04, 05, 06 |
| Sizing by guess; one huge cluster "to be safe" | — (03 warns against it) |
| Namespaces that don't match team ownership (RBAC pain later) | 03 explains the fix |
| Cost discovered on the first bill | everyone except 06 |
| Plan kept private, not agreed | 03 |
