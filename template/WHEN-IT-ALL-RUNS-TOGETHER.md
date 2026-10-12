# When it all runs together: the system view

Every ticket in this template is a sound decision on its own. The risk that
no single ticket shows is that **the failures live in the interactions
between tools**: three policy engines all "working as designed" while traffic
is still blocked, or two controllers both changing the same field.

Read this after P8, and again before you add any optional tool. The rule it
teaches: **every tool you add needs a written "who wins" statement relative
to the tools already there.** Write it when you add the tool, not during the
incident.

---

## 1. "Why is this request blocked?" Policy lives in many places

By the end of P10, several independent layers can each say no:

| Order to check | Layer | Tool | Decides | Symptom when it says no |
|---|---|---|---|---|
| 1 | Account | SCPs (P12), IAM (P5) | Which AWS API calls are possible | `AccessDenied` in CloudTrail |
| 2 | Admission | PSA + Kyverno (P8-5, P8-6) | Which manifests may be created | `kubectl apply` / Argo CD sync error naming the policy |
| 3 | Kubernetes API | RBAC, Access Entries (P4-4, P5-1) | Who may change what | `Forbidden` |
| 4 | Scheduling | Taints, quotas, PriorityClass (P6-5, P6-5b) | Where and whether a pod runs | Pod `Pending` with a reason |
| 5 | Network L3/L4 | NetworkPolicy (P8-7), security groups (P4-3) | Which pod may reach which pod or database | **A timeout with no error anywhere** |
| 6 | Edge | WAF (P9-6) | Which requests reach the app | 403 from the ALB |
| 7 | Mesh *(if used)* | Linkerd/Istio policy (P8-11) | Which service may call which, by identity | 403 / connection reset |
| 8 | Runtime | Falco (P10-5) | Detects (doesn't block) what a running container does | An alert |

**The worst case is layer 5.** Every other layer reports "allowed", and the
traffic is silently dropped, so the app just times out. That's why P3-7 and
P8-7 make you **prove** denies with negative tests, and why seeing dropped
flows matters (network policy agent logs, or Hubble on Cilium, P3-7b).

**Keep one table** in `docs/` that says, for each allowed traffic path (from
the P0-2 dependency diagram), which layers must allow it. When NetworkPolicy
and the mesh's authorization both encode "who may reach the database", they
can drift apart. The table is how you notice.

---

## 2. How many identities does one pod have?

| Identity | From | Used for |
|---|---|---|
| Kubernetes ServiceAccount | P5-4b | Who it is to the Kubernetes API, and the anchor for everything below |
| AWS role via Pod Identity | P5-4 | Who it is to AWS APIs (S3, SQS, Secrets Manager) |
| Mesh identity *(if used)* | P8-11 | Who it is to other services, for mTLS |
| Cilium identity *(if used)* | P3-7b | Label-derived identity for network policy |

Each one can expire, rotate or be misconfigured **independently**. A pod can
be allowed by NetworkPolicy and denied by AWS at the same time, which looks
like a random intermittent failure unless you know to check both. **Keep the
count small:** this template stops at two by default (ServiceAccount + Pod
Identity). Each optional tool that adds one should say so in its ADR.

---

## 3. Who owns which routing decision?

A request can pass through several hops: **WAF → ALB → Gateway/HTTPRoute →
(mesh) → pod.** Each hop adds latency and a place where a rule *could* go.
Without a convention, the same thing (say, a canary split) ends up done three
different ways by three people.

| Decision | Owned by exactly | Not here |
|---|---|---|
| Block bad requests, rate limits | **WAF** (P9-6) | App code, Gateway |
| TLS for the public domain | **ALB + ACM** (P9-4) | Pods (unless an in-cluster gateway, P9-4b) |
| Host and path routing | **HTTPRoute** (P9-3) | ALB rules written by hand |
| Canary traffic split | **Argo Rollouts → HTTPRoute weights** (P8-8) | Mesh routes, manual weights |
| Retries, timeouts between services | **App** (or mesh if adopted, P8-11) | Both at once: retries multiply |
| Service-to-service allow/deny | **NetworkPolicy** (+ mesh identity if adopted) | Security groups between pods |

Write this table when you add the **second** hop, not the fifth. Latency
also adds up hop by hop, and you only see it in load tests (P10-7).

---

## 4. Who is driving the replica count right now?

| Controller | Decides | Conflict to avoid |
|---|---|---|
| HPA (P6-2) | Replicas, from CPU or a metric | `replicas:` in Git **and** an HPA: every sync fights it. Remove `replicas:` |
| KEDA (P6-2, opt.) | Replicas, from queue depth; can scale to 0 | KEDA creates its **own** HPA. Never add a second one on the same workload |
| Argo Rollouts (P8-8) | Canary vs stable pods during a rollout | HPA sees CPU across both and doesn't know the canary only gets 10 % of traffic. Write down the handoff |
| Karpenter (P6-6) | Nodes, from pending pods; consolidation | Consolidation + missing PDBs = surprise evictions; a PDB that allows zero disruptions = drains that hang |
| Argo CD self-heal (P8-4) | Reverts anything not in Git | A field a controller legitimately changes needs `ignoreDifferences`, never disabled self-heal |

The fix is never "fewer autoscalers". It's **explicit scope: who owns the
number at each moment**, written in the ticket and checked once under load.

---

## 5. Count your stateful systems

Every stateful system needs a tested backup, an upgrade path, HA, capacity
planning and an owner. Count them:

| System | In the default template? | Who carries the Day-2 work |
|---|---|---|
| App databases | Yes | AWS (RDS/DynamoDB/ElastiCache, P4-9) |
| Terraform state | Yes | You (S3 versioning, P4-1) |
| Git (config is the backup) | Yes | GitHub |
| Metrics, logs, traces | Yes | AWS (AMP, CloudWatch, X-Ray, P10) |
| Prometheus/Loki/Tempo storage | Only on the self-hosted track (P10-13) | **You** |
| OpenBao storage | Only on the advanced track (P5-6) | **You** |
| CloudNativePG databases | Only on the operator track (P4-9) | **You** |
| Feature-flag store | Only with P8-12 | Depends on the backend |

**The count is the metric.** When choosing between two options, "does this
add a stateful system I run myself?" should weigh far more than a nicer
feature. It's the strongest argument for managed services, stronger than
cost: they take the system out of your backup, upgrade and on-call scope.

---

## 6. Can you recover when the platform itself is down?

Once everything runs on the platform, **the tools you'd use to fix it are on
the thing that's broken**: Argo CD deploys the fix, Grafana shows the problem,
ESO provides the secrets, and the runbook is in a wiki behind SSO.

Answer these in advance (tested in [P10-9](10-observe-and-operate.md#p10-9-backup-and-disaster-recovery)):

1. Cluster unreachable: how do you deploy? (A logged break-glass path, P8-4)
2. Secrets backend down: do running pods keep working? Do restarting ones?
3. Are the runbooks readable without the platform? (A copy in Git)
4. Does the restore need anything you'd be restoring?

Run at least one recovery drill **with the platform actually down**, not healthy.

---

## 7. Upgrades are coupled

EKS, the VPC CNI, add-ons, Karpenter, Argo CD, Kyverno, cert-manager, Velero,
Falco and every CRD each have their own release cadence and constrain each
other. "Upgrade the cluster" is really "upgrade a coupled set of parts in a
valid order". Keep the compatibility matrix and do one part at a time
([P10-10d](10-observe-and-operate.md#p10-10d-cluster-upgrade-runbook)).
Every optional data-path tool (Cilium, a mesh) adds a hard dependency to that
matrix.

---

## 8. Is this tool worth it? Six questions before adding anything

1. **Does it replace something done badly by hand, or add something new?** cert-manager replacing manual certificates is nearly pure gain. A second mesh next to your CNI is a close call.
2. **Is the team comfortable with the layer below it?** Cilium before a mesh; Audit before Enforce; non-prod chaos before prod chaos. The risk is in **skipping steps**, not in the order.
3. **How many places would a new engineer check to answer "why did this fail?"** If that number keeps growing without better correlation (trace IDs in logs, P10-0), debugging debt is growing faster than capability.
4. **Is a real requirement forcing it, or is it best practice for its own sake?** "We might need it later" isn't a reason. "This hurt us last quarter" is.
5. **Does it add a stateful system?** (§5) Weigh it heavily.
6. **Does it sit in the recovery path?** (§6) Then design the break-glass before you deploy it.

See also [Scale the template to your situation](README.md#scale-the-template-to-your-situation).
