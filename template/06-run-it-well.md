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

**Before you start:** P4 (cluster, ECR, add-ons) and P5 (Pod Identity,
External Secrets) are done. P7 and P8 are **not** needed yet: this phase
deploys by hand (P6-0) so you can learn how the workload behaves on EKS
before you automate how it gets there.

---

## P6-0 First deploy to EKS (temporary, by hand)

**Type:** Task · *Replaced by GitOps in P8; until then this is the only way the app reaches EKS*

Nothing has deployed the app to EKS yet: CI arrives in P7, Argo CD in P8.
To run the drills in this phase you need the app running now, so deploy it
by hand **once**, knowing it's temporary.

- [ ] Build each image locally (P2) and push it to ECR (P4-7) with your **SSO role** (P5-1): `aws ecr get-login-password | docker login …`, then `docker push`. Write down each **digest** it prints
- [ ] Copy `k8s/` from P3 into `k8s/overlays/eks-dev/` (a Kustomize overlay) that changes only what differs on EKS: image → `<account>.dkr.ecr.<region>.amazonaws.com/sockshop/<svc>@sha256:<digest>`, StorageClass → `gp3`, Kubernetes Secrets → the `ExternalSecret`s from P5-5
- [ ] `kubectl apply -k k8s/overlays/eks-dev` → every pod `Running` and `Ready`
- [ ] Reach the app **without a load balancer** (the ALB comes in P9): `kubectl -n sock-shop port-forward svc/front-end 8080:80`, then run the CUJ script (P2-8) and the CUJ load loop (P3-8) against `http://localhost:8080`
- [ ] The overlay sets `app.kubernetes.io/managed-by: kubectl` (Kustomize `labels:`), so anyone looking at the cluster can see these resources weren't deployed by Argo CD. P8-2 removes them with `kubectl delete -k` on this same overlay
- [ ] **Never repeat this for staging or prod.** It exists for this one learning environment, and only until P8

> **Why not wait for P8?** You'd be learning probes, HPA, PDBs, Karpenter,
> Argo CD, Helm and Kyverno all at once. Doing P6 by hand first means that when
> something breaks in P8, you know the workload itself was already fine.

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
- [ ] **Prove it:** run the CUJ load loop (P3-8) with more parallel clients (or `hey -z 5m -c 50 http://localhost:8080/`) → pods scale up; stop it → they scale down after the stabilisation window. The full version (p95 against the SLO, with k6) comes back in P10-7
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

- [ ] **Node failure:** the default `tolerationSeconds: 300` on `not-ready`/`unreachable` taints means pods on a dead node keep their place for **5 minutes**, while a load balancer's health checks stop sending traffic in seconds (you'll see this with the ALB in P9). That's why multiple replicas + zone spread matter. Optionally lower it for stateless services
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
- [ ] HPA scale-up under the load loop (re-checked against the latency SLO with k6 in P10-7)

## Common mistakes

| Mistake | Seen in |
|---|---|
| `replicas: 1`, no probes, no requests | 04, 05 |
| PDB on a single replica (blocks drains/upgrades) | user's Section 12 list warns about it |
| Nodes in 3 AZs but no topology spread | 02 (2-AZ nodegroup only) |
| `replicas:` and an HPA on the same Deployment (GitOps flapping) | — |
| min/max on a node group with no autoscaler (does nothing) | 04 (the instructor notes it himself) |
| Karpenter controller running on Karpenter-managed nodes | — |
