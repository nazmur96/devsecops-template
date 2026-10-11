# Phase 6: Run it well

**Goal:** the app on EKS survives node drains, AZ loss, rolling updates, Spot
interruptions and traffic spikes without users noticing, and it scales up
before latency degrades, and back down to save money.

**Why:** a workload that runs isn't the same as one that runs **well**.
References 04 and 05 shipped 1 replica, no probes, no requests. Reference 06
is the only one that got this right; this phase is mostly its pattern plus
the drills from the user's plan and Section 12 list.

**Order matters** (each step depends on the one before):

```
requests + probes (P3) → replicas/HPA → PDB → topology spread → drain drill
                               ↓
                      Metrics Server (P4) → HPA → Karpenter
```

---

## P6-1 Re-verify the P3 workload baseline on EKS

**Type:** Task

- [ ] Every Deployment carries the P3-3 baseline (probes incl. `startupProbe` for JVMs, requests, memory limits, securityContext, own ServiceAccount, `preStop`, grace period)
- [ ] Requests now come from **measured** usage (P2-9), not guesses: CPU request ≈ p50 usage, memory request ≈ p95, memory limit ≈ 1.5× request
- [ ] Namespace PSA label `restricted` still enforced on EKS

---

## P6-2 Replicas and autoscaling

**Type:** Story

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: front-end, namespace: sock-shop }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: front-end }
  minReplicas: 3            # ≥ 3 so one per AZ and a PDB of 2 is safe
  maxReplicas: 12
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
  behavior:
    scaleDown: { stabilizationWindowSeconds: 300 }   # avoid flapping
```

- [ ] Metrics Server running (P4-6); `kubectl top pods` works
- [ ] HPA on every stateless service; `minReplicas: 3` for anything user-facing
- [ ] Remove `replicas:` from Deployments that have an HPA (otherwise every GitOps sync fights the HPA)
- [ ] **Prove it:** load test (P10) → pods scale up **before** p95 latency breaks the SLO; scale down after
- [ ] **[OPT]** Second HPA on a custom metric (requests/second via the Prometheus Adapter)
- [ ] **[OPT] Event-driven autoscaling with KEDA.** *Use when:* a service consumes a **queue or stream** (RabbitMQ, Kafka, SQS), so CPU says nothing about how much work is waiting. KEDA scales workers on **queue depth / consumer lag**, and can scale them to **zero** when the queue is empty. Sock Shop: scale `queue-master` on the length of the RabbitMQ shipping queue (KEDA uses Pod Identity for SQS, or a secret from ESO for RabbitMQ). Skip it for request/response services, where HPA on CPU or RPS is the right tool

---

## P6-3 PodDisruptionBudgets

**Type:** Story

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: front-end, namespace: sock-shop }
spec:
  minAvailable: 2
  selector: { matchLabels: { app.kubernetes.io/name: front-end } }
```

- [ ] A PDB per service with ≥ 2 replicas
- [ ] ⚠️ **Never** `minAvailable: 1` on a 1-replica Deployment: node drains (and so cluster upgrades and Karpenter consolidation) block forever. Reference 06's HPA min 3 + PDB min 2 is the safe pairing
- [ ] **Drill:** `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` under load → zero failed requests; the drain waits for replacements

---

## P6-4 Topology spread across nodes and zones

**Type:** Story

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule      # hard: never put all replicas in one AZ
    labelSelector: { matchLabels: { app.kubernetes.io/name: front-end } }
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway     # soft: prefer different nodes, but don't block
    labelSelector: { matchLabels: { app.kubernetes.io/name: front-end } }
```

- [ ] Every multi-replica service spread across zones; `kubectl get pods -o wide` + node zones confirm it
- [ ] **Experiment** (reference 06 does this side by side): set the zone rule to `ScheduleAnyway`, scale to 3 with one AZ's nodes cordoned → pods pile into 2 AZs. With `DoNotSchedule` → one stays `Pending`. Write down when you'd want each
- [ ] Nodes in every AZ is **not** HA on its own; the spread constraint is what makes it HA (reference 03)

---

## P6-5 Resource guardrails per namespace

**Type:** Story · *Reference 03*

- [ ] `ResourceQuota` per team namespace (pods, CPU, memory, PVCs, LoadBalancers = 0)
- [ ] `LimitRange` with default requests/limits so a manifest without them still gets sane values
- [ ] **Negative test:** a Deployment that would exceed the quota is rejected with a clear event

---

## P6-6 Node autoscaling with Karpenter (and Spot)

**Type:** Story · *Reference 06, section 17*

- [ ] Karpenter installed via Terraform/Helm with Pod Identity; its controller runs on the **managed node group** (never on nodes it manages)
- [ ] `NodePool` for on-demand (critical) and one for **Spot** (stateless, interruption-tolerant), with several instance types/families for capacity
- [ ] `EC2NodeClass` with IMDSv2 required, hop limit 1, encrypted volumes, private subnets
- [ ] **Interruption queue:** EventBridge → SQS → Karpenter, so a Spot 2-minute warning drains the node gracefully
- [ ] Consolidation on; PDBs (P6-3) make it safe
- [ ] **Drill:** scale a Deployment past capacity → `Pending` pods → new node within ~1 minute → scale down → node removed
- [ ] **Drill:** simulate a Spot interruption (AWS FIS, or send a test interruption event) → pods move with no failed requests
- [ ] Record the cost before/after (reference 06 claims up to ~70 % savings on Spot-eligible workloads)

---

## P6-7 Failure timing you must know

**Type:** Task · *User's Section 12 list*

- [ ] **Node failure:** the default `tolerationSeconds: 300` on `not-ready`/`unreachable` taints means pods on a dead node keep their place for **5 minutes**, while ALB health checks stop sending traffic in seconds. That's why multiple replicas + zone spread matter. Optionally lower it for stateless services
- [ ] **AZ failure drill:** cordon + drain all nodes in one AZ → CUJ still passes; record how long it takes for capacity to recover
- [ ] **Data stores:** EBS volumes are zonal. A StatefulSet pod can only restart in its volume's AZ. Write down what that means for in-cluster databases (and why managed/multi-AZ data stores exist)

---

## P6-8 [OPT] Right-sizing tools

> **Use when:** you have a few weeks of real usage data and cost matters. Too early, and the recommendations are based on noise.

- **VPA in recommendation mode:** suggests requests from real usage (don't run it in `Auto` alongside an HPA on the same metric)
- **Goldilocks** or **Kubecost/OpenCost:** request vs usage, cost per namespace

---

## Prove it

Run all of these **under the CUJ load loop** (P3-8) and record zero failures:

- [ ] Rolling update of every service
- [ ] Drain of one node; cordon + drain of a whole AZ
- [ ] A Karpenter consolidation and a simulated Spot interruption
- [ ] HPA scale-up during the load test (P10) before the latency SLO is breached

## Common mistakes

| Mistake | Seen in |
|---|---|
| `replicas: 1`, no probes, no requests | 04, 05 |
| PDB on a single replica (blocks drains/upgrades) | user's Section 12 list warns about it |
| Nodes in 3 AZs but no topology spread | 02 (2-AZ nodegroup only) |
| `replicas:` and an HPA on the same Deployment (GitOps flapping) | — |
| min/max on a node group with no autoscaler (does nothing) | 04 (the instructor notes it himself) |
| Karpenter controller running on Karpenter-managed nodes | — |
