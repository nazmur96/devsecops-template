# Study bank — Phase 2: Containerise

Interview practice for [Phase 2](../template/02-containerise.md). Each
**case** is how a scenario interview actually runs: the interviewer sets a
scene and asks an open question, then pushes with follow-ups to see how deep
your understanding goes.

**New to Phase 2?** Read [Phase 2 explained simply](p2-explained.md) first,
then come back here to practise.

**How to use it:** treat it as a mock interview. Read the scene, answer out
loud for about two minutes, *then* open the model answer. Do the follow-ups
the same way. There is rarely one right answer. The model answer shows the
reasoning a strong candidate gives, and **Weak answer** shows what makes an
interviewer stop listening.

**Evidence:** answers are general first. **In the Podinfo run** notes give
evidence from the [Podinfo fork](https://github.com/nazmur96/podinfo) and its
platform repo. Use them as interview stories.

`★` = came from a real finding or mistake in a run.

**Adding a question:** add a follow-up under the case it belongs to (next
number, e.g. `1.4`), or a new case at the end of its section's numbering.
Phrase it as something an interviewer would say. Write the model answer the
way you would say it out loud: short paragraphs or **Key points:** bullets,
never a table. Cite the file/line, ADR or RCA that proves it. Never renumber
existing cases or follow-ups.

**Case numbers are IDs.** Cases 1 and 2 were written during the run and are
cited elsewhere, so they kept their numbers when the rest of the bank was
added. Sections follow the phase order, so the numbers don't always run in
order down the page.

---

## Big picture

### 3. "Upstream already ships an image"

> **Interviewer:** The open-source app you're deploying publishes a
> perfectly good image on Docker Hub. Why would you build your own?

<details>
<summary><b>Model answer</b></summary>

Because "it runs" and "I'd put it in production" are different bars, and
I'm the one on call for it.

**Key points:**

- **I need to know what's inside.** Upstream images often carry debugging
  tools (a shell, `curl`, `netcat`) that are handy for the author and handy
  for an attacker. Every extra package is more CVEs to triage.
- **It has to fit the platform's rules.** Kubernetes' `runAsNonRoot` can only
  verify a *numeric* user. An image that says `USER app` fails that check.
- **Provenance.** The image my pipeline builds once, scans, signs and
  promotes by digest is the one I can vouch for. A third party's tag can be
  re-pushed under me.
- **Patches.** If I have to change the app (switch off debug endpoints, fix a
  bug), I'm building anyway.

None of that means throwing upstream away: I read its Dockerfile first, so I
know what the app needs at runtime.

**Weak answer:** "Our policy says so." True somewhere, but it doesn't say
what the policy protects against.

**In the Podinfo run:** upstream's image was Alpine with `curl`, `netcat`,
an extra CLI (`podcli`) and a *named* user, so it would fail `runAsNonRoot`.
Ours is distroless, 11.5 MiB, `USER 65532:65532`, no shell
(`podinfo/Dockerfile`, MANIFEST §1).
</details>

---

## P2-1 — Your own image

### 1. "Switch off the dangerous endpoints" ★

> **Interviewer:** The app has debug and admin endpoints: one dumps
> environment variables, one crashes the process, one serves the profiler.
> You add a flag that switches them off by default. How do you know they're
> really off?

<details>
<summary><b>Model answer</b></summary>

By probing **every way into the process**, not just the main port. A flag
that removes routes from one router only protects that router. A typical
service has more doors: a separate metrics port, a gRPC port, an admin
port. Each has its own router, and each can expose the same functions.

**Key points:**

- **List the listeners first.** Read where the code starts servers and which
  handler each one uses, then probe each port with the flag off.
- **Watch for global registries.** Some libraries register themselves on a
  shared default. In Go, importing `net/http/pprof` adds `/debug/pprof/` to
  `http.DefaultServeMux`, so any server using that default serves the
  profiler, whatever your flag says.
- **Gate the twins.** If an endpoint exists over HTTP and gRPC, one flag
  must cover both.
- **Lock it with tests that can fail.** A test per door, and check it goes
  red when you put the bug back.

**Weak answer:** "I tested `/env` on port 9898 and got 404."

**In the Podinfo run:** the first version of the gate (P2-1a) passed every
probe on `:9898`. But the metrics server, on when `--port-metrics` is set
(the upstream chart sets it), served `http.DefaultServeMux`:
`/debug/pprof/` and `/debug/pprof/cmdline` answered **200** on `:9797` with
the flag off. Heap dumps from there can contain the JWT secret. The gRPC
`Env`, `Panic` and `Delay` services also ignored the flag. Fixed with a
dedicated mux and a gated gRPC registration, plus two tests that fail when
either fix is reverted. RCA
`platform/docs/rca/2026-10-15-p2-1a-gate-side-doors.md`.
</details>

**Follow-ups**

<details>
<summary><b>1.1</b> "Why 404 and not 403 for a switched-off endpoint?"</summary>

A 404 says the route doesn't exist, which is the truth: it was never
registered. A 403 says "it's here, you're just not allowed", which tells an
attacker it's worth trying harder, and keeps the dangerous code reachable
behind one check. It also makes the test unambiguous: with gorilla/mux, a
known path with the wrong method answers 405, an unknown path 404. So "404
on a wrong-method probe" proves the route is gone without ever running it.
</details>

<details>
<summary><b>1.2</b> "Why does the version endpoint show the commit only when CI builds the image?"</summary>

The program can't read Git at runtime; the build writes the commit into it
(in Go, `-ldflags -X …REVISION=<sha>`, fed by `--build-arg REVISION`). Build
without the argument and it says `unknown`. An OCI label on the image isn't
enough: registries can read it, the app can't.

**In the Podinfo run:** with `--build-arg REVISION=$(git rev-parse HEAD)`,
CUJ step 1 returned the full SHA for the first time.
</details>

### 4. "Walk me through your Dockerfile"

> **Interviewer:** Here's a Go service. Talk me through the Dockerfile you'd
> write for production, line by line, and why each choice.

<details>
<summary><b>Model answer</b></summary>

Two stages: one to build, one to run. Only the program crosses between them.

**Key points:**

- **Build stage on a full Go image, pinned by digest.** Copy `go.mod` and
  `go.sum` first and download modules, so that layer is cached until the
  dependencies change. Then copy the source and build.
- **A static binary.** `CGO_ENABLED=0` means no C libraries are needed at
  runtime, so the final image can be almost empty. `-trimpath` drops my
  machine's paths from the binary; `-ldflags "-s -w"` drops debug symbols;
  `-X …REVISION=$REVISION` writes the commit in.
- **Runtime stage on distroless `static`, pinned by digest.** It has CA
  certificates, time zones and `/etc/passwd`, and nothing else: no shell, no
  package manager.
- **A numeric non-root user**, `USER 65532:65532`.
- **Exec-form `ENTRYPOINT`** (`["/podinfo"]`), so the program is PID 1 and
  receives SIGTERM directly. Default flags go in `CMD`.
- **A `.dockerignore`** that keeps `.git`, `.env`, secrets, tests and build
  junk out of the build context.

**Weak answer:** "`FROM golang`, `COPY . .`, `go build`, `CMD`." One stage,
so the compiler, the source and everything in the folder ship to production.

**In the Podinfo run:** `podinfo/Dockerfile` is exactly this: `golang:1.27-bookworm`
→ `gcr.io/distroless/static-debian12:nonroot`, both by digest, BuildKit
cache mounts for modules and the build cache.
</details>

**Follow-ups**

<details>
<summary><b>4.1</b> "Why a number for the user, not a name?"</summary>

Kubernetes checks `runAsNonRoot` before the container starts, by looking at
the image's user. A name like `app` can't be checked without reading the
image's `/etc/passwd`, so the kubelet refuses to start it: "container has
runAsNonRoot and image has non-numeric user (app), cannot verify user is
non-root". A number can be checked directly. It
also makes the pod's `runAsUser` and file ownership (`--chown=65532:65532`)
line up without guessing.
</details>

<details>
<summary><b>4.2</b> "No shell in the image. How do you debug it at 3 a.m.?"</summary>

From the outside, not by getting a shell inside. Logs and metrics first. In
Kubernetes, `kubectl debug` attaches a temporary **ephemeral container** with
tools, sharing the pod's process namespace, so I get a shell next to the app
without shipping one in it. Locally, `docker run` a tools image on the same
network. The trade is deliberate: an attacker who gets code execution finds
no shell, no `curl`, nothing to download a second stage with.
</details>

<details>
<summary><b>4.3</b> "You pin base images by digest. How do they ever get security updates?"</summary>

On purpose, through a PR. A bot (Renovate or Dependabot) notices the tag now
points to a new digest and opens a PR that changes it. CI builds, scans and
runs the CUJ on that PR, and I merge it like any other change. Without the
pin, the image silently changes on the next build and nobody can say what's
in it; with the pin and no bot, it silently goes stale. You need both.
</details>

<details>
<summary><b>4.4</b> ★ "<code>ENTRYPOINT</code> or <code>CMD</code>: does it matter which holds the flags?"</summary>

Yes, because they're replaced differently. `ENTRYPOINT` is the program;
`CMD` is its default arguments. Kubernetes `args:` (and `docker run image
--flag`) **replace the whole `CMD`**. So if the image's defaults matter,
anyone who passes one extra flag must repeat all the others, or they're
silently gone. Prefer configuration that adds rather than replaces:
environment variables, a config file.

**In the Podinfo run:** the image's `CMD` set `--data-path=/tmp/data`, but
every doc mounted the writable volume at `/data`. The hardening test passed
only because it re-typed `--data-path` itself, replacing the `CMD`. Run as
shipped, `/store` failed with a 500. RCA
`platform/docs/rca/2026-10-15-p2-4-data-path.md`.
</details>

---

## P2-4 — Hardening audit

### 5. "Prove the image is hardened"

> **Interviewer:** You say your image is hardened. What would you show me to
> prove it?

<details>
<summary><b>Model answer</b></summary>

Tool output and a run, not the Dockerfile. Each check answers one question.

**Key points:**

- **hadolint:** is the Dockerfile written safely (pinned bases, no `latest`,
  exec-form)?
- **Trivy**, HIGH and CRITICAL, fixable only: does it ship known
  vulnerabilities?
- **dive:** what's actually in each layer? No `.git`, source, tests, keys or
  caches, and little wasted space.
- **Dockle:** does it follow container best practice (non-root, no setuid
  files, no secrets in env)?
- **`docker inspect`:** a numeric, non-root user.
- **A run the way production runs it:** read-only root filesystem, every Linux
  capability dropped, `no-new-privileges`, only the documented writable
  mounts. Then the user journey must pass.
- **Every listener probed:** debug and admin routes off on every port the app
  opens.

**Weak answer:** "Trivy is clean." That's one of seven questions.

**In the Podinfo run:** hadolint clean; Trivy 0 HIGH/CRITICAL after a
dependency bump; dive 99.999 % efficient with only the base, the binary and
`ui/`; Dockle no warnings; CUJ passes read-only with all capabilities dropped
(app README, "Hardening audit").
</details>

**Follow-ups**

<details>
<summary><b>5.1</b> ★ "Trivy finds a HIGH CVE in a library your app pulls in. What do you do?"</summary>

First, triage: is it in our code path, and is there a fixed version? For a
Go module with a fix, bump it (`go get module@fixed`), run the tests, rebuild,
rescan, and check the finding is gone **and** that it was present before, so
I know the scan really sees it. If there's no fix, or it truly doesn't
apply, record an exception with a reason and an expiry date, never a silent
ignore.

**In the Podinfo run:** `CVE-2026-78669` in `golang.org/x/net` v0.58.0, fixed
in v0.60.0. After the bump, Trivy on the new image showed 0; the image from
before the bump still showed the finding.
</details>

<details>
<summary><b>5.2</b> ★ "The read-only test passed. Can it still fail in production?"</summary>

Yes, if the test didn't run the image the way production does. A test that
adds its own flags or mounts tests those, not the image. Run it with the
image's **own default arguments**, plus only the writable mounts the
manifests will give it, with capabilities dropped. And try the paths that
write, not just the health check.

**In the Podinfo run:** see 4.4. Re-run as shipped, the documented `/data`
mount was the wrong folder.
</details>

<details>
<summary><b>5.3</b> "Dockle says you have no <code>HEALTHCHECK</code>. Add one?"</summary>

Not here. A `HEALTHCHECK` runs a command *inside* the container, and a
distroless image has no shell or `curl` to run. More importantly,
Kubernetes ignores `HEALTHCHECK` entirely; it uses its own liveness and
readiness probes, which call the app from outside. So I note it as a
deliberate INFO finding and design the probes in P3.
</details>

<details>
<summary><b>5.4</b> ★ "The template says <code>docker stop</code> must take under 2 seconds. Yours takes 3.4. Who's wrong?"</summary>

The rule, in this case. The time was a stand-in for the real question: does
SIGTERM reach the app, so it shuts down cleanly instead of being killed? The
real check is the **exit code**: 0 means it handled SIGTERM; 137 means Docker
gave up and sent SIGKILL after 10 seconds. An app that waits on purpose, so
the load balancer stops sending it traffic first, will take longer, and
that's correct, as long as the wait is known and written down.

**In the Podinfo run:** 3.4 s, exit 0: a deliberate 3 s readiness drain.
The template now checks the exit code (RCA
`platform/docs/rca/2026-10-15-p2-docker-stop-budget.md`).
</details>

---

## P2-5 — Size budget

### 6. "Why does image size matter?"

> **Interviewer:** Your image is 900 MB and it works. Why should anyone
> care?

<details>
<summary><b>Model answer</b></summary>

**Key points:**

- **Speed when it matters most.** Every new node pulls the image before the
  pod can start. During a scale-up or a node failure, a 900 MB pull is
  minutes of lost capacity.
- **Attack surface.** Size is mostly *stuff*: compilers, shells, package
  managers. Each one is code an attacker can use and a CVE you have to
  triage.
- **Cost.** Registry storage and network transfer, multiplied by every
  version and every node.

So set a budget per language and make CI **fail** the build when an image
crosses it. Without a gate, images only grow.

**Weak answer:** "Smaller is just tidier."
</details>

**Follow-ups**

<details>
<summary><b>6.1</b> ★ "Your size gate passed an image that was over budget. How?"</summary>

Rounding. Converting bytes to whole megabytes with integer division throws
away the fraction, so 11.5 becomes 11, and an 11 MB budget passes it.
Compare **bytes** with bytes. And test the gate right at its edge: a budget
just below the real size must fail, just above must pass. A gate only ever
tested far from its limit hasn't been tested.

**In the Podinfo run:** the template's script passed an 11.45 MiB image
against an 11 MiB budget. Ours compares bytes: 11 fails, 12 passes (RCA
`platform/docs/rca/2026-10-15-p2-5-size-gate-rounds-down.md`).
</details>

<details>
<summary><b>6.2</b> "The registry says 4 MB, <code>docker images</code> says 11. Which is the real size?"</summary>

Both. The registry stores layers **compressed**; that's what's transferred
on a pull. `docker image inspect` reports the **unpacked** size on disk. Pick
one for the budget and say which in the script. We measure unpacked,
because it's stable and doesn't depend on the compression the registry uses.
</details>

---

## P2-6 — Multi-architecture images (optional)

### 7. "Should we build for ARM too?"

> **Interviewer:** Graviton nodes are cheaper. Should your images support
> arm64?

<details>
<summary><b>Model answer</b></summary>

Only when something will run them. Decide from where the images will run,
not as a default.

**Key points:**

- **What it costs:** a multi-arch build produces one image per architecture
  under one name (a manifest list). Each one has to be scanned, signed and
  given an SBOM, and ideally tested on that architecture.
- **For Go it's cheap to build:** the compiler cross-compiles
  (`GOARCH=arm64`) with no emulation, and distroless has arm64 bases. Slow
  emulated builds are mainly a problem for languages that compile natively
  inside the image.
- **When it pays:** Graviton nodes (about 20 % cheaper), or developers on
  Apple Silicon who'd otherwise run amd64 images under emulation.

**In the Podinfo run:** amd64 only for v1. The nodes are `t3.large` (x86)
and the workstation is amd64, so arm64 images would have nothing to run on.
Recorded as a decision with a trigger to revisit (platform
`docs/questions.md` #14).
</details>

---

## P2-7 — Compose

### 8. "Why Compose if production is Kubernetes?"

> **Interviewer:** You'll deploy to Kubernetes. Why bother with a Docker
> Compose file?

<details>
<summary><b>Model answer</b></summary>

Because it's the cheapest place to run the *real image* with the *real
security settings* before Kubernetes adds its own moving parts.

**Key points:**

- **The same restrictions as production:** read-only root, all capabilities
  dropped, `no-new-privileges`, the same user. If the app breaks under them,
  I find out on my laptop in seconds.
- **The same secret handling:** the secret arrives as a mounted file, not an
  environment variable.
- **One command for everyone:** `docker compose up`, then the journey script.
  It's the baseline every later environment is compared against.
- **For multi-service apps, network segmentation:** put data stores on an
  internal network and **prove** the front end can't reach the database.

**Weak answer:** "For local development." True, but it misses that it's a
test of the production settings.

**In the Podinfo run:** one service, so segmentation had nothing to prove
(that lesson moves to Retail Store). The compose file runs the image with its
own `CMD` and adds only env vars and the secret file (`podinfo/compose.yaml`).
</details>

**Follow-ups**

<details>
<summary><b>8.1</b> "Why mount the secret as a file instead of an environment variable?"</summary>

Environment variables leak more easily. `docker inspect` and the Kubernetes
API show them, they're inherited by every child process, they turn up in
crash dumps, and some apps have a debug route that prints them. A file has
permissions, isn't copied into child processes, and can be swapped without
rebuilding anything.

**In the Podinfo run:** ADR-0006. The upstream app could print its whole
environment over HTTP (`/env`), now switched off; the JWT secret comes in as
`/config/config.yaml`.
</details>

<details>
<summary><b>8.2</b> "Your Compose file has no healthcheck. Isn't that a gap?"</summary>

A Compose healthcheck runs a command inside the container, and this image
has no shell or `curl` for it to run. Options: a tiny health-check mode in
the binary itself, a sidecar that checks from outside, or nothing, if
nothing depends on it. With one service and no `depends_on`, there's nothing
to wait for, and the journey script waits on `/readyz` before it starts. In
Kubernetes the probes call the app from outside, so no in-image tool is
needed.
</details>

<details>
<summary><b>8.3</b> "What's the difference between <code>depends_on</code> and <code>depends_on: condition: service_healthy</code>?"</summary>

Plain `depends_on` only waits for the other container to **start**, not to
be ready. A database container starts in a second and accepts connections
much later, so the app starts, fails to connect and crashes or retries.
`service_healthy` waits until the other service's healthcheck passes. That
needs a real healthcheck on the dependency, and the app should still retry
on its own, because in Kubernetes there's no start order at all.
</details>

---

## P2-8 — Script the CUJ

### 2. "The check says 401, the client sees 200" ★

> **Interviewer:** Your journey script checks that a forged token is
> rejected, and it passes. A colleague points out the load balancer logs show
> HTTP 200 for those requests. How can both be true?

<details>
<summary><b>Model answer</b></summary>

The script read the wrong thing. Some apps put an error code **inside the
JSON body** and still answer **HTTP 200**. A check that parses the body sees
`401`; everything outside the app sees success.

**Key points:**

- **Clients and load balancers act on the status line**, not the body. A
  retry policy, a health check or a WAF rule will treat these errors as fine.
- **Metrics count status codes.** An error-rate SLO built on them never
  moves, so alerts never fire.
- **Assert the HTTP status** in every check (`curl -w '%{http_code}'`), and
  the body as well where it matters.
- **Fix the app when it lies**, in one place, with a test that fails if the
  bug returns.

**Weak answer:** "The body says 401, so it's rejected."

**In the Podinfo run:** upstream's `ErrorResponse` always wrote 200, for all
18 error paths; an upstream test even asserted 200 for an invalid token. The
fork now sends the real status, the test expects 401, and reverting the patch
makes both tests fail. RCA
`platform/docs/rca/2026-10-15-p2-8-error-status-200.md`.
</details>

**Follow-ups**

<details>
<summary><b>2.1</b> "How do you know your journey script can fail at all?"</summary>

Break one thing at a time and check it fails **only** the step that covers
it: run without the real secret (the forged-token step), switch the debug
routes on (the route probes), build without the commit SHA (the version
step), stop the app (readiness). A check that has never failed hasn't been
tested.

**In the Podinfo run:** each of those four drills failed exactly its own step.
</details>

<details>
<summary><b>2.2</b> "Your journey script has a JWT hard-coded in it. Isn't that a leaked secret?"</summary>

Not this one, and the reason matters. It's signed with the app's
**published default** secret, so anyone can make the same token. Its whole
job is to prove we're *not* running that default: a real deployment must
reject it with 401. It's marked as allowed for the secret scanner, with the
reason in a comment, so the exception is visible and reviewed rather than
the rule being switched off. A token signed with a real secret would be a
leak, and would be rotated.

**In the Podinfo run:** `scripts/cuj.sh`, `DEFAULT_SIGNED_TOKEN` (claims
`name=cuj-forged`, `iss=podinfo`, expiry in 2100), `# gitleaks:allow`.
</details>

---

## P2-9 — Measure resources

### 9. "What CPU and memory does it need?"

> **Interviewer:** You're about to write the Kubernetes manifest. What do
> you put in `resources.requests`, and where do the numbers come from?

<details>
<summary><b>Model answer</b></summary>

From a measurement, never from the upstream chart or a guess. Run the user
journey in a loop for a while and record CPU and memory, then set requests
a bit above what you saw.

**Key points:**

- **Requests are for scheduling:** Kubernetes reserves them on a node. Too
  low and pods get packed onto nodes that can't serve them; too high and you
  pay for idle capacity.
- **A memory limit protects the node**, so set one, with headroom above the
  peak. Go can be told about it (`GOMEMLIMIT` at about 90 % of the limit), so
  the garbage collector works before the kernel kills the pod.
- **No CPU limit, usually:** a CPU limit throttles the app even when the
  node has spare CPU, which shows up as mysterious latency.
- **Say what the number is.** One client in a loop gives a first sizing
  number, not a capacity number. Load testing (P10) finds the real limits.

**Weak answer:** "The chart says 16Mi, so 16Mi."

**In the Podinfo run:** 10 minutes, 2,310 journey runs, 0 failures, about
65 requests per second from one client. CPU about 36m (3.6 % of a core);
memory working set peaked at 31 MiB. The upstream chart requests 16Mi,
below even the starting working set. Provisional request `100m`/`64Mi`,
re-checked in P3 (platform `docs/sizing-and-cost.md`).
</details>

**Follow-ups**

<details>
<summary><b>9.1</b> "<code>docker stats</code> says 31 MiB, the app's own metric says 53 MiB. Which do you size from?"</summary>

They count different things. The app's resident memory (RSS) includes
file-backed pages, like the mapped binary, that the kernel can drop and
reload. The **working set** is what can't be reclaimed, and it's what
`docker stats`, `kubectl top` and the kubelet's eviction decisions use.
Size requests from the working set; keep the limit above both, and watch
the trend over time, because steady growth means a leak.
</details>

<details>
<summary><b>9.2</b> "Memory climbed from 17 to 28 MiB in the first minute. Is that a leak?"</summary>

Not on its own. A leak is growth that never stops. Warm-up is normal: caches
fill, the runtime grows its heap to match the load, then it levels off.
The test is the shape over time. In the Podinfo run it reached 28.5 MiB at
73 seconds and stayed around 30 MiB for the remaining nine minutes, which is
a plateau, not a leak.
</details>

---

## P2-10 — The playbook

### 10. "The second service"

> **Interviewer:** It took you two days to containerise the first service.
> How do you make the next one take an hour?

<details>
<summary><b>Model answer</b></summary>

Write down what you did, in order, with a check for each step, and the
mistakes, so nobody has to rediscover them.

**Key points:**

- **The steps in order:** read the code, list env vars and ports, find the
  health endpoint, pick base images, write the Dockerfile, `.dockerignore`,
  run it non-root and read-only, check SIGTERM, scan, add it to Compose, run
  the journey script.
- **A check per step**, so "done" means the same thing to everyone.
- **The failures and their fixes**, from this service: they're the part
  that saves the most time.
- **Prove the playbook works:** someone else containerises a service using
  only the document. Where they get stuck, the playbook is wrong.

**In the Podinfo run:** platform `docs/playbooks/containerise-a-service.md`:
11 steps and 11 failures with fixes. Its own proof waits for Retail Store,
where someone follows it for a service it wasn't written from.
</details>

---

## How we work

### 11. "Prove no secret is baked into the image" ★

> **Interviewer:** A developer had a `.env` file with real credentials in
> the repo folder when the image was built. How do you prove they're not in
> the image?

<details>
<summary><b>Model answer</b></summary>

Search every layer, and prove the search works.

**Key points:**

- **Plant a canary:** put a fake, random secret in the places that worry
  you (`.env`, a secrets folder) and build.
- **Search every layer of the final image**, plus the build stage: a
  multi-stage build's first stage often does `COPY . .`, and if it's ever
  pushed or cached, the secret travels with it. Check `docker history` too,
  because build arguments and env values show up there.
- **Run a control:** build an image that deliberately copies the canary,
  and check the same search finds it. A search that finds nothing has only
  proven something if it could have found something.
- **Then explain why it's absent:** `.dockerignore` keeps the files out of
  the build context entirely.

**Weak answer:** "We have `.env` in `.dockerignore`." That's the reason it
*should* be absent; it's not proof.

**In the Podinfo run:** a random `AKIA…` key in `.env` and `.secrets/`: zero
hits in every layer of the final image and of the build stage, zero in
`docker history`; the control image that copies it was found (platform
backlog, Phase 2 Prove it).
</details>

### 12. "Your teammate's audit says it all passed" ★

> **Interviewer:** A teammate, or an AI agent, hands you a PR: Dockerfile,
> hardening audit, all checks green, results in the README. Do you merge it?

<details>
<summary><b>Model answer</b></summary>

Not on the README's word. I reproduce the key claims myself: build the
image from the PR, run the tests, run it the way production will, and try
to break it. A results table records what someone saw; it isn't a test.

**Key points:**

- **Rebuild and rerun**, from the PR's exact commit, not my local copy.
- **Look for what wasn't tested:** other ports, the image's own defaults,
  the failure paths. That's where the bugs were.
- **Check the checks:** does each test fail when the bug is put back? Does
  each drill fail for the right reason?
- **Fix on the PR, with a test**, and record the lesson so the next review
  looks there first.

**In the Podinfo run:** independent review of green PRs found three real
bugs: the profiler still reachable on the metrics port (case 1), the
`/tmp/data` vs `/data` mismatch (4.4), and errors sent as HTTP 200 (case 2).
Each became a fix, a test that fails if it returns, and an RCA.
</details>
