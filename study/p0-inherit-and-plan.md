# Study bank — Phase 0: Inherit & plan

Review and interview questions for [Phase 0](../template/00-inherit-and-plan.md).
A living file: add a question whenever a run surprises you, a check fails, or
an RCA gets written.

**Worked example:** the answers are grounded in a real run on
[Podinfo](https://github.com/nazmur96/podinfo) (fork of `stefanprodan/podinfo`,
read at `3e0ff8a` / tag `6.15.0`). File:line references are into that repo.
Paths like `platform/docs/…` are that project's platform repo. The *questions*
apply to any app. Answer them for yours, and use the Podinfo answers to check
your reasoning.

**How to use it:** read the question, answer out loud or on paper, *then*
open the answer. Scenario and interview questions have no single right
answer. The answer shows the reasoning a strong candidate would give.

**Tags:** `[R]` review (recall) · `[S]` scenario (what would you do / what
went wrong) · `[I]` interview (explain it to someone else) · `★` came from a
real finding or mistake in a run.

**Adding a question:** put it under its ticket, tag it, keep the answer short,
and cite the file/line or doc that proves it. If it came from a mistake, link
the RCA. Number new questions after the last one; never renumber, so
references like "study P0 #17" stay valid.

---

## Big picture

<details>
<summary><b>1. [R]</b> What is Phase 0 for, and why is "create a cluster" the wrong first step?</summary>

P0 answers *what are we running, what does it need, what does it cost, what
can go wrong, and how will we know it works*, before any infrastructure.
A cluster created first gets sized, networked and secured by guesswork, and
later phases inherit those guesses (wrong ports → 502s, wrong sizes → OOM or
waste, no threat model → over-exposed endpoints). P0 outputs are cheap to
change; a VPC/EKS layout is not.
</details>

<details>
<summary><b>2. [R]</b> Inventory vs service manifest vs CUJ: one sentence each, and where each one lives.</summary>

- **Inventory** (`platform/docs/inventory.md`): map of the whole system:
  components, ports, who calls whom, constraints. Feeds NetworkPolicy and sizing.
- **Manifest** (`MANIFEST.md` in the **app** repo): contract for *one* service:
  port, protocol, state, start/stop, secrets, failure behaviour, owner. It is
  the diff you use in an incident.
- **CUJ** (`platform/docs/cuj.md`): the shortest scripted path proving users
  are actually served. Run after every deploy.

The manifest lives next to the code so it changes in the same PR as the code.
</details>

<details>
<summary><b>3. [I]</b> "Isn't all this documentation just overhead? Real engineers just deploy and fix what breaks."</summary>

Each P0 artifact prevents a specific, expensive failure: the manifest prevents
target-group/port mismatches and drain-time 502s; the inventory prevents
"open everything so it works" network rules; the CUJ prevents celebrating
green pods that don't serve users. Fixing these *after* deploy means
debugging in production, under pressure, often by someone who didn't build
it. The cost is hours now vs incidents later. And the docs are small: one
table, one form, one script.
</details>

---

## P0-1 — Get access and read everything

<details>
<summary><b>4. [R]</b> List the fork safety checks you ran and what each one prevents.</summary>

| Check | Prevents |
|---|---|
| Actions disabled on the fork | Upstream workflows running on *your* repo: burning minutes, failing on missing secrets, or trying to publish |
| Default workflow token = read | A workflow (once enabled) writing to the repo/packages without being granted it explicitly |
| `upstream` push URL set to `no-pushing-to-upstream` | Accidentally pushing to someone else's project |
| `gh repo set-default` = your fork | `gh pr create` / `gh issue` silently targeting upstream |
| Default branch noted (`master`) | Later OIDC `sub` claims and workflow triggers written for `main` that never match |

Verify by observing (`gh api …/actions/permissions`), not by trusting that you clicked the setting.
</details>

<details>
<summary><b>5. [S] ★</b> The upstream `release.yml` is still in your fork. A teammate enables Actions to "test the pipeline quickly". What can go wrong?</summary>

`release.yml` pushes images, charts and Timoni modules to **stefanprodan's**
Docker Hub and GHCR and signs with a `COSIGN_KEY` secret. On your fork it
fails (no credentials), but any workflow whose trigger matches will run, and
you've just executed code you didn't review with whatever permissions the
token has. Rule: delete or replace upstream workflows **in the same commit**
that enables Actions (P7), and review every `on:` trigger first.
</details>

<details>
<summary><b>6. [S]</b> A developer says "it runs on port 3000, I think." How do you get the real answer?</summary>

Check the code, not memory: the listen call or flag default
(podinfo: `fs.Int("port", 9898, …)` in `cmd/podinfo/main.go:34`), then what
deploy config actually passes (chart `--port={{ .Values.service.httpPort }}`),
then env overrides (`PODINFO_PORT`). Then confirm by running it (P2). The
answer to write down is *the value and where it came from*, not "3000".
</details>

<details>
<summary><b>7. [S] ★</b> README, Helm chart and Kustomize disagree on resources (`1m`/`16Mi` vs `100m`/`64Mi`→`2000m`/`512Mi`). Which do you trust?</summary>

None of them, for sizing. They're defaults written for different purposes
(the chart for tiny test clusters, kustomize for a demo). Code is the source
of truth for *behaviour* (ports, paths); **measurement** is the source of
truth for *size*. Record the disagreement as a finding and measure under load
in P3. P0's rule: don't guess sizes.
</details>

<details>
<summary><b>8. [R]</b> Where can a podinfo setting come from, and in what order do they win?</summary>

Three sources through viper: command-line flags, env vars `PODINFO_<FLAG>`
(`-` → `_`; `main.go:90-92`), and `config.yaml` under `--config-path`
(`main.go:93-100`). Viper's precedence: explicit flag > env > config file >
default. Implication: a value in a ConfigMap-mounted file can be silently
overridden by an env var someone added to the Deployment.
</details>

<details>
<summary><b>9. [I]</b> How do you hunt for config and secrets in a codebase you've never seen, in any language?</summary>

Search for the stack's config entry points: `os.Getenv`/viper/flags (Go),
`process.env`/dotenv (Node), `System.getenv`/`application.yml`/`@Value`
(Java/Spring), `os.environ`/pydantic settings (Python), `std::env` (Rust),
`ENV[]` (Ruby). Then look for **hardcoded defaults** (podinfo's
`SetDefault("jwt-secret", …)`), mounted file paths, and cloud SDK clients
(which imply credentials). Read the deploy config for what's actually passed.
List every key as plain or secret.
</details>

<details>
<summary><b>10. [S]</b> It's an open-source app. P0-1 says "book 30 minutes with the dev lead". There is none. Now what?</summary>

Answer each question by one of three routes, and write down which:
**code** (read it, cite file:line), **experiment** (run it in P2/P3 and
observe), or **decision** (we choose, record as an ADR). The `questions.md`
"How we answer it" column exists for this. (This gap was fixed in the
template's P0-1.)
</details>

<details>
<summary><b>11. [R] ★</b> Your <code>grep -rn … --include=*.go</code> printed "no matches found" without searching anything. Why?</summary>

The shell is **zsh**: an unquoted `*.go` is expanded by the shell first, and
zsh errors when the glob matches nothing in the current dir. Quote it:
`--include='*.go'`. Same trap with `?` in URLs (`gh api 'repos/…?recursive=1'`).
</details>

---

## P0-2 — Application inventory

<details>
<summary><b>12. [R]</b> Name the inventory columns and the later decision each one feeds.</summary>

| Column | Feeds |
|---|---|
| Port(s) | Service, probes, target group / Gateway backend (P6, P9) |
| Health endpoint | Probes, ALB health checks |
| Datastore | Data plan, backups, secrets (P4, P5) |
| Sync callers / callees | NetworkPolicy allow-list (P8) |
| Async (queue) links | Queue infra, idempotency, KEDA scaling |
| Owner | Who gets paged; who approves changes |
| Constraints | Tickets: hardcoded creds, no SIGTERM, EOL runtimes, licences |
</details>

<details>
<summary><b>13. [S] ★</b> The skeleton said podinfo only calls Redis. What did the code actually show, and why does it matter for P8?</summary>

Three opt-in outbound calls: Redis (`--cache-server`), backend podinfo via
`POST /echo` (`--backend-url`), and an OTLP collector (`--otel-service-name`,
endpoint from `OTEL_EXPORTER_OTLP_ENDPOINT`). With no flags it calls nothing.
So the **egress NetworkPolicy depends on configuration**. Turn on tracing
in P10 without adding an egress rule and spans silently drop. Config
choices must be fixed before P8, and changing them later means changing the
policy in the same commit.
</details>

<details>
<summary><b>14. [I]</b> How do you find every outbound dependency of a service you didn't write?</summary>

1. Code: HTTP clients, gRPC dials, DB/cache drivers, cloud SDKs, and env
   vars named `*_URL`/`*_HOST`/`*_ENDPOINT`.
2. Config: what the deploy actually sets.
3. Behaviour: run it under default-deny egress and watch what breaks, or
   observe flows (VPC flow logs, Hubble, conntrack).
Code tells you what's *possible*, config tells you what's *on*, and traffic
tells you what's *actually happening*. You need all three.
</details>

<details>
<summary><b>15. [S] ★</b> Why is a hardcoded default JWT secret a vulnerability when the code is public anyway?</summary>

*Because* the code is public: everyone knows the key
(`cmd/podinfo/main.go:85`). Anyone can mint a token that `/token/validate`
accepts. A default secret is a shared secret with the whole internet. Fix:
always set a real one (P5), and add a CUJ negative step proving the default is
rejected.
</details>

<details>
<summary><b>16. [S] ★</b> Go through podinfo's dangerous endpoints. What can an attacker do with each if it's public?</summary>

| Endpoint | Abuse |
|---|---|
| `/env` | Returns `os.Environ()` (`env.go`): every env var, **including any secret injected as an env var** |
| `/debug/pprof/` | Heap profiles can contain secrets/tokens in memory; CPU profiling costs CPU on demand |
| `/panic` | Kills the process. Repeat it and you get a crash loop (DoS) |
| `/fault_injection/enable`, `/readyz/disable` | Make a replica fail or leave the LB on purpose |
| `/store` | Unauthenticated writes to `/data`. emptyDir with no `sizeLimit` → node disk pressure → evictions |
| `/delay/{s}`, `/chunked/{s}` | Hold connections open cheaply → connection/goroutine exhaustion |
| `/metrics` | Internal names, versions and traffic shape: useful recon |

Mitigation: path allow-list on the public HTTPRoute (P9), not "hope nobody finds it".
</details>

<details>
<summary><b>17. [S] ★</b> Follow-up to 16: in P5 you'll inject the JWT key. Why is <code>PODINFO_JWT_SECRET</code> as an env var risky <i>for this app specifically</i>, and what's the alternative?</summary>

Because `/env` dumps the environment. Any route to `/env` (a missed path on
the HTTPRoute, a port-forward, a compromised pod in the namespace) leaks the
key. Alternatives: mount the secret as `config.yaml` under `--config-path`
(viper reads `jwt-secret` from it at startup, `main.go:93-100`), so it never
appears in the environment; *and* block `/env` at the edge. Either way it
still needs a restart to rotate. **Open decision for P5**, recorded in
`questions.md` #18 and the manifest config table.
</details>

<details>
<summary><b>18. [R]</b> Why does an inventory list the licence?</summary>

The licence decides what you may do with a fork. Podinfo is Apache-2.0: fork,
modify, redistribute, keep `LICENSE` and notices. Copyleft licences (GPL,
AGPL) can oblige you to publish changes; AGPL triggers on *network use*, which
is exactly what running it as a service is. Better to find out in P0 than from legal.
</details>

---

## P0-2b — Service manifest

<details>
<summary><b>19. [R]</b> Name the 9 manifest sections. Which ones are usually missing, and why do they matter most?</summary>

Identity, Network, Compute, Data, Observability, Config, Start/stop,
Dependencies, Ownership. **7–9 (start/stop, dependencies, ownership)** are
the ones usually missing, and they're what you need at 3 a.m.: how long it takes
to start, whether it drains, what it calls, and who to page.
</details>

<details>
<summary><b>20. [S]</b> 3 a.m., users get 502 from the ALB; pods are 1/1 Running. How does the manifest shorten this?</summary>

Compare manifest vs live config field by field: manifest says HTTP/1.1 on
**9898**. Check the target group port/protocol and the health check path. A
TG on 80, or HTTPS to a plain-HTTP pod, or health check on a path that
doesn't exist → 502 with healthy-looking pods. The manifest turns
"guess" into "diff".
</details>

<details>
<summary><b>21. [S] ★</b> Cursor's draft said the JWT key might reload via the file watcher. What does the code say, and what's the general lesson?</summary>

The key is read **once** at startup into the server config (`viper.Unmarshal`,
`main.go:165`). The `fscache` watcher only feeds `GET /configs`
(`configs.go:11`). The app never uses those values. So rotation needs a
restart. Lesson: **"has a file watcher" ≠ "reloads the secret it uses".**
Trace the value from where it's loaded to where it's used.
</details>

<details>
<summary><b>22. [S] ★</b> You rotate the JWT key in Secrets Manager. ESO syncs the K8s Secret. What happens to running podinfo pods, and what goes wrong mid-rollout?</summary>

Nothing changes in running pods: they keep the old key until restarted (env
vars never update in a running process; podinfo doesn't re-read the file
either). You need a rollout on change (Reloader, or a checksum annotation
that changes the pod template). **During** the rollout, old and new pods
have different keys: a token issued by a new pod fails validation on an old
pod. With 2 replicas behind an LB that's roughly 50% failures for the
1-minute token lifetime. Short token TTL limits the damage; the CUJ's steps 5→6
detect it.
</details>

<details>
<summary><b>23. [S] ★</b> Walk through what happens when Kubernetes terminates a podinfo pod. Where can requests drop?</summary>

SIGTERM → podinfo marks healthz/readyz false → **sleeps 3s** → OTel shutdown
→ gRPC GracefulStop → HTTP `Shutdown` with 5s timeout (`signals/shutdown.go`).
In parallel, Kubernetes removes the pod from EndpointSlices, and the AWS LB
Controller deregisters the target. That can take longer than 3s. If the
process stops listening before the ALB stops sending, the ALB returns 502s.
Fixes: `preStop` sleep longer than the propagation delay, pod readiness gates,
and `terminationGracePeriodSeconds` > preStop + shutdown. Also note that the
readiness probe (period 10s) won't even observe the unready flag within 3s.
Endpoint removal on termination is what actually takes it out of service.
</details>

<details>
<summary><b>24. [S] ★</b> During an incident someone sets <code>--level=debug</code> to get more logs. What new problem did they just create?</summary>

The drain sleep is skipped at debug level (`shutdown.go`: `if level != "debug"`).
Every subsequent rollout drops in-flight traffic. That's more 502s, caused by
the debugging. Recorded in the manifest as "never debug in prod".
</details>

<details>
<summary><b>25. [I] ★</b> Should podinfo's readiness probe check Redis?</summary>

Probably not. Redis only serves `/cache/*`. If readiness checked it, a Redis
blip would mark **every** replica unready at once and the LB would drop the
whole service, turning a partial outage (cache endpoints fail) into a total
one. Rule: readiness checks what the pod *needs to serve its core traffic*;
liveness checks only the process itself (never dependencies, or a dependency
outage restarts every pod). Do record "readiness ignores Redis" as a known
constraint so nobody assumes otherwise.
</details>

<details>
<summary><b>26. [S] ★</b> "It's stateless." Prove or disprove it for podinfo by behaviour, not definition.</summary>

Ask: *if I kill the pod mid-request, what does the user lose?* and *does it
write anything to disk?* `POST /store` is always routed and writes to `/data`
(`store.go:35`). Kill the pod, and stored content is gone; with 2 replicas,
`GET /store/{hash}` succeeds only on the replica that wrote it. Verdict: no
state *we rely on* (not in the CUJ). Disk writes exist, so: emptyDir, no
PVC, and document it. Without `readOnlyRootFilesystem` + a `/data` mount the
answer would change again.
</details>

<details>
<summary><b>27. [S] ★</b> If Redis were added later: 10 replicas, traffic spike, Redis falls over. What in the code contributes?</summary>

The pool sets `MaxIdle: 3` and **no `MaxActive`** (`cache.go:147`). Connections
are unbounded per replica, so a spike × 10 replicas can exceed Redis
`maxclients`. And there's no retry or circuit breaker, so failures pass
straight to users as 500s while pods stay Ready. Manifest section 4 asks
"pooled? peak connections?" for exactly this.
</details>

<details>
<summary><b>28. [S] ★</b> <code>/echo</code> to a slow backend: client timeout default is 2 minutes, the ALB idle timeout is 60s. What happens?</summary>

The ALB gives up at 60s and returns 504 to the user. Podinfo keeps waiting up
to 2m per backend, holding goroutines and connections for requests nobody is
waiting on. Under load, resources pile up. Timeouts should shrink as you go
deeper (edge > service > dependency), not grow.
</details>

<details>
<summary><b>29. [R] ★</b> <code>/metrics</code> is "on the public port". Does that mean it's on the internet?</summary>

Not automatically. It's served on 9898 (`server.go:138`) as well as 9797 when
enabled, so **whatever routes 9898 to users routes `/metrics` too**. Whether
that's the internet depends on the Gateway/HTTPRoute. Fix: route only allowed
paths publicly (P9), and scrape on 9797 inside the cluster.
</details>

<details>
<summary><b>30. [I]</b> Why does "who keeps the manifest current" matter more than how complete it is?</summary>

A stale manifest is worse than none: it sends the on-call engineer down the
wrong path with confidence. Completeness decays with every change; an owner
plus "update it in the same PR that changes ports/deps/secrets" keeps it true.
</details>

---

## P0-3 — Critical User Journey

<details>
<summary><b>31. [R]</b> What is a CUJ, and why not just test every endpoint?</summary>

The shortest path that proves users are actually served. It runs after every
deploy, sync and canary step, so it must be fast and stable. Every endpoint
means slow, flaky and noisy (drills like `/panic` don't belong). Full coverage
is the test suite's job; the CUJ is the deploy gate.
</details>

<details>
<summary><b>32. [S] ★</b> All pods are 1/1 Running, health checks pass, users get 500 on everything. How is that possible in podinfo?</summary>

Fault injection exempts `/healthz`, `/readyz`, `/metrics`, `/debug/` and
`/fault_injection/*` (`fault_injection.go:18-30`), so the platform keeps the
sick pod in service while every app path fails. That's why the CUJ must
include steps that prove *serving* (`/version`, `/`, `/token`) and assert
response bodies. A CUJ of only health checks passes here.
</details>

<details>
<summary><b>33. [S] ★</b> Argo CD says Synced/Healthy, the CUJ passes, but you're running the old release. How would you know?</summary>

Step 1: `/version` must return `commit` equal to the SHA CI just built. The
binary defaults `commit` to `"unknown"` (`version.go:4`), which still returns 200.
So "status 200" proves nothing. Assert the value. Typical causes: tag reused
(`latest`), image digest not updated in gitops, pod not restarted.
</details>

<details>
<summary><b>34. [S] ★</b> The token issue → validate round-trip passes. Does that prove P5 wired the real secret?</summary>

No. It passes with *any* shared key, including the hardcoded default. Only a
**negative** step proves it: a token signed with the upstream default must be
rejected (CUJ step 7). General rule: for every secret with a default in code,
the CUJ needs a step that fails if the default is in use.
</details>

<details>
<summary><b>35. [S]</b> Your CUJ script stores the token in a file and reuses it next run. It fails intermittently. Why?</summary>

Tokens expire after 1 minute (`token.go`). Issue and validate back to back,
in the same run. A CUJ must not depend on state left over from the previous run.
</details>

<details>
<summary><b>36. [S]</b> Your CUJ checks <code>GET /</code> returns 200. A colleague runs it from a browser-like client and gets HTML. Is the check wrong?</summary>

`/` content-negotiates on `User-Agent`: `Mozilla*` gets the HTML UI, everything
else JSON (`server.go:139-140`). Status-only checks pass on both and prove
little. Script it with curl and assert JSON fields (`hostname`, `version`).
</details>

<details>
<summary><b>37. [I]</b> How do you know a check is any good?</summary>

Ask: **what broken state would still pass this?** If you can name one, the
check is incomplete. Then deliberately break it (enable fault injection,
deploy an old tag, use the default secret) and confirm the check goes red.
A check that has never failed in testing hasn't actually been tested.
</details>

<details>
<summary><b>38. [I]</b> CUJ vs smoke test vs synthetic monitoring vs load test: how do they relate?</summary>

Same journey, different trigger and question. **Smoke/CUJ** after a deploy:
"did this change break the core path?" **Synthetic monitoring** on a
schedule in prod: "is it working right now, from outside?" **Load test**
(k6, P10): "does it still work at N users, within the SLO?" Writing the CUJ
once and reusing it across all three keeps them consistent.
</details>

<details>
<summary><b>39. [I]</b> Liveness, readiness, startup probes: what does each mean, and what goes in each for podinfo?</summary>

- **Liveness:** "restart me if I'm wedged." Process-only (`/healthz`). Never
  check dependencies, or a dependency outage restarts every pod.
- **Readiness:** "send me traffic?" (`/readyz`). It goes false during
  shutdown, so endpoint removal and draining work.
- **Startup:** "give me time before liveness starts." Needed for slow
  starters. Podinfo becomes ready right after its listeners start, so the
  chart leaves it off. A 90s-start app without it gets killed in a loop by
  liveness.
</details>

---

## P0-4 — Criticality and rollout order

<details>
<summary><b>43. [R]</b> Why does P0-4 roll out low-criticality services first and critical ones last?</summary>

The first services you move are where your patterns are still wrong: the
Dockerfile, probes, policies and pipeline. Mistakes there should hit
something that hurts little. By the time you move the critical service, those
patterns have been fixed on cheaper failures. In a greenfield build the same
rule decides which service you containerise and harden first as the
pattern-setter.
</details>

<details>
<summary><b>44. [I]</b> Who decides a service's criticality, and why not engineering?</summary>

The product owner (or whoever owns the business outcome). Criticality is
*business impact*: revenue, users, legal exposure. Engineers tend to rank by
technical complexity or by what they're proud of. In an OSS/learning run with
no product owner, write down the proxy you used (podinfo: "does it block the
CUJ?") so the rule is visible, not implied.
</details>

<details>
<summary><b>45. [S] ★</b> The template says the pilot must include a stateful component. Podinfo has none. What do you do?</summary>

Deviate **explicitly**: write down that there's no stateful pilot and why
(Redis out of v1, `/store` is emptyDir only). Also write down which lessons
the pilot therefore *can't* teach: `startupProbe` (podinfo is ready in about 1s),
backups/PVCs/DB migrations, connection pooling, multi-service NetworkPolicy.
Say where you'll learn each one (Retail Store). A silent deviation means you
think you covered something you didn't. (`platform/docs/rollout.md`)
</details>

---

## P0-5 — Size from data, not guesses

<details>
<summary><b>46. [S] ★</b> 3 × <code>t3.medium</code>, memory 50% used, but the new canary pod is <code>Pending</code>: "0/3 nodes are available: Too many pods". Explain and fix.</summary>

The VPC CNI gives each pod a real VPC IP, so pods per node are capped by
ENIs and IPs: `t3.medium` = 17. Subtract ~6 DaemonSet pods per node and the
cluster has ~33 slots. Argo CD (~7), Kyverno (~4), ESO (3), the LB controller,
CoreDNS, metrics-server and Karpenter fill them before the app arrives. Fixes:
bigger nodes (`t3.large` = 35), VPC CNI **prefix delegation** (assigns /28
prefixes, about 110 pods/node), or fewer platform pods. Lesson: on EKS, size by
pod count as well as CPU/memory.
</details>

<details>
<summary><b>47. [R]</b> Max pods formula on EKS with the VPC CNI. Why "−1" and "+2"?</summary>

`ENIs × (IPv4 per ENI − 1) + 2`. Each ENI's primary IP belongs to the ENI
itself, not to pods (−1). `aws-node` and `kube-proxy` use host networking, so
they don't consume an ENI IP but do count as pods (+2). `t3.medium`: 3 × (6 − 1)
+ 2 = 17. `t3.large`: 3 × (12 − 1) + 2 = 35. (The template used to say
"ENIs × IPs". It was corrected during the podinfo run.)
</details>

<details>
<summary><b>48. [S]</b> Greenfield, no metrics exist. How do you produce resource requests you can defend?</summary>

Run the app under the **CUJ** load (P2 Compose with `docker stats`, then P3
kind with `kubectl top`), take a high percentile, and set requests near
steady state and the memory limit above peak. Then sum in this order:
DaemonSets × nodes → platform pods → app × replicas (+ canary) → ×1.3
headroom. Upstream defaults (podinfo's chart `1m`/`16Mi` vs kustomize
`100m`/`64Mi`) are recorded as "claims", never used as the answer.
</details>

<details>
<summary><b>49. [I] ★</b> Should podinfo have a CPU limit? A memory limit?</summary>

**Memory limit: yes.** Memory can't be throttled, only killed, so the limit
protects the node and its neighbours. **CPU limit: usually no.** CFS
quotas throttle the container even when the node is idle, causing latency
spikes. Requests already guarantee its share. For Go, also set `GOMEMLIMIT`
(~90% of the memory limit) so the GC works hard *before* the OOM killer
acts. Recent Go versions derive `GOMAXPROCS` from a CPU limit if one is set.
Without one, Go sees all node cores, which is fine for a small binary.
</details>

<details>
<summary><b>50. [S] ★</b> An idle dev cluster's control plane line on the bill is 6× what you expected. What happened?</summary>

The cluster runs a Kubernetes version that left **standard support**, so
EKS moved it to **extended support**: $0.60/h instead of $0.10/h. It's easy to
miss when you reuse old Terraform pinned to an old version. Fix: create on a
version in standard support, and check the EKS version calendar each run.
</details>

<details>
<summary><b>51. [S] ★</b> You ran a 2-hour load test on <code>t3.large</code> nodes. The EC2 line is higher than the hourly rate explains. Why?</summary>

T3 is burstable and launches in **unlimited** mode by default. Sustained CPU
above baseline (30% for `t3.large`) spends surplus credits, which are billed
per vCPU-hour. Keep load tests short, use `standard` credit mode (it throttles
instead of billing), or run load tests on non-burstable `m`-family nodes.
</details>

<details>
<summary><b>52. [S]</b> <code>terraform destroy</code> succeeded, but the next day's bill still shows a load balancer and EBS volumes. How?</summary>

Kubernetes controllers created them, not Terraform. The AWS LB Controller
makes ALBs/NLBs and security groups for `Ingress`/`Gateway`/`Service`, and the
EBS CSI driver makes volumes for PVCs. They aren't in Terraform state, so
`destroy` doesn't know about them (and they can block VPC deletion). Delete
the K8s objects first and let the controllers clean up, then `destroy`, then
check the console or bill the next day.
</details>

<details>
<summary><b>53. [I]</b> "Destroy every session" works for learning. What replaces it for a real team's dev environment?</summary>

Scheduled scale-down (nodes to zero outside working hours, Karpenter
consolidation), right-sized requests (so bin-packing works), Spot for
interruptible workloads, Savings Plans for the steady baseline, retention
limits on observability, and budget alerts per account/team. Persistent
environments are cheaper to keep than to recreate only when recreating is
slow or fragile. Measure both.
</details>

---

## P0-6 — Environment strategy

<details>
<summary><b>54. [R]</b> Why do non-prod environments share one cluster while prod gets its own?</summary>

Cost vs blast radius. Non-prod environments can share a control plane,
separated by namespaces, quotas, RBAC, NetworkPolicy and separate Argo CD
apps, because an incident there hurts no users. Prod gets its own cluster so a
bad CRD, a noisy load test, a cluster-wide add-on upgrade or a compromised dev
workload can't touch it. Namespaces isolate workloads; they don't isolate the
control plane, cluster-scoped resources or node-level compromise.
</details>

<details>
<summary><b>55. [S] ★</b> The prod <code>terraform apply</code> job fails with "Not authorized to perform sts:AssumeRoleWithWebIdentity". Dev works. The trust policy says <code>…:environment:prod</code>. What's wrong?</summary>

The GitHub Environment is named `production`, so the OIDC token's `sub` is
`repo:…:environment:production`, which doesn't match. Dev works because its
names agree. One spelling of each env name in every system (P0-12) prevents
this. The template itself had this bug (P5-3 said `production`, P5-2's trust
policy said `prod`). It was fixed during the podinfo run. Debug by printing
the token's claims in the job, not by widening the trust policy to `*`.
</details>

<details>
<summary><b>56. [I]</b> Why promote the same image <b>digest</b> through environments instead of rebuilding per environment, or retagging?</summary>

A rebuild produces a different artifact from the one you tested: new base
layers, new dependency resolutions, a new SBOM. Tags are mutable pointers.
`:1.4.0` today may not be `:1.4.0` tomorrow. A digest is the content hash, so
"what ran in staging" is provably "what runs in prod". The signature and
attestations (P7) attach to the digest, which is what admission (P8) verifies.
Environment differences belong in config (gitops values), never in the image.
</details>

<details>
<summary><b>57. [I]</b> Trunk-based development vs long-lived environment branches (<code>dev</code> → <code>qa</code> → <code>main</code>). Which, and why?</summary>

Trunk-based: short branches merge to one default branch, and environments are
folders in the gitops repo. Environment branches drift (hotfixes land in one
and not another), merges between them become releases in their own right, and each branch
tends to grow its own pipeline. With trunk + gitops folders, "what's in
staging" is one file diff, and promotion is a reviewed commit.
</details>

<details>
<summary><b>58. [S]</b> Your prod region goes down. You chose "no DR cluster, rebuild on demand". What's your RTO, and how do you know?</summary>

Only a *measured* number counts: time to `terraform apply` a new cluster in
another region, install add-ons, let Argo CD sync, restore data, and move
DNS. Typically hours. If you've never rehearsed it, you don't have an RTO,
just a guess. Rehearse it (P10), write the result in the ADR, and if the business
needs less than that, that's the trigger for warm standby (P10-11).
</details>

<details>
<summary><b>59. [I]</b> One AWS account or several? When do you split?</summary>

One account is a house with locked rooms: IAM policies, names and tags are
the only walls, and one mistake in IAM crosses them. Separate accounts per
environment are separate houses: nothing crosses unless you build a door
(cross-account role). Start with one while learning. Split when there is real
customer data, a second team or person, or a compliance requirement (P12).
Also split when billing and quotas need separating.
</details>

---

## P0-7 — Budget and cost guardrails

<details>
<summary><b>60. [S] ★</b> Dev uses public subnets and no NAT to save money. What are the risks, and what compensates for them?</summary>

Nodes are internet-addressable, so one wrong security group rule exposes them.
Compensate with: no inbound rules beyond the cluster's own, no SSH (SSM only),
the EKS API restricted to your `/32`, IMDSv2 with hop limit 1, Checkov on the
Terraform, and destroying after every session. Also know the side effects:
each public IPv4 is billed, dev's egress path differs from prod's (NAT), and
add-ons that discover their region or VPC from IMDS need those set explicitly once pods
can't reach IMDS. (ADR-0002)
</details>

<details>
<summary><b>61. [S] ★</b> Next session, <code>kubectl</code> times out against the dev cluster. Nothing changed in Terraform. Likely cause, and the wrong fix?</summary>

Your public IP changed (ISP, router restart, travel), and the EKS public
endpoint only allows your old `/32`. Right fix: update the allowed CIDR
variable and apply. Wrong fix: `0.0.0.0/0`, which puts the Kubernetes API
on the internet, protected only by authentication.
</details>

<details>
<summary><b>62. [S] ★</b> Your 100% budget alert arrived the morning after the session that overspent. Why so late, and what actually protects you?</summary>

Billing data refreshes only a few times a day, so budgets alert on spend that
already happened, often hours later. A budget is a backstop that catches the
forgotten cluster on day 2 rather than day 30. The guard is the **teardown
checklist** at the end of every session, plus a *forecasted* alert and Cost
Anomaly Detection for earlier warning.
</details>

<details>
<summary><b>63. [S] ★</b> P0-7 says to create a budget before anything else, but you have no SSO yet. Creating an access key "just for this" is tempting. Why not, and what instead?</summary>

An access key is a long-lived credential. It sits in `~/.aws/credentials`,
leaks through dotfile backups and screenshots, and outlives its purpose. It
breaks the project's "no long-lived credentials" rule from the very first
action. Instead: enable IAM Identity Center with one admin permission set and
use `aws configure sso` (pulled forward from P5-1), or create the budget in
the console. And look for old keys already on the machine. (Template P0-7 was
fixed to say this.)
</details>

<details>
<summary><b>64. [S]</b> Session 2's <code>terraform apply</code> fails: "You can't create this secret because a secret with this name is already scheduled for deletion." Why?</summary>

Secrets Manager doesn't delete immediately: destroy schedules deletion with a
7–30 day recovery window, and the name stays reserved. For `dev` only, set
`recovery_window_in_days = 0` so destroy really deletes. Keep the window in
prod, where recovery matters more than reuse.
</details>

<details>
<summary><b>65. [S]</b> You put the Route 53 hosted zone in the session stack. After every apply, DNS for <code>podinfo.&lt;domain&gt;</code> doesn't resolve. Why?</summary>

Each new hosted zone gets a new set of name servers, and the registrar still
delegates to the old ones. Keep the hosted zone outside the per-session stack
(it costs ~$0.50/month) and create and destroy only the records (ExternalDNS, P9).
Same principle for anything referenced from outside AWS: keep it stable.
</details>

---

## Meta — how we work

<details>
<summary><b>40. [S] ★</b> An AI assistant (or a teammate) gives you a confident technical claim about the app. How do you handle it?</summary>

Treat it as a hypothesis and verify it against the code, citing file:line.
In this run, three claims in a drafted manifest were wrong (JWT reload, `/store`
"only if enabled", `/metrics` only on 9797), and the first CUJ draft would
have passed on a broken pod. Being confident doesn't make a claim correct.
</details>

<details>
<summary><b>41. [R]</b> The template says one thing, reality says another. What do you do?</summary>

Fix the step in the template repo (commit to `main`), and write an RCA in
`platform/docs/rca/`: what happened, root cause, fix, lesson. Example:
`2026-10-13-p0-3-cuj-checks.md` → template `0103c4a`.
</details>

<details>
<summary><b>42. [R]</b> Why did <code>questions.md</code> and the RCA have to move into the platform repo?</summary>

"If it's not in a committed file, it doesn't exist." Loose files in a
project folder aren't versioned, reviewed or backed up, and the next person
(or the next session) can't find them.
</details>

---

## To add as we go

- P0-7 Part B (budget created) · P0-8 ADR log · threat model · NFRs ·
  backlog
