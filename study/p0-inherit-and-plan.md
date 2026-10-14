# Study bank — Phase 0: Inherit & plan

Interview practice for [Phase 0](../template/00-inherit-and-plan.md). Each
**case** is how a scenario interview actually runs: the interviewer sets a
scene and asks an open question, then pushes with follow-ups to see how deep
your understanding goes.

**New to Phase 0?** Read [Phase 0 explained simply](p0-explained.md) first,
then come back here to practise.

**How to use it:** treat it as a mock interview. Read the scene, answer out
loud for about two minutes, *then* open the model answer. Do the follow-ups
the same way. There is rarely one right answer. The model answer shows the
reasoning a strong candidate gives, and **Weak answer** shows what makes an
interviewer stop listening.

**Evidence:** answers are general first. Where a run proved the point, an
**In the Podinfo run** note gives the evidence: the
[Podinfo fork](https://github.com/nazmur96/podinfo) read at `3e0ff8a`
(tag `6.15.0`), with file:line references into that repo. Paths like
`platform/docs/…` are that project's platform repo. Use them as interview
stories: "on a project I did, I found…" beats any definition.

`★` = came from a real finding or mistake in a run.

**Adding a question:** add a follow-up under the case it belongs to
(next number, e.g. `6.4`), or a new case at the end of its section's
numbering. Phrase it as something an interviewer would say, give the model
answer, and cite the file/line, ADR or RCA that proves it. Never renumber
existing cases or follow-ups: references like "study P0 6.2" must stay valid.

---

## Big picture

### 1. "Deploy it by Friday"

> **Interviewer:** You've just joined. On day one a team hands you an
> application and says "put it on Kubernetes by Friday." What do you do in
> your first week?

<details>
<summary><b>Model answer</b></summary>

I don't create a cluster first. I spend the first days finding out what I'm
running: read the code (ports, config, secrets, dependencies), sit down with
the developers for the things the code can't tell me (state, start-up and
shut-down behaviour, data), and write down how I'll prove it works (a
Critical User Journey). Then I size it from data, decide the environments,
set a budget, threat-model it, and get the plan agreed.

The reason: a cluster created first is sized, networked and secured by
guesswork, and every later phase inherits those guesses. Wrong port → 502s.
Wrong size → OOM kills or waste. No threat model → debug endpoints on the
internet. A document is cheap to change; a VPC and EKS layout isn't. If
Friday is fixed, I'd say what I can safely ship by then (dev only, with the
known risks written down) rather than skip the questions.

**Weak answer:** "I'd spin up EKS with Terraform and get the app running,
then fix things as they come up."
</details>

**Follow-ups**

<details>
<summary><b>1.1</b> "Isn't all this documentation just overhead? Real engineers deploy and fix what breaks."</summary>

Each document prevents a specific, expensive failure. The service manifest
prevents port and drain-time mismatches that show up as 502s. The inventory
prevents "open everything so it works" network rules. The CUJ prevents
celebrating green pods that don't serve users. Fixing these after deploy
means debugging in production, under pressure, often by someone who didn't
build it. The docs are small: one table, one form per service, one script.
Hours now against incidents later.
</details>

<details>
<summary><b>1.2</b> "You mentioned an inventory, a service manifest and a CUJ. Aren't those the same thing?"</summary>

- **Inventory** (`platform/docs/inventory.md`): a map of the *whole system*:
  components, ports, who calls whom, constraints. It feeds NetworkPolicy and
  sizing.
- **Service manifest** (`MANIFEST.md` in the *app* repo): the contract for
  *one* service: port, protocol, state, start and stop, secrets, failure
  behaviour, owner. In an incident it's what you diff live config against.
- **CUJ** (`platform/docs/cuj.md`): the shortest scripted path that proves
  users are actually served. It runs after every deploy.

The manifest lives next to the code so it changes in the same PR as the code.
</details>

<details>
<summary><b>1.3</b> "How would you know your planning was good enough to start building?"</summary>

Two tests. First, someone who hasn't seen the project reads `docs/` in about
20 minutes and can explain what the app does, what gets built first, what it
costs, and the top three risks. Second, every decision made in later phases
can point back to an ADR or a requirement from the plan. If I keep making
decisions that link to nothing, the plan missed something.
</details>

---

## P0-1 — Get access and read everything

### 2. "A repo you've never seen" ★

> **Interviewer:** You get access to a repo you've never seen, in a language
> you don't use every day. How do you find out what it needs to run: ports,
> config, secrets?

<details>
<summary><b>Model answer</b></summary>

I read in a fixed order: README, the dependency manifest (`go.mod`,
`package.json`, `pom.xml`…), the Dockerfile, any Helm or Kustomize values, the
CI files. Then I hunt for every **config entry point** for that stack, not
just `getenv`:

- Go: `os.Getenv`, viper, flags
- Node: `process.env`, dotenv
- Java/Spring: `System.getenv`, `application*.yml`, `@Value`
- Python: `os.environ`, pydantic settings
- Rust: `std::env` · Ruby: `ENV[]`, Rails credentials

Then the things people miss: **hardcoded defaults** (often a secret), mounted
file paths, and cloud SDK clients (which imply credentials). Last, I read the
deploy config to see what is actually passed. The result is a list of every
key, marked plain or secret, with where it came from. Anything I can't answer
goes into `questions.md` with *why it matters* and *how I'll find out*.

**In the Podinfo run:** this found a hardcoded JWT secret,
`SetDefault("jwt-secret", …)` in `cmd/podinfo/main.go:85`.

**Weak answer:** "I'd ask the developers." Ask them, yes, but only after
reading, so you can tell a vague answer from a correct one.
</details>

**Follow-ups**

<details>
<summary><b>2.1</b> "The developer says 'it runs on port 3000, I think.' What do you do with that?"</summary>

Check the code, not memory. Look at the listen call or the flag default, then
what the deploy config actually passes, then any env-var override. Then
confirm by running it. What I write down is *the value and where it came
from*, not just a number. A wrong port is the most common cause of 502s
behind a load balancer.

**In the Podinfo run:** the default is `fs.Int("port", 9898, …)`
(`cmd/podinfo/main.go:34`), the chart passes
`--port={{ .Values.service.httpPort }}`, and `PODINFO_PORT` can override both.
</details>

<details>
<summary><b>2.2 ★</b> "The README, the Helm chart and the Kustomize files all give different CPU and memory numbers. Which do you trust?"</summary>

None of them, for sizing. They're defaults written for different purposes:
tiny test clusters, a demo. **Code** is the source of truth for *behaviour*
(ports, paths). **Measurement** is the source of truth for *size*. I record
the disagreement as a finding and measure the app under load later.

**In the Podinfo run:** the chart says `1m`/`16Mi`, Kustomize says
`100m`/`64Mi` going up to `2000m`/`512Mi`.
</details>

<details>
<summary><b>2.3</b> "You set a value in a ConfigMap, but the app is clearly using a different one. How?"</summary>

The app reads config from more than one place, and something with higher
precedence wins. Many config libraries go: explicit flag > env var > config
file > default. So a value in a mounted config file can be silently
overridden by an env var someone added to the Deployment. I'd find the
library's precedence order and check every source.

**In the Podinfo run:** viper reads flags, env vars `PODINFO_<FLAG>`
(`main.go:90-92`) and `config.yaml` under `--config-path` (`main.go:93-100`),
in that order of precedence.
</details>

<details>
<summary><b>2.4</b> "It's an open-source app. There's no dev lead to ask. Now what?"</summary>

Every open question gets answered by one of three routes, and I write down
which: **code** (read it, cite file:line), **experiment** (run it in a later
phase and observe), or **decision** (I choose and record it as an ADR). The
"How we answer it" column in `questions.md` exists for exactly this. A gap
with no route to an answer is a guess waiting to happen.
</details>

<details>
<summary><b>2.5 ★</b> "Your search for config across the Go files said 'no matches found' instantly, in a repo full of Go. What happened?"</summary>

The shell expanded the glob before the tool saw it. In zsh, an unquoted
`--include=*.go` is expanded by the shell, and zsh errors when the glob
matches nothing in the current directory, so grep never ran. Quote it:
`--include='*.go'`. Same trap with `?` in URLs (`gh api 'repos/…?recursive=1'`).
The general lesson: "no results" from a search you didn't verify is not
evidence.
</details>

### 3. "Forking an open-source app" ★

> **Interviewer:** You fork an open-source project to build a platform
> around it. Before your first commit, what do you lock down, and why?

<details>
<summary><b>Model answer</b></summary>

| Check | Prevents |
|---|---|
| Actions disabled on the fork | Upstream workflows running on *my* repo: burning minutes, failing on missing secrets, or trying to publish |
| Default workflow token = read-only | A workflow (once enabled) writing to the repo or packages without being granted it explicitly |
| `upstream` push URL set to `no-pushing-to-upstream` | Accidentally pushing to someone else's project |
| `gh repo set-default` = my fork | `gh pr create` silently targeting upstream |
| Default branch noted (`master` vs `main`) | Workflow triggers and OIDC trust policies written for `main` that never match |

And I verify each one by observing it (`gh api …/actions/permissions`), not
by trusting that I clicked the setting.
</details>

**Follow-ups**

<details>
<summary><b>3.1 ★</b> "The upstream release workflow is still in your fork. A teammate turns Actions on to 'test the pipeline quickly'. What can go wrong?"</summary>

Every workflow whose trigger matches will run: code I haven't reviewed,
with whatever permissions the token has. Upstream release workflows push
images and charts to the *upstream* owner's registries and sign with their
keys. On my fork they mostly fail for lack of credentials, but "mostly fails"
is not a safety control. Rule: delete or replace upstream workflows **in the
same commit** that enables Actions, and review every `on:` trigger first.

**In the Podinfo run:** `release.yml` pushes to stefanprodan's Docker Hub and
GHCR and signs with a `COSIGN_KEY` secret.
</details>

<details>
<summary><b>3.2</b> "Why would you check the licence of an app you're only deploying?"</summary>

The licence decides what you may do with a fork. Apache-2.0 or MIT: fork,
modify, redistribute, keep the notices. Copyleft licences (GPL, AGPL) can
oblige you to publish your changes, and AGPL triggers on *network use*, which
is exactly what running it as a service is. Better to find out in planning
than from legal. (Podinfo is Apache-2.0.)
</details>

<details>
<summary><b>3.3</b> "The platform needs a change in the app, say disabling debug endpoints. In a company you'd file a ticket. On a fork you <i>are</i> the dev team. How do you keep that from becoming a mess?"</summary>

Small patch, behind a flag that defaults to safe, with a test. List it in the
app repo so whoever syncs upstream knows what to re-apply, and ideally send
it upstream as a PR. Record *why* in an ADR. Then add a CUJ step that fails if
an upstream sync silently brings the endpoint back. The discipline is the
same in a company; the ticket just goes to someone else.
</details>

---

## P0-2 — Application inventory

### 4. "What does it talk to?" ★

> **Interviewer:** You're about to write default-deny network policies for a
> service you didn't write. How do you find every outbound dependency it has?

<details>
<summary><b>Model answer</b></summary>

Three sources, because each tells you something different:

1. **Code** tells you what's *possible*: HTTP clients, gRPC dials, database
   and cache drivers, cloud SDKs, and env vars named `*_URL`, `*_HOST`,
   `*_ENDPOINT`.
2. **Config** tells you what's *switched on*: what the deploy actually sets.
3. **Traffic** tells you what's *actually happening*: run it under
   default-deny egress and watch what breaks, or watch flows (VPC flow logs,
   Hubble, conntrack).

You need all three. Code alone over-allows; traffic alone misses the rare
path that only runs at month-end.
</details>

**Follow-ups**

<details>
<summary><b>4.1 ★</b> "Your notes say the app only calls Redis. The code shows more. Why does that matter for network policy?"</summary>

Because dependencies can be **configuration-dependent**. If a call only
happens when a flag is set, the egress policy depends on the config. Turn on
tracing later without adding the egress rule and spans silently drop. So
config choices must be settled before network policy is written, and any
later config change ships its policy change in the same commit.

**In the Podinfo run:** three opt-in outbound calls: Redis
(`--cache-server`), a backend podinfo via `POST /echo` (`--backend-url`), and
an OTLP collector (`--otel-service-name`, endpoint from
`OTEL_EXPORTER_OTLP_ENDPOINT`). With no flags it calls nothing.
</details>

<details>
<summary><b>4.2</b> "Your security lead asks what that inventory table is actually <i>used</i> for. What do you tell them?"</summary>

Every column feeds a later decision:

| Column | Feeds |
|---|---|
| Port(s) | Service, probes, load balancer target group |
| Health endpoint | Probes, load balancer health checks |
| Datastore | Data plan, backups, secrets |
| Sync callers / callees | The NetworkPolicy allow-list |
| Async (queue) links | Queue infrastructure, idempotency, queue-based scaling |
| Owner | Who gets paged, who approves changes |
| Constraints | Tickets: hardcoded credentials, no SIGTERM handling, EOL runtimes, licences |

If a column feeds nothing, drop it.
</details>

---

## P0-2b — Discovery session and service manifest

### 5. "30 minutes with the developers"

> **Interviewer:** You've got 30 minutes with the lead developer before you
> design the platform for their service. What do you ask?

<details>
<summary><b>Model answer</b></summary>

I ask the questions where a vague answer turns into an outage later:

- **Network:** "Which port, *from the code*? Public or internal? HTTP/1.1,
  HTTP/2, gRPC, TLS?" (a port or protocol mismatch is the #1 cause of 502s)
- **State:** "If I kill the container mid-request, what does the user lose?
  Does it write *anything* to local disk?"
- **Start and stop:** "How long from start to serving? Does it handle
  SIGTERM and finish in-flight work? How long does draining take?"
- **Data:** "Which database and version? Is the data real? Does it pool
  connections, and how many at peak?"
- **Scaling:** "What causes spikes? What signals overload: CPU, requests,
  queue depth? Can it process the same message twice safely?"
- **Secrets:** "Which credentials, and does it reload them without a restart?"

How I ask matters as much: give either/or choices with the consequence
attached ("single-AZ to save money, or Multi-AZ because it's real customer
data?"), test claims by behaviour, and write answers into the manifest
*during* the meeting. If I only get five minutes, I ask the four that
everything else depends on: port and protocol, disk, SIGTERM, and which
database and whether the data is real.

**Weak answer:** "I'd ask them what they need." That gets you "it's
stateless, port 3000, I think."
</details>

**Follow-ups**

<details>
<summary><b>5.1 ★</b> "They say 'it's stateless.' Do you believe them?"</summary>

I test it by behaviour, not by definition. Two questions: *if I kill the pod
mid-request, what does the user lose?* and *does it write anything to disk?*
Developers sincerely believe "stateless" while writing uploads to `/tmp`.

**In the Podinfo run:** `POST /store` is always routed and writes to `/data`
(`store.go:35`). Kill the pod and the content is gone; with two replicas,
`GET /store/{hash}` works only on the replica that wrote it. Verdict: no
state *we rely on* (it's not in the CUJ), but disk writes exist, so: emptyDir,
no PVC, documented.
</details>

<details>
<summary><b>5.2</b> "What goes in your service manifest, and which part is usually missing?"</summary>

Nine sections: Identity, Network, Compute, Data, Observability, Config,
Start/stop, Dependencies, Ownership. The last three are the ones usually
missing, and they're exactly what you need at 3 a.m.: how long it takes to
start, whether it drains, what it calls, and who to page.
</details>

<details>
<summary><b>5.3</b> "Why does 'who keeps the manifest current' matter more than how complete it is?"</summary>

A stale manifest is worse than none: it sends the on-call engineer down the
wrong path with confidence. Completeness decays with every change. An owner,
plus the rule "update it in the same PR that changes ports, dependencies or
secrets", keeps it true.
</details>

<details>
<summary><b>5.4 ★</b> "The developer says the app reloads its secrets automatically because it has a file watcher. Do you write that down?"</summary>

Not until I've traced it. "Has a file watcher" isn't the same as "reloads the
secret it uses". Follow the value from where it's loaded to where it's used.

**In the Podinfo run:** an AI-drafted manifest claimed this. The code reads
the JWT key **once** at startup (`viper.Unmarshal`, `main.go:165`); the
`fscache` watcher only feeds `GET /configs` (`configs.go:11`), which the app
itself never uses. So rotating the key needs a restart.
</details>

### 6. "502s at 3 a.m., pods are green" ★

> **Interviewer:** It's 3 a.m. Users are getting 502s from the load balancer.
> Every pod is 1/1 Running and Ready. Walk me through it.

<details>
<summary><b>Model answer</b></summary>

A 502 with healthy pods usually means the load balancer and the pod disagree
about *how* to talk: wrong port, wrong protocol, or the pod went away while
the load balancer was still sending. So I compare the service manifest with
the live config, field by field:

- Target group port vs the port the app listens on
- Protocol: HTTPS to a plain-HTTP pod, or HTTP to a TLS socket
- Health check path: does it exist on the app?
- Did this start with a deploy? Then suspect draining (6.1).

The manifest turns a guessing session into a diff: "manifest says HTTP/1.1 on
9898, target group says 80" is a five-minute fix.

**Weak answer:** "Restart the pods." They're healthy; restarting changes
nothing and destroys evidence.
</details>

**Follow-ups**

<details>
<summary><b>6.1 ★</b> "The 502s only happen during deploys. Walk me through what happens when Kubernetes terminates a pod, and where requests drop."</summary>

Two things happen *in parallel*: the pod gets SIGTERM, and Kubernetes removes
it from the endpoints, after which the AWS Load Balancer Controller
deregisters the target. The deregistration can take several seconds. If the
process stops listening before the load balancer stops sending, those
requests get 502s.

Fixes: a `preStop` sleep longer than the propagation delay, pod readiness
gates, and `terminationGracePeriodSeconds` larger than preStop + shutdown.

**In the Podinfo run:** on SIGTERM podinfo marks itself unready, **sleeps
3 s**, then shuts down HTTP with a 5 s timeout (`signals/shutdown.go`). 3 s
can be shorter than deregistration. And the readiness probe (every 10 s)
won't even notice the unready flag in 3 s; endpoint removal is what actually
takes the pod out of service.
</details>

<details>
<summary><b>6.2 ★</b> "During the incident, someone switched the log level to debug to get more detail. Any problem?"</summary>

Possibly a new one: check what else the debug flag changes. **In the Podinfo
run:** the drain sleep is skipped at debug level (`shutdown.go`:
`if level != "debug"`), so every rollout after that drops in-flight requests:
more 502s, caused by the debugging. It went into the manifest as "never set
debug in prod".
</details>

<details>
<summary><b>6.3 ★</b> "Some requests return 504 after exactly 60 seconds. The service calls a slow backend. What's going on?"</summary>

The load balancer's idle timeout (60 s on an ALB) is shorter than the
service's client timeout. The ALB gives up and returns 504, but the service
keeps waiting on the backend, holding goroutines and connections for requests
nobody is waiting for. Under load those pile up. Timeouts should *shrink* as
you go deeper (edge > service > dependency), never grow.

**In the Podinfo run:** `/echo`'s backend client waits up to 2 minutes.
</details>

### 7. "Design the health checks"

> **Interviewer:** Liveness, readiness and startup probes. What does each one
> mean, and what would you put in each for this service?

<details>
<summary><b>Model answer</b></summary>

- **Liveness:** "restart me if I'm wedged." Checks the process only. Never
  dependencies, or a dependency outage restarts every pod at once.
- **Readiness:** "should I get traffic?" Checks what the pod needs to serve
  its *core* traffic. It must go false during shutdown so draining works.
- **Startup:** "give me time before liveness starts." Needed for slow
  starters: a 90-second Java app with a 30-second liveness check gets killed
  in an endless loop that looks like an app bug.

And every check must be able to **fail**: a health check on `/` that returns
200 while the database is dead does nothing.

**In the Podinfo run:** liveness on `/healthz`, readiness on `/readyz`; no
startup probe, because podinfo is ready about a second after start.
</details>

**Follow-ups**

<details>
<summary><b>7.1 ★</b> "Should readiness check the Redis cache?"</summary>

Usually not, if the cache only serves part of the app. If readiness checks
Redis, a Redis blip marks **every** replica unready at once and the load
balancer drops the whole service: a partial outage (cache endpoints fail)
becomes a total one. I'd write "readiness ignores Redis" into the manifest
as a known constraint so nobody assumes otherwise.

**In the Podinfo run:** Redis only serves `/cache/*`.
</details>

<details>
<summary><b>7.2 ★</b> "Traffic spikes, the service scales to 10 replicas, and the cache falls over. The pods stay Ready while users get 500s. What would you look for in the code?"</summary>

Connection pooling and failure handling. If the pool has no upper bound per
replica, a spike × 10 replicas can blow past the cache's `maxclients`. If
there's no retry with backoff or circuit breaker, every failure goes straight
to users as a 500 while the pods look fine. That's why the manifest asks
"pooled? peak connections?" and "what happens when a dependency is down?"

**In the Podinfo run:** the pool sets `MaxIdle: 3` and **no `MaxActive`**
(`cache.go:147`), so connections are unbounded.
</details>

---

## P0-3 — Critical User Journey

### 8. "Green dashboard, broken app" ★

> **Interviewer:** All pods are Running, every health check passes, Argo CD
> says Healthy. Users get 500 on everything. How is that possible, and how
> would your process have caught it?

<details>
<summary><b>Model answer</b></summary>

Health endpoints often don't go through the same code paths as real
requests: they're exempt from middleware, fault injection or dependency
calls. So a sick pod can answer 200 on `/healthz` and 500 on every real path,
and the platform keeps it in service.

That's why I define a **Critical User Journey**: the shortest end-to-end
path that proves users are actually served, run after every deploy. At least
one step must exercise real functionality, and it must check **response
bodies**, not just status codes. A CUJ made only of health checks would pass
here.

**In the Podinfo run:** fault injection exempts `/healthz`, `/readyz`,
`/metrics`, `/debug/` and `/fault_injection/*`
(`fault_injection.go:18-30`). The first CUJ draft would have passed on a
broken pod; it was caught and fixed
(RCA `2026-10-13-p0-3-cuj-checks.md`).
</details>

**Follow-ups**

<details>
<summary><b>8.1</b> "Why not just test every endpoint after each deploy?"</summary>

The CUJ runs after every deploy, sync and canary step, so it must be fast and
stable. Testing everything is slow, flaky and noisy, and some endpoints are
destructive drills that don't belong in a gate at all. Full coverage is the
test suite's job; the CUJ is the deploy gate.
</details>

<details>
<summary><b>8.2</b> "How do you know a check is actually any good?"</summary>

Ask: **what broken state would still pass this?** If you can name one, the
check is incomplete. Then break the system on purpose (enable fault
injection, deploy an old version, use the default secret) and confirm the
check goes red. A check that has never failed hasn't been tested.
</details>

<details>
<summary><b>8.3</b> "CUJ, smoke test, synthetic monitoring, load test: how do they relate?"</summary>

Same journey, different trigger and question. **Smoke test / CUJ** after a
deploy: "did this change break the core path?" **Synthetic monitoring** on a
schedule in prod: "is it working right now, from outside?" **Load test**:
"does it still work at N users, within the SLO?" Write the journey once and
reuse it in all three so they stay consistent.
</details>

### 9. "Which version is actually running?" ★

> **Interviewer:** Argo CD says Synced and Healthy, your CUJ passes, but
> users are still seeing the old release. How would you have known?

<details>
<summary><b>Model answer</b></summary>

One CUJ step must check **build identity**: a version endpoint or header
whose value must equal what CI just built (the commit SHA). And it must
assert the *value*, because a default like `commit: "unknown"` still returns
200. "Status 200" proves nothing about which build answered.

Typical causes when it doesn't match: a reused mutable tag (`latest`), the
image digest not updated in the gitops repo, or pods that never restarted.

**In the Podinfo run:** the binary defaults `commit` to `"unknown"`
(`version.go:4`).
</details>

**Follow-ups**

<details>
<summary><b>9.1 ★</b> "Your CUJ issues a token and validates it, and it passes. Does that prove the real secret is deployed?"</summary>

No. The round trip passes with *any* shared key, including a hardcoded
default. Only a **negative** step proves it: a token signed with the known
default must be **rejected**. General rule: for every secret that has a
default in the code, the CUJ needs a step that fails if the default is in
use.
</details>

<details>
<summary><b>9.2</b> "Your CUJ script fails now and then, never on the first run. It saves the token to a file and reuses it. Why?"</summary>

Tokens expire (podinfo's after one minute, `token.go`). Issue and validate
back to back in the same run. A CUJ must never depend on state left over from
the previous run.
</details>

<details>
<summary><b>9.3</b> "Your check on the home page passes. A colleague opens it in a browser and gets HTML, not JSON. Is your check wrong?"</summary>

The endpoint changes its answer depending on the client: podinfo's `/` sends
HTML to `Mozilla*` user agents and JSON to everything else
(`server.go:139-140`). A status-only check passes on both and proves little.
Script it with curl and assert JSON fields (`hostname`, `version`).
</details>

---

## P0-4 — Criticality and rollout order

### 10. "Thirty services to migrate"

> **Interviewer:** You're moving 30 services onto a new platform. Which do
> you move first, and why?

<details>
<summary><b>Model answer</b></summary>

The **least critical first, the most critical last**. The first services
moved are where my patterns are still wrong: the Dockerfile, the probes, the
policies, the pipeline. Mistakes there should hit something that hurts
little. By the time I move the payment service, those patterns have been
fixed on cheaper failures.

I'd also pick a **pilot** that spans tiers and includes at least one
stateful service and one slow starter, because those teach the lessons a
simple stateless service never will (backups, migrations, startup probes).

**Weak answer:** "The most important one first, to show value." That's where
your untested patterns do the most damage.
</details>

**Follow-ups**

<details>
<summary><b>10.1</b> "Who decides how critical each service is?"</summary>

The product owner, or whoever owns the business outcome, not engineering.
Criticality is *business impact*: revenue, users, legal exposure. Engineers
tend to rank by technical complexity. With no product owner (a learning
project), write down the proxy you used (Podinfo: "does it block the CUJ?")
so the rule is visible.
</details>

<details>
<summary><b>10.2 ★</b> "Your pilot rule says 'include a stateful service', but this app has none. What do you do?"</summary>

Deviate **explicitly**. Write down that there's no stateful pilot and why,
and list the lessons the pilot therefore *can't* teach (startup probes,
backups, PVCs, database migrations, connection pooling, multi-service network
policy) and where you'll learn each one instead. A silent deviation means you
believe you covered something you didn't.

**In the Podinfo run:** recorded in `platform/docs/rollout.md`; those lessons
move to the Retail Store project.
</details>

---

## P0-5 — Size from data

### 11. "How big should the cluster be?"

> **Interviewer:** It's a new app. There are no metrics yet. How do you
> decide the resource requests and the node size, in a way you can defend?

<details>
<summary><b>Model answer</b></summary>

Measure, don't guess. Run the app under the CUJ load (Docker Compose with
`docker stats`, then a local cluster with `kubectl top`), take a high
percentile, set requests near steady state and the memory limit above peak.
Then add it up in order: DaemonSets × nodes → platform pods (Argo CD,
observability, policy engine: about 2–3 GiB) → app × replicas, plus a canary
→ × 1.3 headroom. Prefer fewer, larger nodes, at least three for spread. On
EKS, check the **pod count** limit too, not just CPU and memory (11.1).

Upstream defaults are recorded as claims, never used as the answer.

**Weak answer:** "t3.medium is usually fine." Usually fine for what?
</details>

**Follow-ups**

<details>
<summary><b>11.1 ★</b> "Three t3.medium nodes, memory 50% used, but the new pod is Pending: 'Too many pods'. Explain it and fix it."</summary>

On EKS the VPC CNI gives each pod a real VPC IP, so pods per node are capped
by the instance's network interfaces: a `t3.medium` fits only 17. Subtract
about six DaemonSet pods per node and the cluster has roughly 33 slots, and
Argo CD (~7), Kyverno (~4), External Secrets (3), the load balancer
controller, CoreDNS, metrics-server and Karpenter fill them before the app
arrives. Fixes: larger nodes (`t3.large` = 35), VPC CNI **prefix delegation**
(about 110 pods per node), or fewer platform pods.
</details>

<details>
<summary><b>11.2</b> "Where does the 17 come from?"</summary>

`ENIs × (IPv4 addresses per ENI − 1) + 2`. Each network interface's primary
IP belongs to the interface itself, not to pods (−1). `aws-node` and
`kube-proxy` use host networking, so they count as pods without using an IP
(+2). `t3.medium`: 3 × (6 − 1) + 2 = 17. `t3.large`: 3 × (12 − 1) + 2 = 35.
(The template used to say "ENIs × IPs"; it was corrected during the Podinfo
run.)
</details>

<details>
<summary><b>11.3 ★</b> "Would you set a CPU limit on this Go service? A memory limit?"</summary>

**Memory limit: yes.** Memory can't be throttled, only killed, so the limit
protects the node and its neighbours. **CPU limit: usually no.** CPU limits
throttle the container even when the node is idle, which shows up as latency
spikes, and requests already guarantee its share. For Go, also set
`GOMEMLIMIT` to about 90% of the memory limit so the garbage collector works
hard *before* the OOM killer acts.
</details>

### 12. "The bill is double the estimate" ★

> **Interviewer:** Your manager forwards this month's AWS bill. It's twice
> what you estimated. Where do you look?

<details>
<summary><b>Model answer</b></summary>

Cost Explorer grouped by service, then by tag, and I check the usual traps:

- **NAT gateways** (about $32/month each, often one per AZ) and their data
  processing
- **Interface VPC endpoints** (about $7/month *per AZ, each*)
- **Idle EKS control planes** (about $73/month each)
- **Observability storage** with no retention limit
- **EKS extended support** (12.1) and **T3 unlimited credits** (12.2)
- **Resources Terraform doesn't know about**: load balancers and volumes
  created by Kubernetes controllers (see 18.3)

Then I fix the cause and add a guard so it can't happen silently again.
</details>

**Follow-ups**

<details>
<summary><b>12.1 ★</b> "It's the EKS control plane line. It's six times higher than expected, on an idle cluster. Why?"</summary>

The cluster runs a Kubernetes version that has left **standard support**, so
EKS charges **extended support**: $0.60/h instead of $0.10/h. Easy to miss
when you reuse old Terraform pinned to an old version. Fix: create clusters
on a version in standard support and check the EKS version calendar every
run.
</details>

<details>
<summary><b>12.2 ★</b> "After a two-hour load test, the EC2 line is higher than the hourly rate explains. Why?"</summary>

T3 instances are burstable and launch in **unlimited** mode by default.
Sustained CPU above the baseline (30% for `t3.large`) spends surplus credits,
billed per vCPU-hour. Keep load tests short, use `standard` credit mode
(throttles instead of billing), or run load tests on non-burstable nodes.
</details>

<details>
<summary><b>12.3</b> "Destroying dev every session works for you. What would a real team do instead?"</summary>

Scale down on a schedule (nodes to zero outside working hours, Karpenter
consolidation), right-size requests so bin-packing works, use Spot for
interruptible work and Savings Plans for the steady baseline, set retention
limits on observability, and give each account or team its own budget alerts.
A persistent environment is only worth keeping if recreating it is slow or
fragile. Measure both.
</details>

---

## P0-6 — Environment strategy

### 13. "How many environments, how many clusters?"

> **Interviewer:** We need dev, QA, staging and prod. How many clusters would
> you run, and why?

<details>
<summary><b>Model answer</b></summary>

Two: one **non-prod** cluster with dev, QA and staging as namespaces, and a
separate **prod** cluster. It's cost against blast radius. Non-prod
environments can share a control plane, separated by namespaces, quotas,
RBAC, NetworkPolicy and separate Argo CD apps, because an incident there hurts
no users. Prod gets its own cluster so a bad CRD, a noisy load test, a
cluster-wide add-on upgrade or a compromised dev workload can't touch it.
Namespaces isolate workloads; they don't isolate the control plane,
cluster-scoped resources or a node-level compromise.

For disaster recovery I'd default to **no always-on DR cluster**: rebuild on
demand from Terraform, the gitops repo and backups, and rehearse it. A warm
standby only makes sense when the business needs a faster recovery than a
rebuild gives.

**Weak answer:** "One cluster per environment, to be safe." Four control
planes, four upgrade cycles, and no reason given.
</details>

**Follow-ups**

<details>
<summary><b>13.1</b> "Prod's region goes down. You chose 'rebuild on demand'. What's your recovery time, and how do you know?"</summary>

Only a *measured* number counts: the time to `terraform apply` a new cluster
in another region, install add-ons, let Argo CD sync, restore data, and move
DNS. Typically hours. If you've never rehearsed it, you don't have a recovery
time, you have a guess. Rehearse it, write the result in the ADR, and if the
business needs less, that's the trigger for a warm standby.
</details>

<details>
<summary><b>13.2</b> "One AWS account or several? When would you split?"</summary>

One account is a house with locked rooms: IAM policies, names and tags are
the only walls, and one IAM mistake crosses them. Separate accounts per
environment are separate houses: nothing crosses unless you build a door (a
cross-account role). Start with one while learning; split when there's real
customer data, a second team, a compliance requirement, or a need to separate
billing and quotas.
</details>

<details>
<summary><b>13.3 ★</b> "Your naming standard says namespace <code>podinfo</code> in every environment. Dev and staging share the non-prod cluster. What goes wrong?"</summary>

Both environments sync into the same namespace: two Argo CD apps fight over
one Deployment, and staging is whatever dev wrote last. A naming rule depends
on the topology. One cluster per environment: `{app}` works. Shared cluster:
`{app}-{env}` (`podinfo-dev`, `podinfo-staging`). Check the conventions
against the cluster decision.
</details>

### 14. "Getting a release to prod"

> **Interviewer:** How does a release get from dev to prod in your setup?
> Do you rebuild the image for each environment?

<details>
<summary><b>Model answer</b></summary>

Build once, then promote the **same image digest** dev → staging → prod.
Never rebuild and never just retag. A rebuild produces a different artifact
from the one I tested: new base layers, new dependency versions, a new SBOM.
Tags are mutable pointers: `:1.4.0` today may not be `:1.4.0` tomorrow. A
digest is the content hash, so "what ran in staging" is provably "what runs
in prod", and the signature and attestations attach to the digest, which is
what admission control verifies. Environment differences live in config (the
gitops values), never in the image.

Promotion itself is a reviewed commit that changes the digest in the next
environment's folder.
</details>

**Follow-ups**

<details>
<summary><b>14.1</b> "Trunk-based development or long-lived branches per environment (dev → qa → main)?"</summary>

Trunk-based: short branches merge to one default branch, and environments are
folders in the gitops repo. Environment branches drift (a hotfix lands in one
and not another), merges between them become releases in their own right,
and each branch tends to grow its own pipeline. With trunk plus gitops
folders, "what's in staging" is one file diff, and promotion is a reviewed
commit.
</details>

<details>
<summary><b>14.2 ★</b> "Dev's deploy job can assume its AWS role. Prod's fails with 'Not authorized to perform sts:AssumeRoleWithWebIdentity'. The trust policy allows <code>environment:prod</code>. What's wrong?"</summary>

The GitHub Environment is probably named `production`, so the OIDC token's
`sub` claim says `environment:production` and doesn't match. Dev works
because its names agree. Debug by printing the token's claims in the job,
never by widening the trust policy to `*`. The real fix is one spelling of
each environment name everywhere.

**In the Podinfo run:** the template itself had this bug (one step said
`production`, the trust policy said `prod`); it was fixed during the run.
</details>

<details>
<summary><b>14.3</b> "Why does one misspelled environment name matter so much? Give me places it breaks."</summary>

- The OIDC trust policy's `environment:prod` claim (role assumption fails
  as AccessDenied)
- Secrets Manager prefix policies (`podinfo/prod/*`)
- Tag-based cost reports (two buckets for one environment)
- DNS and namespace names that Argo CD and NetworkPolicies reference

One spelling, everywhere, including GitHub Environments.
</details>

---

## P0-7 — Budget, guardrails and human access

### 15. "Day one in a fresh AWS account"

> **Interviewer:** You've been given a fresh AWS account for this project.
> Before you create a single resource, what do you set up?

<details>
<summary><b>Model answer</b></summary>

1. **Sign in the right way.** First look at what exists: how people reach the
   console today, existing profiles, IAM users and access keys. Use federated
   short-lived access (an existing IdP, or IAM Identity Center). Never create
   an access key for a human, and delete old ones I find.
2. **A budget with alerts** at 50%, 80% and 100% of the monthly limit, plus a
   *forecasted* 100% alert. List existing budgets first.
3. **A cost model per phase**: control plane, nodes, NAT, load balancer,
   endpoints, observability storage.
4. **Teardown discipline**: every session ends with `terraform destroy` (or a
   stop schedule), and cost-driven trade-offs are written down as ADRs.

**Weak answer:** "Create an IAM user with admin and an access key so
Terraform can run." That's the first long-lived credential of the project.
</details>

**Follow-ups**

<details>
<summary><b>15.1 ★</b> "You don't have SSO yet. Creating an access key 'just to set up the budget' is tempting. Why not?"</summary>

An access key is a long-lived credential. It sits in `~/.aws/credentials`,
leaks through dotfile backups and screenshots, and outlives its purpose. It
breaks the "no long-lived credentials" rule with the very first action.
Instead: pull the minimum of identity setup forward (one admin role via
federation or Identity Center), or create the budget in the console.
</details>

<details>
<summary><b>15.2 ★</b> "You were about to enable IAM Identity Center for SSO, then noticed the console shows you signed in through your company's IdP. Why stop?"</summary>

Federated, keyless access already exists: the IdP signs people in to an IAM
role, and `aws login` gives the CLI short-lived credentials. Identity Center
would be a **second owner of the same layer**: two places to grant and revoke
access, two to audit. It also creates an AWS Organization and makes this
account its management account. Look at what exists before adding anything.

**In the Podinfo run:** Authentik → `AuthentikAdmin` role. This became
ADR-0004, which superseded ADR-0003 before it was ever implemented.
</details>

<details>
<summary><b>15.3 ★</b> "You create a $40 budget. A $20 account-wide budget already exists. Problem?"</summary>

Both watch the same total, so the $20 alerts always fire first and the $40
one never tells you anything new. In a later review, two budgets also look
like two limits. Replace, don't stack: create the new one, verify its alerts
and subscribers, then delete the old one. Separate budgets only make sense
when they're *filtered* differently (per tag, service or account).
</details>

<details>
<summary><b>15.4 ★</b> "Your 100% budget alert arrived the morning after the session that overspent. Why so late, and what actually protects you?"</summary>

Billing data refreshes only a few times a day, so a budget alerts on spend
that has already happened, often hours later. It's a backstop that catches a
forgotten cluster on day 2 instead of day 30, not a brake. The real guard is
the **teardown checklist** at the end of every session, with a forecasted
alert and Cost Anomaly Detection for earlier warning.
</details>

### 16. "Leftover credentials" ★

> **Interviewer:** While auditing the account you find an IAM user called
> `github-actions-ci` with an active access key. What is it, and what do you
> do about it?

<details>
<summary><b>Model answer</b></summary>

It's a long-lived CI credential, probably stored as a GitHub secret by an
earlier project. Anyone who can read repo secrets or edit a workflow can use
it until it's rotated. The replacement is **GitHub OIDC**: the workflow
requests a short-lived token, and AWS lets it assume a role whose trust policy
names the exact repo and branch or environment.

Before deleting it: check when it was last used, find which repos hold it,
move them to OIDC, *then* delete the key (16.1).
</details>

**Follow-ups**

<details>
<summary><b>16.1 ★</b> "You deleted the key. What else do you have to do so nothing breaks confusingly later?"</summary>

Find every copy: Actions secrets across your repos (`gh secret list`, by
name), CI variables, `.env` files, laptops. Remove or replace them. Otherwise
a workflow fails weeks later with `InvalidClientTokenId` and nobody connects
it to the deletion. Then re-run `list-access-keys` for every user and verify
by observing, not by trusting exit code 0.

**In the Podinfo run:** the only remaining copy was in `devops-ai-playbook`.
</details>

<details>
<summary><b>16.2</b> "Your human access depends on a self-hosted identity provider. What happens when it's down?"</summary>

No federated sign-in, so no console and no CLI. Prepare a **break-glass**
path: the root user with hardware MFA and credentials stored offline, used
only when the IdP is down, with an alert on any root sign-in. Write the
procedure in the runbook and rehearse it once. Never "solve" it by keeping an
IAM user with an access key around just in case.
</details>

<details>
<summary><b>16.3</b> "Why would you put the identity provider on your threat model's list of assets?"</summary>

Because it's the front door. Whoever administers the IdP can grant themselves
the admin role in AWS, and federation means AWS will trust them. Protect it
like the root account: MFA for admins, few admins, logging, and a break-glass
path that doesn't involve a long-lived key.
</details>

### 17. "No NAT gateway in dev" ★

> **Interviewer:** To save money, your dev environment uses public subnets
> and no NAT gateway. Defend that decision. What are the risks?

<details>
<summary><b>Model answer</b></summary>

It saves about $100/month, and dev runs a few hours a week. The cost is that
nodes have public IPs, so one wrong security group rule exposes them. The
compensating controls: no inbound rules beyond the cluster's own, no SSH (SSM
only), the EKS API endpoint restricted to my IP, IMDSv2 with hop limit 1,
Checkov on the Terraform, and destroy after every session.

I'd also name the side effects: each public IPv4 is billed, dev's egress path
differs from prod's (which uses NAT), and add-ons that discover their region
or VPC from instance metadata need those set explicitly. Prod uses private
subnets and NAT. All of this is in an ADR with the trigger for revisiting
(real data in dev, a second team member).

**In the Podinfo run:** ADR-0002.
</details>

**Follow-ups**

<details>
<summary><b>17.1 ★</b> "Next morning, kubectl times out against the dev cluster. Nothing changed in Terraform. What's the likely cause, and what's the wrong fix?"</summary>

Your public IP changed (ISP, router restart, travel), and the API endpoint
only allows your old `/32`. Right fix: update the allowed CIDR and apply.
Wrong fix: `0.0.0.0/0`, which puts the Kubernetes API on the internet,
protected only by authentication.
</details>

### 18. "Rebuild every session"

> **Interviewer:** To save money you destroy dev every evening and rebuild it
> every morning. What breaks when you do that?

<details>
<summary><b>Model answer</b></summary>

Anything that doesn't really disappear on destroy, or that something outside
AWS points to. The usual three: names that stay reserved after deletion
(18.1), things referenced from outside AWS like DNS delegation (18.2), and
resources created by Kubernetes controllers that Terraform doesn't know about
(18.3). The general rule: keep anything slow or externally referenced in a
separate long-lived stack, and destroy only the per-session parts.
</details>

**Follow-ups**

<details>
<summary><b>18.1</b> "Morning apply fails: 'a secret with this name is already scheduled for deletion'. Why?"</summary>

Secrets Manager doesn't delete immediately: destroy schedules deletion with a
7–30 day recovery window, and the name stays reserved. For dev only, set
`recovery_window_in_days = 0` so destroy really deletes. Keep the window in
prod, where recovery matters more than reusing the name.
</details>

<details>
<summary><b>18.2</b> "After every rebuild, your app's DNS name stops resolving. Why?"</summary>

The hosted zone is in the per-session stack. Each new hosted zone gets new
name servers, and the registrar still delegates to the old ones. Keep the
hosted zone in a long-lived stack (about $0.50/month) and create and destroy
only the records.
</details>

<details>
<summary><b>18.3</b> "<code>terraform destroy</code> succeeded, but the next day's bill still shows a load balancer and EBS volumes. How?"</summary>

Kubernetes controllers created them, not Terraform. The AWS Load Balancer
Controller creates load balancers and security groups for Ingress, Gateway
and Service objects; the EBS CSI driver creates volumes for PVCs. They aren't
in Terraform state, so destroy doesn't know about them (and they can block
VPC deletion). Delete the Kubernetes objects first and let the controllers
clean up, then destroy, then check the console or bill the next day.
</details>

---

## P0-8 — Architecture decisions

### 19. "Why did you do it that way?"

> **Interviewer:** Six months from now, a new engineer asks why dev has no
> NAT gateway. Where do they find the answer?

<details>
<summary><b>Model answer</b></summary>

In an **Architecture Decision Record**: one short file per decision in
`docs/adr/`, with Context, Decision, Consequences (including the downsides),
and **when we'd revisit it**. It records the reasoning, not just the result,
which is what lets anyone, including me in an interview, answer "why?" with
something better than "the tutorial did it".

The first ones I write: the application choice, environment strategy,
network design, workload secrets approach, CI/CD split, and human access to
AWS.
</details>

**Follow-ups**

<details>
<summary><b>19.1</b> "An ADR you accepted turns out to be wrong an hour later. Do you edit it?"</summary>

No. I set its status to **Superseded by ADR-00NN** and write the new decision
as a new ADR that says what it replaces and why. The log is a history: the
reason you *didn't* do something is as useful in a review as the reason you
did. Only typos get fixed in place. (Podinfo: ADR-0003 → ADR-0004.)
</details>

<details>
<summary><b>19.2</b> "What goes in a 'secrets approach' ADR, and what doesn't?"</summary>

**In:** where workload secrets live (path convention), how they reach pods
(External Secrets, CSI driver, env var or file), who may read which paths
(IAM scope per environment and service), rotation (and whether the app needs
a restart), and how you prove it (CUJ negative steps). **Not in:** how
*humans* sign in to AWS; that's a separate identity decision. Mixing them
hides the fact that a perfect human SSO setup still delivers zero secrets to
pods.
</details>

<details>
<summary><b>19.3</b> "Why shouldn't CI have cluster credentials and run <code>kubectl apply</code>?"</summary>

Blast radius. CI runs third-party actions and dependency code on every PR. If
it holds cluster-admin, a compromised action owns production. With GitOps,
CI's worst case is pushing an image (which admission rejects if it isn't
signed) and committing a digest (reviewed, revertable). The cluster *pulls*
desired state. Bonus: the gitops history is a complete deploy log, and
rollback is `git revert`.
</details>

### 20. "Env var or mounted file?" ★

> **Interviewer:** The app needs a signing key. Do you inject it as an
> environment variable or mount it as a file?

<details>
<summary><b>Model answer</b></summary>

The textbook answer is "a file, because env vars leak into process listings,
crash dumps and child processes." But the real answer depends on **whether
this app exposes either one**. Some apps serve their environment over HTTP
(Spring's `/actuator/env` is the classic), and some serve their config
directory. So I check the app's routes first. If it exposes the path I'd
use, I fix the app (disable that endpoint), *then* inject, and add a CUJ step
proving the endpoint is gone. An edge rule alone isn't enough: in-cluster
callers can still reach it.

**In the Podinfo run:** *both* paths leak. `GET /env` returns
`os.Environ()`, and `GET /configs` returns every file in `--config-path`
(`pkg/fscache/fscache.go` → `configs.go`). The bank's earlier answer
recommended the file mount; that was wrong and caught at P0-8
(RCA `2026-10-13-p0-8-config-file-leak.md`). Decision, ADR-0006: patch our
build to disable `/env`, `/configs` and `/debug/pprof/`, mount the key as a
file, add an edge allow-list, and a CUJ step that checks they return 404.
</details>

**Follow-ups**

<details>
<summary><b>20.1 ★</b> "The app's source code is public anyway. Why is a hardcoded default secret a vulnerability?"</summary>

*Because* the code is public: everyone knows the key. Anyone can mint tokens
the app accepts. A default secret is a secret shared with the whole internet.
Fix: always set a real one, and add a CUJ negative step proving the default
is rejected (9.1).
</details>

<details>
<summary><b>20.2 ★</b> "So is that hardcoded key a critical vulnerability?"</summary>

Rate a risk by what it actually unlocks in *this* system, not by its category
name. In Podinfo, `POST /token` hands a valid token to *anyone*, so the JWT
guards nothing; the default key only adds the ability to forge tokens with
chosen claims and expiry. It's still worth fixing, because it's the practice
run for handling real secrets, and the habit has to be right before an app
where the token *is* the access control.
</details>

<details>
<summary><b>20.3 ★</b> "You rotate the key in Secrets Manager and the Kubernetes Secret updates. What happens to the running pods?"</summary>

Nothing, until they restart: env vars never change inside a running process,
and an app that reads a file once at startup won't re-read it either. You
need a rollout on change (Reloader, or a checksum annotation on the pod
template). **During** the rollout, old and new pods hold different keys, so
a token issued by a new pod fails on an old one: with two replicas, roughly
50% failures for the token's lifetime. Short token lifetimes limit the damage.
</details>

<details>
<summary><b>20.4</b> "You rotate the key mid-session and the CUJ's validate step fails. Is that an incident?"</summary>

Not yet. It's expected while old and new pods hold different keys, and it
should clear when the rollout finishes. That's why the ownership contract
says rotations are announced first. It becomes an incident if the step still
fails after every pod has restarted: then the new key didn't reach the pods
(External Secrets, the mount, or the path).
</details>

---

## P0-9 — Threat model

### 21. "Threat-model this service" ★

> **Interviewer:** You have 15 minutes. Threat-model this service for me.

<details>
<summary><b>Model answer</b></summary>

1. **Assets:** customer data, credentials, the CI pipeline, the image
   registry, Terraform state, the identity provider, and write access to the
   gitops repo (a commit there is a deploy).
2. **Entry points:** the public load balancer, GitHub (PRs, Actions, fork
   PRs), the registry, the cluster API, my laptop, the IdP login, the **pod
   network** (in-cluster callers reach every route), and the supply chain
   (dependencies, base images, third-party Actions).
3. **The app's own routes, from the code, not the README:** for each
   unauthenticated endpoint, what can a caller read, crash, fill or toggle?
4. **Top risks with STRIDE**, each mapped to the control that mitigates it:

| | Meaning | Example |
|---|---|---|
| **S**poofing | Pretending to be someone | A leftover CI access key used by whoever finds it |
| **T**ampering | Changing what you shouldn't | An image tag moved to a malicious build |
| **R**epudiation | Denying it, because nothing records it | "Who changed prod?" with no gitops history or CloudTrail |
| **I**nformation disclosure | Reading what you shouldn't | A secret via `/env` |
| **D**enial of service | Making it unavailable | `GET /panic` in a loop |
| **E**levation of privilege | Gaining more rights | A pod role with `iam:*` |

Also: look for **existing** long-lived credentials in the account, not just
the ones I'm about to create.
</details>

**Follow-ups**

<details>
<summary><b>21.1 ★</b> "Walking the routes, you find debug and demo endpoints. What can an attacker do with each if it's public?"</summary>

**In the Podinfo run:**

| Endpoint | Abuse |
|---|---|
| `/env` | Every env var, **including any secret injected as one** |
| `/debug/pprof/` | Heap profiles can contain secrets in memory; CPU profiling costs CPU on demand |
| `/panic` | Kills the process. In a loop: a crash loop (DoS) |
| `/fault_injection/enable`, `/readyz/disable` | Make a replica fail or leave the load balancer on purpose |
| `/store` | Unauthenticated writes to `/data`; with no size limit → node disk pressure → evictions |
| `/delay/{s}`, `/chunked/{s}` | Hold connections open cheaply → exhaustion |
| `/metrics` | Internal names, versions and traffic shape: useful recon |
</details>

<details>
<summary><b>21.2 ★</b> "Your public route only allows the CUJ paths. Is <code>/panic</code> still a risk?"</summary>

Yes. The edge only protects the internet path. Any pod that network policy
lets reach the port (or anyone with `port-forward`) can still call it, and
NetworkPolicy works at L3/L4: it can't say "allow `/version` but not
`/panic`". One request kills the process, so a compromised neighbour pod can
keep every replica crash-looping. Fix it in the app; the edge rule is defence
in depth, not the fix. (Podinfo: ADR-0008.)
</details>

<details>
<summary><b>21.3 ★</b> "There's a 30-second server write timeout in front of <code>/delay/{n}</code>. Why is it still a DoS?"</summary>

The handler sleeps without watching the request context. The timeout closes
the *connection*, but the goroutine keeps sleeping for `n` seconds:
`/delay/999999999` is a goroutine that never ends, for the price of one
request. Timeouts only help if the handler honours `r.Context()`. Look for
sleeps, loops and outbound calls that ignore it.
</details>

<details>
<summary><b>21.4 ★</b> "<code>/metrics</code> is served on the main app port. Does that mean it's on the internet?"</summary>

Not automatically, but *whatever routes that port to users routes `/metrics`
too*. Whether that's the internet depends on the Gateway or Ingress rules.
Fix: route only allowed paths publicly, and scrape metrics on a separate
internal port. (Podinfo serves it on 9898 as well as 9797,
`server.go:138`.)
</details>

<details>
<summary><b>21.5 ★</b> "You want the CUJ to prove <code>/panic</code> is disabled. Why not just call it and expect 404?"</summary>

If the gate ever regresses, your own check kills the process: the CUJ becomes
the outage, in prod. Probe with the **wrong method** instead (`POST /panic`
on a GET-only route). gorilla/mux answers **405** when the path is
registered but the method doesn't match, and **404** when the path isn't
registered (`mux.go:204-209`, v1.8.1). So 405 = still there (fail), 404 =
gated (pass), and the handler never runs. Confirm your router behaves this
way: some answer 404 for both.
</details>

<details>
<summary><b>21.6</b> "What is 'denial of wallet', and what limits it?"</summary>

An attacker, or just a bug, drives autoscaling: traffic → more pods → more
nodes → a growing bill, while the service "handles" it. Limits: HPA
`maxReplicas` and node-autoscaler limits, rate limits at the edge (WAF), and
the budget alert as a backstop (it lags by hours, so it's no brake).
</details>

---

## P0-10 — Non-functional requirements

### 22. "It must be highly available"

> **Interviewer:** The product owner says "it must be highly available and
> fast." Turn that into something you can build and test.

<details>
<summary><b>Model answer</b></summary>

I turn it into numbers, per step of the CUJ:

- **SLI**: what I measure, e.g. the share of CUJ requests that succeed, and
  p95 latency.
- **SLO**: my internal target, e.g. 99.5% success, p95 < 500 ms.
- **SLA**: a *contract* with consequences, like refunds. Usually none for
  internal services; when there is one, it's looser than the SLO.
- **Error budget**: 100% − SLO. 0.5% of a 30-day window is about 3.6 hours.
  Spend it on risky changes; when it's gone, freeze features and fix
  reliability.

Plus **RTO** (how long we may be down) and **RPO** (how much data we may
lose, in time), compliance rules (PII, retention, region), and expected
normal and peak traffic. These are drafts until real measurements exist.

**Weak answer:** "Five nines." For what, measured how, and who pays for it?
</details>

**Follow-ups**

<details>
<summary><b>22.1</b> "You test the 99.5% SLO during a 4-hour lab session. Is that a fair test?"</summary>

No. Express the error budget **in time for its window**: 0.5% of 4 hours is
about 72 seconds, which one slow restart can spend. Judge the SLO over
staging or prod windows, not single sessions.
</details>

<details>
<summary><b>22.2</b> "Why keep 'safety caps' separate from the SLOs?"</summary>

They answer different questions. SLOs say how good it must be for users, and
you *aim* at them. Caps (HPA max, quotas, node limits, budget) say how bad a
flood, a bug or an attacker can make the bill or the blast radius, and you
*never exceed* them. Mixed together, someone raises a cap "to meet the SLO"
and quietly removes the denial-of-wallet protection. Caps come from the
threat model; SLOs come from the CUJ.
</details>

<details>
<summary><b>22.3 ★</b> "The plan says HPA minimum 2 and a PodDisruptionBudget of <code>minAvailable: 2</code>. What happens at the next node upgrade?"</summary>

The drain can never evict a pod: evicting one would leave 1 < 2, so the PDB
refuses, and node upgrades and consolidation hang. Use a minimum of 3 with
PDB 2, or PDB `maxUnavailable: 1`. Same trap, worse: `minAvailable: 1` on a
one-replica Deployment.
</details>

<details>
<summary><b>22.4 ★</b> "To protect the budget, you capped the cluster at exactly its 3-node floor. What breaks?"</summary>

Anything that replaces nodes by adding one first: managed node group rolling
updates, Karpenter drift and expiry replacement, AMI upgrades. They stall or
fall back to disruptive replacement. Leave room for one surge node. (In
Karpenter, limits are CPU and memory on the NodePool, not a node count.)
</details>

<details>
<summary><b>22.5</b> "The app has no data, so the RPO is 'n/a'. Is the DR conversation over?"</summary>

No. There's still an RTO: cluster, add-ons, secrets, DNS and certificates
must come back, and the measured rebuild time *is* the RTO. And "no app data"
isn't "no state": Terraform state, the gitops repo and the secret values are
what the rebuild depends on. Lose them and there's nothing to rebuild from.
Write down how they're protected.
</details>

---

## P0-11 / P0-11b / P0-12 — Backlog, ownership, conventions

### 23. "Nobody rolled back" ★

> **Interviewer:** A bad release ran at 100% for 40 minutes. The developers
> thought platform would roll back; platform thought the developers would.
> How do you stop that happening again?

<details>
<summary><b>Model answer</b></summary>

It's not a technical failure, so no tool fixes it. I'd write an
**ownership contract** with the dev team: developers own the Dockerfile,
tests, error handling and versioning; platform owns the pipeline, the deploy
gate, rollback mechanics and infrastructure alerting. And five things are
agreed explicitly, in writing:

1. **Image tags:** git SHA (plus semver for releases), never `latest`.
2. **What blocks a deploy:** e.g. CRITICAL/HIGH findings block, MEDIUM warns;
   who can grant an override, and for how long.
3. **Who approves prod:** e.g. automatic to staging when green; prod needs
   an approval.
4. **Who triggers rollback:** automatic on canary failure, and *either* side
   may `git revert` without asking.
5. **Who keeps each service manifest current.**

Plus incident rules: page by layer (load balancer and nodes → platform, app
errors → dev team, unclear → both), **one incident commander**, and any
console fix reconciled back into Git within 24 hours.
</details>

**Follow-ups**

<details>
<summary><b>23.1</b> "On your project you're both the developer and the platform engineer. Why write an ownership contract at all?"</summary>

The failures it prevents are unstated assumptions: "tests passed so it's
live", "the other side will roll back", "nobody said rotation restarts pods".
Writing it says which hat makes which call, and it's exactly what a second
person (or an interviewer) needs. It also forces concrete rules: never
`latest`, CRITICAL/HIGH blocks, either hat may `git revert`.
</details>

<details>
<summary><b>23.2 ★</b> "Your gitops repo requires code-owner review on <code>envs/prod/</code>, and you're the only code owner. What happens?"</summary>

GitHub never lets a PR author approve their own PR, so prod promotions can
never merge, unless you bypass as admin every time, which trains you to
ignore the gate. Solo, the enforced gate is a **GitHub Environment `prod`
with a required reviewer** (self-review allowed: a deliberate, logged click).
Keep `CODEOWNERS` as documentation and turn code-owner review on when a
second person joins. Name the control that is *actually* enforced.
</details>

<details>
<summary><b>23.3</b> "How do you show which parts of the plan you deliberately skipped?"</summary>

In the backlog: `[OPT]` for a ticket the app *could* do but you choose to
skip (a service mesh, a second secrets store), and `n/a` for one the app
*can't* exercise (schema migrations with no database), with where you'll
learn it instead. Both make the skip visible. A deleted ticket looks exactly
like a forgotten one.
</details>

---

## P0-13 — Why EKS?

### 24. "Why Kubernetes?"

> **Interviewer:** Why did you choose Kubernetes for this? Why not ECS or
> Lambda?

<details>
<summary><b>Model answer</b></summary>

What the question really asks is *"do you know when **not** to use it?"*
I walk down the options and stop at the first honest yes:

1. Event-driven, stateless, finishes in under 15 minutes? → **Lambda.**
   Don't containerise it out of habit.
2. Under ~20 services, one team, no multi-tenancy, no need for the
   Kubernetes ecosystem? → **ECS on Fargate.** No control plane fee, no node
   upgrades.
3. Need GitOps, admission policy, operators or a mesh, several teams on
   shared infrastructure, or portability? → **EKS**, accepting about
   $73/month per control plane plus real Day-2 work (upgrades every few
   months, add-ons, patching).

Then I'm honest about my own case: for one small app, ECS would be simpler.
I chose EKS because the project's goal is to *learn platform engineering*
(GitOps, policy, workload identity). That's a valid learning reason, but it
isn't an architecture reason, and I don't mix the two up.

**Weak answer:** "Kubernetes is the industry standard," or anything that
can't name a case where you *wouldn't* use it.
</details>

**Follow-ups**

<details>
<summary><b>24.1</b> "A seed startup: three services, two engineers. What do you give them?"</summary>

ECS on Fargate, or Lambda, with CloudWatch. No GitOps, mesh or policy engine
yet: each one is another system two people must run. Revisit when there are
more teams, more services, or a compliance requirement. What you leave out is
the real decision.
</details>

<details>
<summary><b>24.2</b> "A payments company with card data. Does that change the shape?"</summary>

Yes. Compliance decides topology. The service that touches card data gets
**its own cluster and its own account**, however small, so the PCI scope
stays small. Partner banks often need fixed IPs or mutual TLS, which can
force an NLB at the edge. The orchestrator matters less than the controls
around it.
</details>

<details>
<summary><b>24.3</b> "Say you're wrong. What would it cost to move off EKS later?"</summary>

The reversal cost is moderate: container images are portable as they are;
Kubernetes manifests have to be rewritten as ECS task definitions, and the
EKS-specific parts (Argo CD, Kyverno, operators) have to be replaced or
dropped. Weeks, not a rewrite. Saying that in the ADR, together with **the
case against my own choice** and what would change my mind, is what makes
the decision defensible.
</details>

<details>
<summary><b>24.4</b> "Would you ever mix them?"</summary>

Yes, most real systems do. EKS for the core services, Lambda for event glue
and scheduled jobs, Fargate for a few isolated workloads. "Which platform"
is a per-workload question, not a company-wide religion.
</details>

---

## How we work

### 25. "A confident wrong answer" ★

> **Interviewer:** An AI assistant, or a senior colleague, gives you a
> confident technical claim about the app. How do you handle it?

<details>
<summary><b>Model answer</b></summary>

As a hypothesis. I verify it against the code and cite file:line before it
goes into any document. Being confident doesn't make it correct.

**In the Podinfo run:** three claims in an AI-drafted manifest were wrong
(the JWT key "reloads", `/store` "only if enabled", `/metrics` "only on
9797"), and the first CUJ draft would have passed on a broken pod.
</details>

**Follow-ups**

<details>
<summary><b>25.1</b> "You're following the team's runbook and a step turns out to be wrong. What do you do?"</summary>

Fix the runbook itself, not just my local copy, and write a short RCA: what
happened, root cause, fix, lesson. Otherwise the next person hits the same
wall. (Here: fix the template step, commit to `main`, RCA in
`platform/docs/rca/`, e.g. `2026-10-13-p0-3-cuj-checks.md` → template
`0103c4a`.)
</details>

<details>
<summary><b>25.2</b> "A teammate keeps their investigation notes in a local folder. Why push them to put the notes in the repo?"</summary>

If it's not in a committed file, it doesn't exist. Loose files aren't
versioned, reviewed or backed up, and the next person, or the next session,
can't find them. That's why `questions.md` and the RCAs moved into the
platform repo.
</details>
