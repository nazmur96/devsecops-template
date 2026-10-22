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

**New to Phase 3?** Read [Phase 3 explained simply](p3-explained.md) first,
then come back here to practise.

`★` = came from a real finding or mistake in a run.

**Adding a question:** add a follow-up under the case it belongs to (next
number, e.g. `1.4`), or a new case at the end of its section's numbering.
Phrase it as something an interviewer would say. Write the model answer the
way you would say it out loud: short paragraphs or **Key points:** bullets,
never a table. Cite the file/line, ADR or RCA that proves it. Never renumber
existing cases or follow-ups.

**Case numbers are IDs.** Cases 1-4 were written first (P3-7 to P3-10);
cases 5-7 cover P3-1 to P3-6. Sections follow the phase order, so the
numbers don't run in order down the page.

---

## P3-1 — The local cluster

### 5. "Why not just learn Kubernetes on EKS?"

> **Interviewer:** You spent a whole phase on a cluster that runs on a
> laptop. Why not go straight to EKS? And why three control-plane nodes on
> a laptop?

<details>
<summary><b>Model answer</b></summary>

Because a local cluster removes two of the three hard things at once: AWS
networking and IAM, and a running bill. What's left is Kubernetes itself,
and that's where the drills are: stop nodes, cut the network, kill pods
mid-request. Free and rebuilt in minutes, you do them ten times instead of
once. Then you re-verify on EKS knowing what "working" looks like.

**Key points:**

- **Same minor version** as kubectl and EKS, or a manifest can pass here
  and fail there.
- **Three control planes** so etcd keeps quorum: two of three still form a
  majority, so you can drill losing one. With two, losing one freezes the
  cluster.
- **PSA labels from day one**, so the security profile is learned cheaply.
- **Rescheduling isn't instant:** a node that goes silent is presumed alive
  for 300 s (the default `not-ready`/`unreachable` toleration). Expect that
  number in a drill and explain it.

**In the Podinfo run:** a stopped worker's pods moved after **355 s**
(300 s + ~55 s); with one control plane stopped, the API still answered.

**Weak answer:** "Local is cheaper." True, but it misses the point: it's
where you can afford to break things.

</details>

**5.1 — "Your laptop resolves the hostname to IPv6 first. You add an IPv6
port mapping to fix it. What happens?"** ★

<details>
<summary><b>Model answer</b></summary>

Possibly the opposite of a fix. Without an IPv6 mapping, the IPv6 attempt
is **refused** at once and the client falls back to IPv4, which works. With
a mapping, Docker *accepts* the IPv6 connection and forwards it to an
address an IPv4-only cluster doesn't serve, so the connection hangs and the
client never falls back. Test both before adding a "fix".

**In the Podinfo run:** IPv4-only mapping → 200 via fallback; adding
`listenAddress: "::"` → 000. Documented in `kind/cluster.yaml`.

</details>

---

## P3-2 / P3-3 — Manifests and the workload

### 6. "Prove your readiness probe works" ★

> **Interviewer:** You want to show that an unready pod gets no traffic.
> You deploy a copy of the app with a flag that makes it report not-ready,
> check that it isn't in the Service's endpoints, and call it proven. Is it?

<details>
<summary><b>Model answer</b></summary>

Only if the Service would have selected that pod in the first place. If the
copy has different labels from the ones the Service selects, it was never
going to appear in its endpoints, ready or not. "Not in the endpoints" is
then true either way, and the test proves nothing.

**Key points:**

- Put the test pod **behind a Service that selects it**, and check the
  endpoint reads `ready=false`.
- Then **remove the flag** and watch it flip to `ready=true`. A test that
  sees both states can fail; one that sees only the expected state may not.
- Same idea for liveness: the restart count must climb.
- And keep liveness free of dependencies: if it checks the database, one
  blip restarts every pod at once.

**In the Podinfo run:** the first readiness test used a copy with another
`instance` label: never behind the real Service. The review rebuilt it with
its own Service and the flip both ways.

</details>

**6.1 — "How do you check every object has your standard labels?"** ★

<details>
<summary><b>Model answer</b></summary>

With a script that **derives** the list of objects from what you deploy
(render the kustomize overlay, check each object it produces, plus the ones
your scripts create, like the Secret). A hand-written list goes stale with
the first new manifest, and the check keeps passing while it checks less
and less.

And keep the **selector** to labels that never change (`name`, `instance`):
selectors are immutable, so anything else there blocks later changes.

**In the Podinfo run:** the first `check-labels.sh` had a hardcoded list;
the review made it render the overlay, and fail naming the object and the
missing label.

</details>

---

## P3-6 — Gateway API and TLS

### 7. "Who owns the route?" ★

> **Interviewer:** Your platform team installs the Gateway and, in the same
> script, applies every app's HTTPRoute. It worked on the existing cluster.
> What's wrong with it?

<details>
<summary><b>Model answer</b></summary>

Ownership and order. The Gateway (listeners, certificates, which
namespaces may attach) is the platform's. A route (this hostname, these
paths, this backend) is the app's: it changes when the app changes, and it
belongs in the app's manifests. Put it in the platform script and a fresh
cluster breaks: the platform installs first, the app's namespace doesn't
exist yet, and the route can't be created. It also hides the route from
the app's own checks (labels, review).

**Key points:**

- Routes **with the app**; the Gateway allows attachment by namespace label
  (`gateway-access: public`), and a namespace without it is refused.
- Routes as an **allow-list**: exact paths the user journey needs, so
  everything else gets a 404 at the door.
- The front door's own namespaces get PSA too; a server dry-run of the
  label shows what would need an exception.

**In the Podinfo run:** the from-scratch order failed until podinfo's
`HTTPRoute` moved to `k8s/overlays/dev`; the negative (an unlabelled
namespace) came back `NotAllowedByListeners`.

</details>

**7.1 — "How do you prove certificates renew?"** ★

<details>
<summary><b>Model answer</b></summary>

Shorten the lifetime and watch renewal happen **on schedule**, not once.
Deleting the certificate's Secret only makes cert-manager reissue it one
time, which proves issuance, not renewal. With a Gateway-managed
certificate, the duration annotations are read only when cert-manager
creates the Certificate, so: annotate the Gateway, delete the Certificate
and its Secret, then watch `status.revision` climb on its own (a 1 h
lifetime renewed 59 min early renews every minute). Restore afterwards.

**In the Podinfo run:** revision went 1 → 2 → 3 unattended, and Envoy
served each new certificate.

</details>

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
