# Study bank — Phase 3: Local Kubernetes

Interview practice for [Phase 3](../template/03-local-kubernetes.md). Each
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

---

## P3-7 — NetworkPolicy

### 1. "Prove your NetworkPolicies work" ★

> **Interviewer:** You've added a default-deny policy and a few allows to a
> namespace. Your test pod in that namespace can't reach the app any more,
> so you say the policies work. What did that test actually prove?

<details>
<summary><b>Model answer</b></summary>

Less than it looks. Default-deny blocks **egress** too, so a test pod
inside the same namespace can't send anything. Its request fails on the
client's side, whatever the app's ingress rule says. The test would pass
even if the app accepted everyone.

**Key points:**

- **Control first:** before any policy, the same request must succeed.
  Otherwise "it fails" may be a typo in the URL.
- **Isolate the rule under test:** send from a pod whose egress is open, in
  a namespace with no policies, or give the probe a temporary allow-all
  egress, and check that the egress really is open (reach something else).
- **Then enforcement:** with only default-deny, that probe must time out. If
  it gets through, the CNI ignores NetworkPolicy.
- **Script it**, so every later change reruns it.

**In the Podinfo run:** `scripts/check-netpol.sh` opens the in-namespace
probe's egress with a temporary policy and checks it can reach the API
server before it asserts podinfo is unreachable. kindnet enforced
default-deny: DNS failed, probes timed out, the Gateway returned 000.

**Weak answer:** "The pod can't connect, so it's blocked." Blocked by what?

</details>

**1.1 — "Your allow for the ingress controller passes every test. Is it
right?"**

<details>
<summary><b>Model answer</b></summary>

Only if the tests include the case it could get wrong. The classic slip is
writing the namespace selector and the pod selector as **two** `from`
entries. That reads as "any pod in the controller's namespace, **or** pods
with this label in my namespace", so every pod in the controller's
namespace is let in. Probes from other namespaces still fail, and the CUJ
still passes, so a test suite without a probe from a **non-proxy pod in the
proxies' namespace** stays green.

**In the Podinfo run:** splitting the selectors left checks 1-4 green;
check 5 (a busybox in `envoy-gateway-system`) was added because of it, and
it failed as it should.

</details>

---

## P3-8 — Zero-downtime drills

### 2. "Your drain drill shows zero errors" ★

> **Interviewer:** You run a request loop while draining a node, and every
> request returns 200. Your app has two replicas spread across nodes. What
> else could break during a drain, and how do you know the drill itself
> works?

<details>
<summary><b>Model answer</b></summary>

Two things: everything on the request path, not only the app, and the
drill's ability to see a failure at all.

**Key points:**

- **The front door is a workload too.** Ingress or Gateway proxies often
  default to one replica. Drain its node and every request drops while the
  app is healthy. Give it ≥ 2 replicas, spread, and a PDB.
- **See a failure first:** one control run with the graceful-shutdown
  pieces removed must show errors. A loop that can't see a drop proves
  nothing.
- **The action must succeed:** if the drain is refused (a bare pod, a PDB
  that can't be met), nothing moved, and a clean tally is meaningless. The
  script must fail when the drain fails.
- Drain **every** node in turn, so each placement is covered.

**In the Podinfo run:** the first drains dropped 14-20 requests each with
podinfo fine: Envoy Gateway's single proxy was on the drained node. Two
replicas + PDB fixed it (0 errors across 6 drains). One drain was refused by
a leftover probe pod, and the first version of the script still printed
all-200. Controls: no `preStop` + `--level=debug` → 7-8 errors per restart.

</details>

**2.1 — "Debugging needs debug logs. Any reason not to turn them on during
an incident?"**

<details>
<summary><b>Model answer</b></summary>

For this app, yes: podinfo's debug level skips its 3 s drain sleep, so
turning it on brings back dropped requests on every rollout, and it's also
the only stderr log that carries the trace ID. The general lesson: know
what a log level changes besides the logs. Prefer a log stream that has the
fields you need at the normal level (podinfo's OTLP log export carries
`trace_id` at info).

</details>

**2.2 — "You undid a drill by re-applying the manifests. Is the cluster
back to the declared state?"** ★

<details>
<summary><b>Model answer</b></summary>

Not necessarily. `kubectl apply` merges three ways: it removes only fields
that were in its last-applied configuration. A variable added with
`kubectl set env` was never there, so it survives every re-apply. The fix
for drills is removing it the same way (`kubectl set env … VAR-`); the fix
for the class is a reconciler that diffs live state against Git and heals
it (Argo CD self-heal, P8).

**In the Podinfo run:** `PODINFO_RANDOM_ERROR=true` survived
`scripts/deploy.sh` after the SLO drill, and was caught only by reading the
live env.

</details>

---

## P3-9 — Inner loop

### 3. "Developers want to run a service locally against the cluster" ★

> **Interviewer:** The team wants mirrord (or Telepresence) so a developer
> can run one service on their laptop with cluster traffic. What do you
> check before saying yes?

<details>
<summary><b>Model answer</b></summary>

What it needs in the cluster, and whether it really works for our stack.

**Key points:**

- **Privilege:** the agent joins the target pod's namespaces, so it's a
  privileged pod. Where does it run? If any namespace has no Pod Security
  labels (often `default`), it runs there silently. Label `default`
  `restricted`, give the tool its own namespace with an ADR, and create
  that namespace only while it's used.
- **Language fit:** mirrord loads with `LD_PRELOAD`. A static Go binary
  ignores it, and `go run` is skipped as a build tool, so the process runs
  unhooked without an error. Build a dynamically linked binary.
- **Prove it end to end:** the local process gets the pod's Secret (a token
  from a cluster pod validates locally), serves real traffic (responses
  show the local build), and traffic returns after Ctrl-C.

**In the Podinfo run:** the agent first ran in `default`; after labelling
it, PSA rejected the agent (hostPID, allowPrivilegeEscalation), and ADR-0012
records the `mirrord` namespace. `go run` and a static build both ran
unhooked; a `zig cc` build worked. `fs.mode: local` with `read_only`
`/config` left podinfo on the default secret (401s); `read` worked.

</details>

---

## P3-10 — Observability

### 4. "Wire up trace-to-logs" ★

> **Interviewer:** Grafana is set up with Tempo and Loki, and the "logs for
> this trace" button is configured. It shows nothing. Where do you look?

<details>
<summary><b>Model answer</b></summary>

At the logs, before the config: does any log line carry the trace ID, and
in what form?

**Key points:**

- Many apps log the trace ID only in some lines or at some levels. Find the
  stream that has it; if none does, say the link can't work instead of
  faking it.
- Check the **format**: the link matches text. Tempo's search API drops
  leading zeros from trace IDs, so an equality match on a 32-character ID
  misses some traces; match by substring.
- Then the collector: are those logs shipped at all, and with the field
  kept (structured metadata for OTLP)?

**In the Podinfo run:** stderr had the trace ID only in the debug request
log (`pkg/api/http/logging.go`), which is off at info. Every OTLP log
record had it, because the OTel log core ignores the zap level. A slow
`/version` trace found its log line by substring.

</details>
