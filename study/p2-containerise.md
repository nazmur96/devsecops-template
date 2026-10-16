# Study bank — Phase 2: Containerise

Interview practice for [Phase 2](../template/02-containerise.md). Each
**case** is how a scenario interview actually runs: the interviewer sets a
scene and asks an open question, then pushes with follow-ups to see how deep
your understanding goes.

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

This bank grows as Phase 2 runs.

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
