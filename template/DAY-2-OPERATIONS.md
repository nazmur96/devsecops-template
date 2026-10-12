# Day-2 operations: running it after it's built

The phases teach you to **build**. This file is about **running** it, which
is where most of a platform engineer's time goes. Every Day-1 build creates a
permanent Day-2 job: something to patch, watch, restore or pay for.

Use it three ways:
1. Put the **operating rhythm** on a calendar once P10 is done ([P10-10](10-observe-and-operate.md#p10-10-day-2-operations)).
2. Use the **troubleshooting path** and the **common incidents** whenever something breaks.
3. Run the **drills** on schedule, because reading a runbook isn't the same as having run it.

---

## 1. The operating rhythm

| When | What | Template link |
|---|---|---|
| **Daily** | Triage overnight alerts: close or escalate each one | P10-4, P10-8c |
| | Argo CD: anything `OutOfSync` or `Degraded`? | P8-3 |
| | Error-budget burn on the CUJ services | P10-4 |
| | Cost anomaly feed | P10-10c |
| | Auto-remediation actions taken overnight: were they right? | P10-5c |
| **Weekly** | Patch window: node AMIs (Karpenter drift), add-on versions, base-image rebuilds | P6-6, P7-7 |
| | Merge Renovate/Dependabot PRs that passed the gates | P7-7 |
| | Drift report: anything `terraform plan` didn't expect | P7-6 |
| | New security findings: Security Hub, Inspector/ECR, Trivy, Falco | P10-5, P10-5b |
| | End-of-life issues opened by the EOL workflow | P10-8b |
| **Monthly** | **Restore drill:** restore something from backup and time it | P10-9 |
| | Access review: remove unused permissions (IAM Access Advisor) | P5-8 |
| | Rightsizing: requests vs actual usage, then update requests and HPA targets | P10-10c |
| | Five-minute cost audit | P10-10c |
| | Alert-quality review: delete or tune alerts that needed no action | P10-8c |
| | Post-mortem action items: close or re-plan overdue ones | P10-8 |
| **Quarterly** | One Kubernetes minor-version upgrade, non-prod first | P10-10d |
| | DR test: full rebuild or failover, with measured RTO/RPO | P10-9, P10-11 |
| | Game day: unannounced chaos, using only the real runbooks | P10-7 |
| | Policy review: Kyverno rules, NetworkPolicies, SLO targets, suppressions past their date | P8-6, P10-4 |

---

## 2. The troubleshooting path

**Walk the request path in order. Never guess, never skip a hop.** Most
outages are one hop *earlier* than where people start looking, and the most
common mistake is starting at the app.

```
user → DNS → ALB (+WAF) → target group → Gateway / HTTPRoute → Service → Endpoints → Pod → app → dependency
```

At each hop ask: **does it resolve? is it reachable? is it healthy?**

| Hop | Check |
|---|---|
| DNS | `dig shop.example.com`: right answer? ExternalDNS logs if not (P9-4) |
| ALB / WAF | Target group health in the console or `aws elbv2 describe-target-health`; WAF sampled requests for blocks |
| Gateway / HTTPRoute | `kubectl get gateway,httproute -A`: route `Accepted` and `ResolvedRefs`? Hostname and path match? |
| Service | `kubectl get endpointslices -l kubernetes.io/service-name=<svc>`: **empty means the selector matches no ready pod** |
| Pod | `kubectl describe pod` **before** `kubectl logs`: status, restarts, readiness, events |
| App | Logs **at the error's timestamp**, not the current tail; then the trace (P10-3) |
| Dependency | Test **from inside the pod**: DNS, then the port, then credentials (P4-9's gate order) |

---

## 3. The eighteen common incidents

Each: **first check → likely causes → fix.**

1. **Site down / 5xx spike** → target group health, then Service endpoints → every pod failing readiness; endpoints empty after a label change; a dependency down → **roll back first** (`git revert`, P8-8), diagnose second.
2. **CrashLoopBackOff** → `kubectl logs --previous` (the *previous* container) → missing config or secret; failing migration; bad image; startup crash → fix the startup error; check the mounted Secret actually has the keys (ESO `SecretSynced`?).
3. **Pod `Pending`** → `kubectl describe pod`, events at the bottom → not enough CPU/memory; no node tolerates its taint (P6-5b); no NodePool matches; PVC unbound; quota hit (P6-5) → the scheduler's message names the exact constraint.
4. **OOMKilled** → memory limit vs real usage over time → limit below the working set; a leak; JVM heap above the container limit (P2-2) → raise the limit only after ruling out a leak.
5. **AccessDenied from AWS** → CloudTrail for the exact event → SCP; boundary; identity policy; resource or KMS policy; wrong ServiceAccount → Pod Identity association → **name the layer first** (P5-8).
6. **Can't reach the database** → from inside the pod: DNS, then port → security group; NetworkPolicy default-deny without an allow; wrong endpoint; expired credentials → walk the gates (P4-9); don't assume it's the password.
7. **Latency spike** → one service or all of them? Traces, not logs → slow dependency; **connection-pool exhaustion**; noisy neighbour; a recent deploy; a missing index after a migration → line it up with the deploy timeline first: most regressions are releases.
8. **Image pull failure** → the exact error: auth, not-found, or rate limit → node role can't read ECR; wrong digest; Docker Hub rate limit (use the pull-through cache, P4-8) → confirm the digest exists and the puller can read it.
9. **Certificate expired** → ACM status (ALB), or `kubectl get certificate` (cert-manager) → DNS validation record missing; registrar NS not pointing at Route 53; Let's Encrypt rate limit → read the `Order`/`Challenge` objects (P9-4b); alert on expiry < 14 days.
10. **DNS resolution failing in pods** → CoreDNS pods and logs → CoreDNS under-resourced; NetworkPolicy blocking port 53; `ndots` lookups multiplying queries → test from a debug pod in the same namespace; NodeLocal DNSCache (P6-5b).
11. **Node `NotReady`** → `kubectl describe node` conditions → disk/memory pressure; kubelet down; CNI unhealthy; EC2 problem → **cordon, drain, replace** (Karpenter does this). Don't debug a broken node in place mid-incident.
12. **PVC won't attach** → PVC events and StorageClass → **the volume is in a different AZ than the pod** (EBS is zonal); CSI driver issue → the pod must run in the volume's AZ (`WaitForFirstConsumer`, P4-6).
13. **Argo CD stuck `OutOfSync`** → the diff Argo CD shows → a mutating webhook or controller rewrites a field (Kyverno, an HPA on `replicas`) → put the field in Git or `ignoreDifferences` on that field only (P8-6); remove `replicas:` when an HPA exists (P6-2).
14. **The bill jumped 40 %** → cost by namespace and tag, then by AWS service → forgotten debug Deployment; autoscaler runaway; **NAT or cross-AZ transfer**; orphaned volumes, snapshots, load balancers; log volume → run the five-minute audit (P10-10c).
15. **S3 403 although the IAM policy allows it** → the **bucket policy**, the **KMS key policy**, the **VPC endpoint policy** → an explicit deny always wins → all three must agree (P5-8).
16. **Traffic gets in but responses never come back** → route tables on **both** sides → asymmetric routing after adding a VPC, peering or TGW attachment → attaching isn't routing: add the return route (P12).
17. **A long CI job dies with an expired-token error** → how long it ran: exactly 1 h is the tell → the assumed-role session expired → raise the role's max session duration or refresh credentials mid-job (P5-2, P5-8).
18. **"Deployed" but the old code is live** → the running **digest** (`kubectl get pod -o jsonpath='{..image}'`), not the tag → a mutable tag; `IfNotPresent` with a reused tag; wrong environment folder → digests only, never reused tags (P7-5).

---

## 4. The drill calendar

| Drill | How often | What it proves | Ticket |
|---|---|---|---|
| Restore a namespace from backup, timed | Monthly | Backups are real; you know your RTO | P10-9 |
| Kill a node under load | Monthly | Rescheduling, spread and PVCs behave | P6-7, P10-7 |
| Break a security group or NetworkPolicy, find it via the troubleshooting path | Monthly | You can walk the path under pressure | §2 above |
| Make a bucket public; watch auto-remediation | Monthly | The automation still works and is still right | P10-5c |
| Change something in the console; let drift detection catch it; fix via PR | Monthly | The drift loop closes | P7-6 |
| Rotate a secret with zero downtime | Quarterly | Rotation isn't an outage | P5-5 |
| One alert → runbook → fix → post-mortem | Every real incident | The response chain works end to end | P10-8, P10-8c |
| Upgrade non-prod one minor version | Quarterly | Deprecations caught before prod | P10-10d |
| Full rebuild from Git + backups | Quarterly | Nothing lives only in someone's head or the console | P10-9 |

**Rules for drills:** schedule them like real work or they won't happen; time
everything and write it down; run them from the written runbook and fix the
runbook when it's wrong; move a drill from non-prod to prod only once
non-prod is boring.
