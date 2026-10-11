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
├── inventory.md          services, ports, datastores, dependencies, owners
├── cuj.md                the Critical User Journey (your universal test)
├── sizing-and-cost.md    measured or estimated resources, monthly cost
├── environments.md       which environments, how they're separated
├── threat-model.md       assets, entry points, top risks
├── nfr.md                SLO targets, RTO/RPO, compliance needs
├── adr/                  0001-*.md, 0002-*.md …
└── backlog.md            the tickets from this template, adapted
```

---

## P0-1 Get access and read everything

**Type:** Task

- [ ] Access to every repo, the issue tracker, the existing cloud account(s), any monitoring, and the docs wiki
- [ ] Read each service's README, `package.json` / `pom.xml` / `go.mod`, Dockerfiles (if any), CI files, and env-var references (`grep -rn "getenv\|process.env\|System.getenv"`)
- [ ] List the questions you couldn't answer from the code; book 30 minutes with the dev lead to go through them
- [ ] **If the dev team didn't document how to run it locally, ask.** Reference 01: *"you have every right to question the developer"*

**Worked example (Sock Shop):** the upstream deploy repo is **deprecated** and
its manifests use drifted `:latest` images. That's a finding, not a blocker:
record it as a constraint.

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

## P0-3 Define the Critical User Journey (CUJ)

**Type:** Story · **Output:** `docs/cuj.md`

The single end-to-end path that proves the system works. Every "verify" step
in later phases runs it; the load test (P10) automates it.

- [ ] Written as user steps, with the services each step touches
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
- [ ] Choose node size: **fewer, larger nodes** (reference 03), minimum 3 for spread. On EKS also check **max pods per node** (VPC CNI limit = ENIs × IPs, unless prefix delegation is on)

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
└── namespace ppd
```

- **Non-prod environments share one cluster**, separated by namespaces with quotas, RBAC per team, NetworkPolicy, and separate Argo CD Applications and gitops folders (`envs/dev`, `envs/qa`, `envs/ppd`)
- **Prod gets its own cluster**: a bad change, a noisy test or a compromised dev workload can't touch it
- **No always-on DR cluster.** Running a second prod 365 days a year for a rare event is rarely worth the cost. Instead, DR is **recreated on demand** from Terraform + the gitops repo + backups (P10-9), and that recreation is rehearsed. Only build a warm standby in a second region (P10-11) when the business's RTO is shorter than the time it takes to recreate. Write the decision down with the RTO it gives you
- `ppd` is what this template calls **staging**; pick one name and use it everywhere (P0-12)

- [ ] Environments named (`dev`, `staging`, `prod`, plus `qa` if you have a test team), each with a purpose (reference 03: staging must be *stable*, unlike dev)
- [ ] Cluster layout decided (default above) and the **DR approach written down**: on-demand rebuild (default) or warm standby, with the RTO each gives
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
| Environments | exactly `dev`, `staging`, `prod` (not `development`, `production`, `prd`) | | everywhere |
| DNS | `{service}.{env}.{domain}`, prod without `{env}` | `shop.dev.example.com`, `shop.example.com` | review |

- [ ] `docs/conventions.md` committed with this table, adapted
- [ ] Linked from every repo's `CONTRIBUTING.md`
- [ ] Each "enforced by" item turned on when its phase arrives

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
