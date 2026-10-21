# Phase 3: Local Kubernetes bridge

**Goal:** the whole application runs on a local multi-node cluster from
**hand-written manifests**, survives node failures and rolling updates with
zero dropped requests, and you can iterate on code against it in seconds.

**Why:** learning Kubernetes on EKS mixes three hard things at once
(Kubernetes, AWS networking/IAM, and a meter that's running). A local cluster
removes two of them. Probes, resources, shutdown, StatefulSets and network
policy are all proven here, **for free**, then re-verified on EKS (reference
07, ADR-0003). E-3 is expected to be the heaviest epic, and that's the point.

**Rule:** no Helm, no generators, no upstream YAML in this phase. You can only
judge a Helm chart later if you've written the YAML it should produce.

---

## P3-1 Create the local cluster

**Type:** Story

```yaml
# kind/cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: sockshop
nodes:
  - role: control-plane
    image: kindest/node:v1.36.4@sha256:<digest>   # SAME minor as kubectl and EKS (P1-2 ADR); digest from the kind release notes
    extraPortMappings:
      - { containerPort: 80,  hostPort: 80 }
      - { containerPort: 443, hostPort: 443 }
  - role: control-plane
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

- [ ] `kind create cluster --config kind/cluster.yaml` is the single command (wrapped in `make cluster-up`)
- [ ] Node image matches the EKS minor version you'll use in P4
- [ ] Namespaces created **with Pod Security Admission labels** (learn PSA here, cheaply):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sock-shop
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```

- [ ] **Drill:** `docker stop sockshop-worker` → pods reschedule; a sample app stays reachable
- [ ] **Drill:** stop one control-plane container → API still answers (etcd quorum 2/3)

---

## P3-2 Manifest structure and standard labels

**Type:** Task

```
k8s/
├── base/                    # one folder per service + databases + broker
│   ├── namespace.yaml
│   ├── front-end/{deployment,service,serviceaccount}.yaml
│   └── …
├── local/                   # Gateway + HTTPRoutes + issuer for KinD
└── overlays/{dev,prod}/     # kustomize patches (images, replicas, resources)
```

Every resource carries:

```yaml
labels:
  app.kubernetes.io/name: carts
  app.kubernetes.io/instance: sockshop-dev
  app.kubernetes.io/component: backend      # frontend | backend | database | broker
  app.kubernetes.io/part-of: sockshop
  app.kubernetes.io/managed-by: argocd      # becomes true in P8
```

---

## P3-3 Write the workload manifests (Java first)

**Type:** Story · *Java first on purpose: its slow start teaches the
`startupProbe` lesson immediately (reference 07)*

The **workload baseline**, the best of references 06 and 07 combined. Copy it for every service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: carts
  labels: { app.kubernetes.io/name: carts, app.kubernetes.io/component: backend }
spec:
  replicas: 2
  revisionHistoryLimit: 5
  selector: { matchLabels: { app.kubernetes.io/name: carts } }
  template:
    metadata:
      labels: { app.kubernetes.io/name: carts, app.kubernetes.io/component: backend }
    spec:
      serviceAccountName: carts
      automountServiceAccountToken: false
      # Kubernetes injects CARTS_PORT=tcp://…, CARTS_SERVICE_HOST, … for every
      # Service in the namespace. An app whose own env prefix matches a
      # Service name (podinfo reads PODINFO_*) takes them as config.
      enableServiceLinks: false
      terminationGracePeriodSeconds: 45
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile: { type: RuntimeDefault }
      containers:
        - name: carts
          image: sockshop/carts:dev        # replaced by a digest from P7 onwards
          ports: [{ name: http, containerPort: 8080 }]
          env:
            - name: JAVA_TOOL_OPTIONS
              value: "-XX:MaxRAMPercentage=75.0"
          resources:
            requests: { cpu: 250m, memory: 512Mi }
            limits:   { memory: 1Gi }      # memory limit yes; CPU limit optional (throttling)
          startupProbe:                    # gives the JVM up to 30 × 10 s = 5 min to start
            httpGet: { path: /health, port: http }
            periodSeconds: 10
            failureThreshold: 30
          readinessProbe:                  # "send me traffic?" May check dependencies.
            httpGet: { path: /health, port: http }
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          livenessProbe:                   # "restart me?" Must NOT check the database.
            httpGet: { path: /health/liveness, port: http }
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              exec: { command: ["sleep", "10"] }   # let endpoints deregister before SIGTERM
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          volumeMounts:
            - { name: tmp, mountPath: /tmp }        # writable scratch for a read-only root FS
      volumes:
        - { name: tmp, emptyDir: {} }
```

> **The probe trap** (user's Section 12 list): if liveness checks the
> database, a database blip restarts **every** pod at once, turning a
> 10-second outage into a 5-minute one. Liveness answers "is this process
> stuck?", readiness answers "can I serve right now?".
>
> **`sleep` needs a shell.** Distroless images have none; for those, use the
> native `preStop: { sleep: { seconds: 10 } }` hook (Kubernetes ≥ 1.30) instead.

- [ ] Every service: Deployment + ClusterIP Service + its own ServiceAccount
- [ ] **Drill:** delete the `startupProbe` from a Java service → it enters `CrashLoopBackOff`. Restore it; explain why in a manifest comment
- [ ] All pods `Running` and `Ready` under the `restricted` PSA profile
- [ ] DNS check from another pod: `nslookup catalogue.sock-shop.svc.cluster.local`

---

## P3-4 Stateful data stores

**Type:** Story

- [ ] One `StatefulSet` per data store with `volumeClaimTemplates` (**never** a shared PVC) and a **headless** Service (`clusterIP: None`)
- [ ] Stable DNS verified: `carts-db-0.carts-db.sock-shop.svc.cluster.local`
- [ ] Credentials from a `Secret` (generated locally, git-ignored), never a literal in YAML
- [ ] **Drill:** write data → `kubectl delete pod carts-db-0` → data survives
- [ ] Comment in the manifest: what breaks with a normal Service instead of a headless one

> Databases inside the cluster are fine for **learning** and for dev. For
> production, the template uses managed data stores in P4 (reference 06) and
> compares them with an operator (CloudNativePG) as an optional track.

---

## P3-5 Message broker and the async path

**Type:** Story

- [ ] RabbitMQ with credentials from a Secret
- [ ] `shipping` publishes, `queue-master` consumes (visible in the management UI via port-forward)
- [ ] **Drill:** delete the consumer pod mid-flow → messages queue up and drain on recovery, none lost

---

## P3-6 Gateway API and local TLS

**Type:** Story

> **Read first:** [Kubernetes networking, explained simply](https://github.com/nazmur96/devops-learning-journal/blob/main/networking/kubernetes-networking-explained.md):
> the five networking jobs (CNI, front door, routing rules, the controller
> that follows them, optional mesh), L4 vs L7, Ingress vs Gateway API.
>
> **Routing API: Gateway API** (template default; ADR in P9). The community
> `ingress-nginx` controller was retired in March 2026, and Gateway API is the
> standard going forward. Locally the rule follower is **Envoy Gateway**; on
> EKS it's the AWS Load Balancer Controller (P9). **Your `HTTPRoute`s stay the
> same in both places**: only the `GatewayClass`/`Gateway` underneath changes.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata: { name: envoy }
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public
  namespace: gateway
  annotations:
    cert-manager.io/cluster-issuer: local-ca          # cert-manager issues the listener's certificate
spec:
  gatewayClassName: envoy
  listeners:
    - { name: http,  protocol: HTTP,  port: 80, allowedRoutes: { namespaces: { from: Same } } }
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.sockshop.local"
      tls:
        mode: Terminate
        certificateRefs: [{ name: sockshop-local-tls }]
      allowedRoutes:
        namespaces:
          from: Selector
          selector: { matchLabels: { gateway-access: public } }
```

- [ ] Gateway API standard CRDs + **Envoy Gateway** installed; its data-plane Service reachable on KinD's host ports 80/443 (NodePort + `extraPortMappings`, or MetalLB/`cloud-provider-kind`, P3-11)
- [ ] `HTTPRoute`s for the path map from `docs/inventory.md` (the same ones you'll use on EKS in P9-3); `sockshop.local` in `/etc/hosts`. Each route lives **with its app's manifests** (`k8s/overlays/…`), not in the platform install: the Gateway is platform, the route is the app's. A platform script that applies app routes breaks the from-scratch order (the app namespace doesn't exist yet)
- [ ] The gateway's own namespaces (Gateway, controller, cert-manager) get PSA labels too; check with `kubectl label --dry-run=server ns <ns> pod-security.kubernetes.io/enforce=restricted` (no warnings = nothing needs an exception)
- [ ] cert-manager with its Gateway API support enabled and a **self-signed CA** `ClusterIssuer` (`local-ca`); wildcard cert `*.sockshop.local` issued for the listener; CA trusted locally
- [ ] HTTP → HTTPS with the standard `RequestRedirect` filter; undefined paths return 404
- [ ] **Drill:** shorten the cert duration → watch cert-manager reissue it **on schedule**. With a Gateway-shim certificate, the `cert-manager.io/duration` / `renew-before` annotations are read only when cert-manager **creates** the Certificate, and deleting just the Secret reissues once. So: annotate the Gateway, delete the Certificate and its Secret, then watch `status.revision` climb on its own (`1h`/`59m` renews every minute). Restore after
- [ ] **Negative test:** a namespace without the `gateway-access: public` label can't attach a route

TLS comes with the first gateway, not "later in a security phase" (reference 07).

---

## P3-7 NetworkPolicy, locally first

**Type:** Story

```yaml
# default deny for the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: sock-shop }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# everyone may resolve DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-dns, namespace: sock-shop }
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
          podSelector: { matchLabels: { k8s-app: kube-dns } }
      ports: [{ protocol: UDP, port: 53 }, { protocol: TCP, port: 53 }]
```

Then one allow policy per real call from the dependency diagram (P0-2).

- [ ] **First prove enforcement exists:** before any policy, a `curl` to the app must **succeed** (control); with only `default-deny` applied, the same `curl` must **fail**. If it still succeeds, your CNI ignores NetworkPolicy. Recent KinD (kindnet) enforces it; otherwise install Calico, or Cilium ([P3-7b](#p3-7b-opt-cilium-instead-of-kindnet-hubble-l7-policy-wireguard)). **Send the `curl` from a pod whose own egress is open** (a namespace with no policies, or a temporary allow-all-egress for the probe): from inside the default-deny namespace it fails on the *client's* egress, so it proves nothing about the app's ingress
- [ ] Apply deny-all → watch DNS break → allow DNS → add allows call by call until the CUJ passes
- [ ] The front-door allow selects the **proxy pods** (namespace *and* pod selector in **one** `from` entry; as two entries it means "any pod in that namespace, or these pods here"). With Envoy Gateway the proxies run in `envoy-gateway-system`, not the `Gateway`'s namespace. On EKS the ALB replaces it, so keep this allow in the local overlay
- [ ] **Test every allow both ways**, by script: what it must let through (the CUJ), and what it must not. For the front-door allow, that includes a **non-proxy pod in the proxies' namespace**: it's the only probe that catches the split-selector mistake above. Break each allow on purpose (delete it, typo a label, split the selectors) and watch the script fail
- [ ] **Negative test:** `front-end` cannot reach `carts-db` directly
- [ ] An allow trusts **labels**: anyone who can create pods in the allowed namespace can give a pod that label (`app.kubernetes.io/name: prometheus`) and get in. Fine locally; on EKS limit who can create pods there (P5 RBAC, P8 admission)

A starting point for the script; it already avoids both false proofs above.
Add one `reach`/`blocked` line per allow and per must-not path:

```bash
#!/usr/bin/env bash
# scripts/check-netpol.sh: prove the policies both ways. Exit 0 only if all pass.
# Env: NS (app namespace), TARGET (URL inside the cluster), PROXY_NS (front door's pods)
set -uo pipefail
NS=${NS:-sock-shop}; PROXY_NS=${PROXY_NS:-envoy-gateway-system}
TARGET=${TARGET:-http://front-end.$NS.svc.cluster.local/}
IMG=busybox:1.37.0   # pin a digest; must pass PSA restricted
failed=0; ok() { echo "PASS $1"; }; bad() { echo "FAIL $1"; failed=1; }

probe() { # <ns> <name>: restricted-compliant, dies at once (busybox sleep ignores SIGTERM)
  kubectl apply -f - >/dev/null <<EOF
apiVersion: v1
kind: Pod
metadata: { name: $2, namespace: $1, labels: { netpol-probe: "true" } }
spec:
  terminationGracePeriodSeconds: 0
  automountServiceAccountToken: false
  securityContext: { runAsNonRoot: true, runAsUser: 65532, seccompProfile: { type: RuntimeDefault } }
  containers:
    - name: p
      image: $IMG
      command: [sleep, "600"]
      securityContext: { allowPrivilegeEscalation: false, readOnlyRootFilesystem: true, capabilities: { drop: [ALL] } }
EOF
  kubectl -n "$1" wait --for=condition=Ready "pod/$2" --timeout=60s >/dev/null
}
reach() { kubectl -n "$1" exec "$2" -- wget -q -T 5 -O /dev/null "$TARGET" 2>/dev/null; }
trap 'kubectl delete ns netpol-probe --wait=false >/dev/null 2>&1
      kubectl -n "$NS" delete pod,networkpolicy -l netpol-probe=true --wait=false >/dev/null 2>&1
      kubectl -n "$PROXY_NS" delete pod netpol-probe --wait=false >/dev/null 2>&1' EXIT

# The last run's cleanup doesn't wait; reusing objects still being deleted fails oddly.
kubectl delete ns netpol-probe --ignore-not-found --wait=true >/dev/null
kubectl -n "$NS" delete pod netpol-probe --ignore-not-found --wait=true >/dev/null
kubectl -n "$PROXY_NS" delete pod netpol-probe --ignore-not-found --wait=true >/dev/null
kubectl create ns netpol-probe >/dev/null && kubectl label ns netpol-probe pod-security.kubernetes.io/enforce=restricted >/dev/null
probe netpol-probe outside; probe "$NS" netpol-probe; probe "$PROXY_NS" netpol-probe

kubectl -n "$NS" exec netpol-probe -- nslookup kubernetes.default.svc.cluster.local >/dev/null 2>&1 && ok "DNS from $NS" || bad "DNS from $NS"
reach netpol-probe outside && bad "another namespace reached the app" || ok "another namespace blocked"
# Same namespace: open the probe's OWN egress first, or it fails for the wrong reason.
kubectl apply -f - >/dev/null <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: probe-egress, namespace: $NS, labels: { netpol-probe: "true" } }
spec: { podSelector: { matchLabels: { netpol-probe: "true" } }, policyTypes: [Egress], egress: [{}] }
EOF
sleep 2
kubectl -n "$NS" exec netpol-probe -- nc -z -w 5 kubernetes.default.svc.cluster.local 443 2>/dev/null \
  || bad "probe egress didn't open: check 'same namespace' would prove nothing"
reach "$NS" netpol-probe && bad "a pod in $NS reached the app" || ok "same namespace blocked"
# Catches namespaceSelector and podSelector written as two `from` entries.
reach "$PROXY_NS" netpol-probe && bad "non-proxy pod in $PROXY_NS got in (allow too broad)" || ok "non-proxy pod in $PROXY_NS blocked"
scripts/cuj.sh >/dev/null && ok "CUJ through the front door" || bad "CUJ through the front door"
exit "$failed"
```

Run it once **before** any policy (the "blocked" checks must fail: that's the
control), then after each policy you add.

> kubelet probes come from the node, not a pod: on kindnet, default-deny
> doesn't block them (pods stay Ready). Test it on your CNI rather than
> assuming.

> Reference 01 was the only project with NetworkPolicies. Doing it here, with
> fast feedback, makes it a 1-day task instead of a week of cloud debugging.

---

## P3-7b [OPT] Cilium instead of kindnet (Hubble, L7 policy, WireGuard)

> **Use when:** you want to learn Cilium for free before a homelab (Talos) or
> a Cilium-on-EKS project, or you're stuck on "why is this connection
> blocked?". **Skip it** on a pure EKS path: EKS uses the VPC CNI, and
> **Hubble only exists when Cilium is the CNI**. Background:
> [Kubernetes traffic stack: a Cilium-first setup](https://claude.ai/artifact/BDopFWFZhkTaizFShXurbF).

**What you get that kindnet doesn't have:**

| Feature | What it does | Why it matters |
|---|---|---|
| **Hubble** | Shows every network flow: who called whom, and whether policy **allowed or dropped** it | Debugging NetworkPolicy goes from guessing to reading |
| **L7 `CiliumNetworkPolicy`** | Rules on HTTP method and path, not just IP and port | "`front-end` may only `GET /catalogue`", not "may reach port 80" |
| **WireGuard encryption** | Encrypts pod traffic between nodes | Matters on bare metal; on AWS, supported Nitro instance types already encrypt traffic between instances |
| **kube-proxy replacement** | Service load balancing in eBPF instead of iptables | Faster at scale, and one less component |

**Steps:**

- [ ] Add `cilium-cli` and `hubble` to `mise.toml` (P1-2); they're client tools, not host packages
- [ ] A second cluster config (`kind/cluster-cilium.yaml`) that turns off the defaults Cilium replaces:

```yaml
networking:
  disableDefaultCNI: true   # no kindnet
  kubeProxyMode: none       # Cilium replaces kube-proxy
```

- [ ] Install, with Cilium's own Gateway API **off** so it doesn't fight Envoy Gateway (P3-6) over the same `Gateway`s:

```bash
cilium install \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=sockshop-external-load-balancer --set k8sServicePort=6443 \
  --set hubble.relay.enabled=true --set hubble.ui.enabled=true \
  --set encryption.enabled=true --set encryption.type=wireguard
cilium status --wait          # all green; "Encryption: Wireguard"
```

  (`k8sServiceHost` is KinD's API load-balancer container when you run 3 control planes; with one control plane it's `sockshop-control-plane`.)
- [ ] `cilium connectivity test` passes
- [ ] Re-run P3-7 on this cluster: the standard NetworkPolicies work unchanged
- [ ] **See the denies:** `cilium hubble port-forward &` then `hubble observe -n sock-shop --verdict DROPPED` while you test `front-end` → `carts-db`. Each dropped flow names its source, destination and port. `cilium hubble ui` shows the same as a live service map
- [ ] **One L7 rule**, proven with a negative test:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: front-end-reads-catalogue, namespace: sock-shop }
spec:
  endpointSelector: { matchLabels: { app.kubernetes.io/name: catalogue } }
  ingress:
    - fromEndpoints: [{ matchLabels: { app.kubernetes.io/name: front-end } }]
      toPorts:
        - ports: [{ port: "80", protocol: TCP }]
          rules: { http: [{ method: GET, path: "/catalogue.*" }] }
```

  `GET /catalogue` from `front-end` → 200; `POST /catalogue` → **403 from Cilium**, visible in `hubble observe --protocol http`
- [ ] **Prove encryption:** `cilium encrypt status` shows WireGuard peers. Optionally `tcpdump` on a node's interface shows UDP 51871 (WireGuard), not plain HTTP

> **Common mistake:** two Gateway API controllers (Cilium's and Envoy
> Gateway) both claiming the same `Gateway`. Keep one, or give each its own
> `GatewayClass`.

---

## P3-8 Full local verification and graceful-shutdown tuning

**Type:** Task · *Exit gate for this phase*

- [ ] CUJ passes over HTTPS
- [ ] Data survives deleting **every** application pod, then every data-store pod
- [ ] **Rolling update with zero dropped requests** under a continuous load loop while `kubectl rollout restart deploy/catalogue` runs. Tune `preStop` / grace period until only `200`s appear. A starting point that fails both on a dropped request and on an action that didn't happen:
  ```bash
  #!/usr/bin/env bash
  # scripts/rollout-drill.sh restart <deploy> | drain <node>
  # Env: NS, URL (through the front door, e.g. https://sockshop.local/catalogue)
  set -uo pipefail
  NS=${NS:-sock-shop}; URL=${URL:-https://sockshop.local/catalogue}
  out=$(mktemp); stop=$out.stop; trap 'rm -f "$out" "$stop"' EXIT
  ( while [ ! -f "$stop" ]; do curl -s -o /dev/null -w '%{http_code}\n' -m 5 "$URL" >>"$out"; done ) &
  sleep 3; action_ok=1   # a baseline before anything moves
  case $1 in
    restart) kubectl -n "$NS" rollout restart "deploy/$2" && kubectl -n "$NS" rollout status "deploy/$2" --timeout=180s || action_ok=0 ;;
    drain)   kubectl drain "$2" --ignore-daemonsets --delete-emptydir-data --timeout=180s || action_ok=0 ;;
  esac
  sleep 10; touch "$stop"; wait   # old pods finish terminating while the loop runs
  [ "$1" = drain ] && kubectl uncordon "$2"
  sort "$out" | uniq -c
  [ "$action_ok" = 1 ] || { echo "FAIL the $1 itself failed: the tally proves nothing"; exit 1; }
  [ "$(grep -vc '^200$' "$out")" = 0 ]
  ```
- [ ] **Prove the loop can see a drop** before trusting a clean tally: one control run with `preStop` removed (and any app-side drain skipped) must show non-`200`s. The script must also **fail when the rollout or drain itself fails**: a drain refused by a stray bare pod moves nothing, and its all-`200` tally is a false pass
- [ ] The same loop while `kubectl drain` takes out each worker in turn. **The front door counts:** Envoy Gateway runs **one** proxy replica by default, so draining its node drops every request while the app is fine. Give the proxy ≥ 2 replicas spread across nodes and a PDB (`EnvoyProxy` `envoyDeployment.replicas`, `envoyPDB`)
- [ ] Undo drills that used `kubectl set env` with `kubectl set env … VAR-`. A re-apply doesn't remove them: `kubectl apply` merges three ways, and a field that was never in the last-applied config survives (Argo CD self-heal fixes this class in P8)
- [ ] Document which services don't handle SIGTERM and what you did about it
- [ ] `make cluster-down && make cluster-up && make deploy` rebuilds everything from scratch

---

## P3-9 Set up the inner loop

**Type:** Story · *Now that workloads exist (workstation notes, plane 3)*

- [ ] **Pick one** of Skaffold or Tilt (ADR). Code change → running in the cluster in < 30 s, **measured** from save to served through the Gateway
- [ ] Tilt prunes the **host's** Docker build cache by default (other projects' too): `docker_prune_settings(disable=True)` in the `Tiltfile`
- [ ] **mirrord:** run one service from your editor (with a debugger) while it receives the cluster's traffic, env vars and DNS:
  ```bash
  mirrord exec -f .mirrord/mirrord.yaml -- ./bin/catalogue   # a built binary, not `go run`
  ```
  - **The agent is privileged** (host PID, privilege escalation). If the `default` namespace has no PSA labels it lands there **silently**: label `default` `restricted`, and give the agent its own namespace (`agent.namespace`) labelled `privileged`, created only while you use it. That namespace is a PSA exception: write the ADR
  - **Go:** mirrord loads through `LD_PRELOAD` and skips build tools such as `go`, so `go run .` and any static (`CGO_ENABLED=0`) binary run unhooked: no env, no files, no traffic, and no error. Build a dynamically linked binary (cgo; with no host C compiler, `zig cc` from mise works) and run that
  - **Files:** check that the local process really reads the pod's files (its mounted Secret): one test is a token issued by a cluster pod validating on the local one. `fs.mode: read` worked for podinfo; `local` + `read_only` did not
  - **Prove it:** a share of the Gateway's responses comes from the local build (e.g. `/version` shows its commit), and all of it returns to the cluster after Ctrl-C
- [ ] Documented in `CONTRIBUTING.md`: "to work on service X, run …"

---

## P3-10 Demo-grade observability stack (free, local)

**Type:** Story · *Learn what's inside before using the AWS-managed versions in P10 (reference 04, parts 40–41, about an hour of content)*

**Why here:** on EKS the template uses AWS-managed observability (P10), where
AWS hides the storage and plumbing. Running the open-source stack on KinD
first shows you what those services do, costs nothing, and teaches the query
languages (PromQL, LogQL, TraceQL) you'll use everywhere. The **app
instrumentation you add here is the same OpenTelemetry instrumentation P10
uses**; only the collector's export target changes.

| Signal | Local tool (this ticket) | On EKS (P10) |
|---|---|---|
| Metrics | Prometheus (`kube-prometheus-stack`) | Amazon Managed Prometheus |
| Logs | Loki (single binary, filesystem) + Grafana Alloy | CloudWatch Logs |
| Traces | Tempo (single binary) | X-Ray / Application Signals |
| Dashboards | Grafana | Amazon Managed Grafana |
| Collector | OTel Collector (via the OTel Operator) | ADOT (AWS's OTel Collector) |

- [ ] Installed with Helm into an `observability` namespace, from a committed `values-local.yaml` per chart, every version pinned: `kube-prometheus-stack`, `grafana/loki` (single-binary mode), `grafana/alloy`, **`grafana-community/tempo`** (`grafana/tempo` is deprecated and moved), the OpenTelemetry Operator (+ cert-manager from P3-6) **only if** a service needs auto-instrumentation; for SDK-instrumented services Alloy is the collector
- [ ] **PSA `restricted` on `observability` too**, checked with the server dry-run from P3-6. node-exporter needs the host's PID and network namespaces: turn it off locally, or give it its own namespace and an ADR. Alloy reads pod logs **through the API** (`loki.source.kubernetes`), not a hostPath mount of `/var/log`. With a read-only root filesystem, give Tempo a volume at `/var/tempo` when persistence is off. After fixing a crash-looping **StatefulSet** with `helm upgrade`, delete its pod: a StatefulSet doesn't replace a pod that never became Ready, so the old broken pod keeps running and `--wait` times out
- [ ] Never the chart's default Grafana password (`kube-prometheus-stack` ships `prom-operator`): generate a Secret, `grafana.admin.existingSecret`
- [ ] **Demo-grade on purpose:** one replica, local storage, short retention (24 h). Write that down; production-grade self-hosting is P10-13
- [ ] Laptop RAM: the stack wants ~1.5–4 GiB (podinfo run: 1.4 GiB working set without node-exporter). If the 6-node KinD cluster struggles, use a 1 control-plane + 2 worker variant while this stack runs
- [ ] OTel **auto-instrumentation** for Java and Node (`Instrumentation` resource + pod annotation); Go via the SDK
- [ ] `ServiceMonitor` (or `PodMonitor` on a separate metrics port) per service, plus a NetworkPolicy hole for exactly the scrape and the OTLP export; recheck that the metrics port serves nothing else (P2-1a side doors)
- [ ] Grafana data sources wired with **trace ↔ logs ↔ metrics** links (Tempo "trace to logs", Loki derived field on `trace_id`). **First find where the trace ID really is**: podinfo puts it only in its debug-level request log on stderr, but in every OTLP log record. Tempo's search API drops leading zeros from trace IDs, so match logs by substring, not equality
- [ ] The SLI counts **user** requests: leave probe paths out, or the kubelet's own calls pad the "good" count
- [ ] Learn the three query languages by answering:
  - [ ] PromQL: p95 latency of `orders` over 5 minutes
  - [ ] LogQL: all `ERROR` lines from `carts` in the last hour
  - [ ] TraceQL: traces through `orders` slower than 500 ms
- [ ] First **SLO + burn-rate alert** written here as Prometheus rules (or generated with Sloth), routed by Alertmanager to a local webhook receiver. The same rules are uploaded to AMP in P10-4
- [ ] **Drill:** add 500 ms latency to one service → find it from a trace, jump to its logs, see it on the RED dashboard

> This is roughly what reference 04 builds in its monitoring parts (Prometheus,
> Loki, Alloy, Grafana, kube-state-metrics, Tempo) and what reference 08 runs
> on EKS (without traces). Here it's a learning tool, not production.

---

## P3-11 [OPT] Bare-metal equivalents

> **Use when:** you'll also run Kubernetes on-prem or on your homelab (Talos on Proxmox), or want to see exactly what EKS automates for you. Skip for an EKS-only path.

**Type:** Story · *Skills managed EKS hides (reference 07)*

| Ticket | What | Replaces in the cloud |
|---|---|---|
| MetalLB (L2) or `cloud-provider-kind` | `type: LoadBalancer` with no cloud | AWS NLB/ALB |
| NFS / Longhorn StorageClass + node-loss test | Volumes that survive node loss | EBS CSI |
| Cilium + Hubble, L7 `CiliumNetworkPolicy` (learn it in [P3-7b](#p3-7b-opt-cilium-instead-of-kindnet-hubble-l7-policy-wireguard)) | eBPF networking, HTTP-aware policy, flow visibility | VPC CNI |

---

## Prove it

- [ ] All P3-8 checks pass from a clean cluster
- [ ] PSA `restricted` enforced; no workload needed an exception (or each exception has an ADR)
- [ ] Default-deny NetworkPolicy proven enforced; CUJ passes with policies on

## Common mistakes

| Mistake | Seen in |
|---|---|
| Going straight to EKS and learning Kubernetes on a running bill | 01, 02, 04, 05, 06 |
| No probes at all | 04, 05 |
| No `startupProbe` on the JVM → CrashLoopBackOff "mystery" | (07 turns it into a drill) |
| Liveness probe checking the database | user's Section 12 list |
| NetworkPolicy written but never proven enforced | (common; the CNI may ignore it) |
| `replicas: 1` everywhere | 04, 05 |
