# Phase 8: GitOps & enforcement

**Goal:** Git is the only way anything reaches the cluster. Manual changes are
reverted automatically, and the cluster itself **rejects** anything that
skipped the pipeline: unsigned images, root containers, `latest` tags,
missing limits, unknown registries.

**Why:** Layer 2 (CI) checks what goes **through** the pipeline. Layer 3 makes
sure nothing goes **around** it: a `kubectl apply` from a laptop, a stolen
registry token, a teammate's "quick fix". **No reference project implemented
admission control**; all of them listed Kyverno as "future work".

---

## P8-1 Package with Helm (after you've written the YAML)

**Type:** Story · *Reference 07's order: raw manifests in P3, templates now*

- [ ] Umbrella chart `sockshop` with a subchart (or a shared library chart) per service
- [ ] Parameterised: `image.repository` + `image.digest`, replicas/HPA, resources, `HTTPRoute` hostnames/paths, secret references
- [ ] `values.yaml` (defaults) + `envs/{dev,staging,prod}/values.yaml` in the **gitops repo**
- [ ] `helm template` output diffed against your P3 manifests; every difference understood
- [ ] `helm lint` and `checkov --framework helm` pass in CI (P7-2)
- [ ] **Publish the chart as an OCI artifact to ECR** (no separate chart server needed): bump `version` in `Chart.yaml` (semver) → CI runs `helm package charts/sockshop` → `helm push sockshop-1.4.0.tgz oci://<account>.dkr.ecr.<region>.amazonaws.com/charts` (an ECR repo `charts/sockshop`, immutable tags). Argo CD then installs a **pinned chart version** from OCI instead of a Git path. Optionally sign the chart with Cosign like an image
- [ ] Rule: **chart version** changes when the chart's templates change; **image digest** changes on every release. They're separate on purpose
- [ ] Know this detail (reference 09): **Argo CD doesn't run `helm install`.** It renders the chart (like `helm template`) and applies the result itself, so there's **no Helm release** in the cluster and `helm rollback` doesn't apply. Rollback is `git revert` in the gitops repo

---

## P8-2 Install Argo CD as code

**Type:** Story

- [ ] Argo CD installed by Terraform (`helm_release`) or a bootstrap script **checked into Git**, not a one-off `kubectl apply` (reference 06 used a script; 04 never used Argo CD)
- [ ] **Hardening:**
  - [ ] initial admin password rotated and the `argocd-initial-admin-secret` deleted; local admin disabled once SSO works
  - [ ] SSO (GitHub/OIDC) with Argo CD RBAC: `role:readonly` by default, platform admins get sync rights
  - [ ] UI/API **not** exposed publicly (port-forward, or an internal ALB / VPN)
  - [ ] repo credentials stored as a Kubernetes Secret (via External Secrets), never in a manifest
- [ ] Argo CD then **manages itself** (app-of-apps includes Argo CD's own chart)

---

## P8-3 App-of-apps and projects

**Type:** Story

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata: { name: sockshop, namespace: argocd }
spec:
  sourceRepos: ["https://github.com/<org>/sockshop-gitops"]
  destinations:
    - { server: https://kubernetes.default.svc, namespace: sock-shop }
  clusterResourceWhitelist: []           # app teams can't create cluster-scoped objects
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: sockshop-dev, namespace: argocd }
spec:
  project: sockshop                       # not "default" (reference 06 used default)
  source:
    repoURL: https://github.com/<org>/sockshop-gitops
    targetRevision: main
    path: charts/sockshop
    helm: { valueFiles: [../../envs/dev/values.yaml] }
  destination: { server: https://kubernetes.default.svc, namespace: sock-shop }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=false]
```

- [ ] Root app → child apps: `platform-*` (controllers, Kyverno, observability, Falco) and `sockshop-<env>`
- [ ] Platform apps in their own `AppProject` with cluster-scoped permissions; app projects without
- [ ] Sync waves order dependencies (CRDs → controllers → policies → apps)
- [ ] Commit to Git → change in the cluster within ~3 minutes (or instantly via webhook)
- [ ] **Sync window for prod** (a change freeze in config): automatic syncs to prod only in working hours, so nothing deploys at 3 a.m. when nobody is watching. Manual syncs still allowed for emergencies:

```yaml
# in the prod AppProject
spec:
  syncWindows:
    - kind: allow
      schedule: "0 8 * * 1-5"      # cron: 08:00, Monday–Friday
      duration: 9h                 # until 17:00
      applications: ["sockshop-prod"]
      manualSync: true             # break-glass syncs still possible (P8-4)
      timeZone: "Europe/Berlin"
```

---

## P8-3a Deployment notifications

**Type:** Task · *Reference 07, SOCKPLAT-17*

Everyone should find out about a failed deploy from a message, not from a user.

- [ ] **Argo CD Notifications** (built into Argo CD) with triggers `on-deployed`, `on-sync-failed`, `on-health-degraded` → Slack channel (or SNS → email)
- [ ] Message includes app, environment, the Git commit/digest and a link to the Argo CD app
- [ ] **CI notifications:** a final job in the release workflow (`if: failure()`) posts commit SHA, author, branch and a run link
- [ ] Slack webhook / token from Secrets Manager via ESO (Argo CD) and as a GitHub **environment** secret (CI), never in values files
- [ ] **Prove it:** break a deploy on purpose → the failure message arrives within a minute

---

## P8-3b Database schema migrations

**Type:** Story · *Reference 07, SOCKPLAT-84 · Use when: the app has a relational database (almost every app)*

**The problem.** New code often needs a new column or table. If the code
deploys before the schema changes, it crashes; if a migration fails halfway,
the database is in an unknown state. Running migrations by hand doesn't scale
and isn't auditable.

**The pattern: expand → migrate → contract.** Never make a change the
*currently running* version can't handle:

| Release | Database change | Code |
|---|---|---|
| N | **Expand**: add the new column (nullable) or table | Old code ignores it; new code writes both old and new |
| N+1 | Backfill data in batches | New code reads the new column |
| N+2 | **Contract**: drop the old column | Nothing uses it any more |

This is what makes zero-downtime rollouts (P9-5) and rollbacks safe: old and
new pods run side by side during a rollout.

**Steps:**

- [ ] Migration tool chosen (ADR): **Flyway** or **Liquibase** (Java, also standalone), **golang-migrate** or **Atlas** (Go / any), or the framework's own (Spring Boot runs Flyway on startup; prefer a separate Job so the app doesn't need DDL rights)
- [ ] Migrations live **in the app repo** next to the code, numbered and immutable (`V0007__add_order_status.sql`); a merged migration is never edited, only followed by a new one
- [ ] Migrations are packaged as an image (or a stage of the app image), so they're **built, scanned, signed and promoted by digest** like the app (P7)
- [ ] **CI test:** a job starts a throwaway database (a `services:` container in GitHub Actions), applies every migration from zero, runs the app's tests against it, and checks the migration can be applied twice safely
- [ ] **Run them with an Argo CD PreSync hook**, as a Job with its own ServiceAccount (`orders-migrate`, P5-4b) and its own DB credentials (DDL user):

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: orders-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync                 # runs before the new pods roll out
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  backoffLimit: 0                                    # a failed migration is not retried blindly
  activeDeadlineSeconds: 600
  template:
    spec:
      serviceAccountName: orders-migrate
      restartPolicy: Never
      containers:
        - name: migrate
          image: <ecr>/sockshop/orders-migrate@sha256:<digest>
          envFrom: [{ secretRef: { name: orders-db-migrate } }]   # from ESO, DDL user
```

- [ ] A failed migration **fails the sync**: the new app version never rolls out, and the notification fires (P8-3a)
- [ ] Concurrency safe: the tool's lock table (Flyway/Liquibase/golang-migrate all lock) stops two runs colliding
- [ ] **Rollback policy written down:** roll back the *code* with `git revert` (the expand step keeps old code working); fix the *schema* forward with a new migration. Only use "down" migrations if you've tested them
- [ ] **Prove it:** (1) a release that adds a column rolls out with zero errors under the load loop; (2) a deliberately broken migration stops the sync and leaves the old version running
- [ ] Managed databases: take an RDS snapshot before migrations in prod (a PreSync step or an automated backup window)

> **Sock Shop note:** the MongoDB services are schemaless, so this applies to
> `catalogue-db` (MySQL) and to any RDS/CloudNativePG database you add.

---

## P8-4 Prove self-healing (it's a security control)

**Type:** Task · *Reference 07, SOCKPLAT-64*

- [ ] `kubectl scale deploy/front-end --replicas=1` → Argo CD reverts it
- [ ] `kubectl delete svc catalogue` → recreated
- [ ] A resource added by hand → removed by `prune`
- [ ] Drift-detection latency measured
- [ ] Written up as a **security control**: an attacker with cluster write access can't make a change stick
- [ ] **Break-glass procedure** documented: how to pause auto-sync for a genuine emergency, who may do it, how it's logged, and how to re-enable it
- [ ] Humans have **read-only** cluster access day to day (P5-1)

---

## P8-4b Post-deploy smoke test

**Type:** Task · *Use when: always*

"Pods are Ready" doesn't mean "checkout works". Run the CUJ after every sync.

- [ ] The CUJ script (P2-8) packaged as a small image (curl/k6), run as an Argo CD **PostSync** hook Job against the environment's own URL
- [ ] On failure: the Argo CD app shows `Degraded`, the notification fires (P8-3a), and for Argo Rollouts the canary aborts (P8-8)
- [ ] Runs in dev and staging on every sync; in prod, a read-only variant (no test orders) or one using a test account
- [ ] **Prove it:** deploy a version with a broken `/carts` route → the smoke test fails within a minute of sync

---

## P8-5 Pod Security Admission everywhere

**Type:** Task

- [ ] Every app namespace labelled `pod-security.kubernetes.io/enforce: restricted` (proven locally in P3-1)
- [ ] System namespaces that need privileges (CNI, CSI, Falco) explicitly labelled `privileged` with an ADR
- [ ] **Negative test:** a privileged pod in `sock-shop` is rejected by the API server

---

## P8-6 Kyverno policies

**Type:** Story

| Policy | Rule | Mode |
|---|---|---|
| `verify-image-signatures` | Images from your ECR must be **Cosign-signed by your release workflow** | Enforce |
| `require-image-digest` | Image references must use `@sha256:` | Enforce |
| `restrict-registries` | Only your ECR + an allow-list of upstream registries | Enforce |
| `disallow-latest-tag` | No `:latest`, no tag-less images | Enforce |
| `require-non-root` | `runAsNonRoot: true` | Enforce (duplicates PSA, with clearer messages) |
| `require-requests-limits` | CPU/memory requests and memory limits present | Enforce |
| `require-probes` | readiness + liveness present | Audit → Enforce |
| `require-labels` | standard label set | Audit |
| `disallow-default-serviceaccount` | `serviceAccountName` set and not `default` (P5-4b) | Enforce |
| `generate-default-deny` | **Generates** a default-deny NetworkPolicy in every new namespace | — |

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: verify-image-signatures }
spec:
  validationFailureAction: Enforce     # newer Kyverno also accepts per-rule failureAction
  webhookTimeoutSeconds: 30
  rules:
    - name: cosign-keyless
      match: { any: [ { resources: { kinds: [Pod] } } ] }
      verifyImages:
        - imageReferences: ["<account>.dkr.ecr.eu-central-1.amazonaws.com/sockshop/*"]
          mutateDigest: true
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/<org>/sockshop/.github/workflows/release.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
```

- [ ] Kyverno installed via Argo CD; its own pods excluded from its policies
- [ ] Every new policy starts in **Audit**; read the PolicyReports, fix violations, then switch to **Enforce** one namespace at a time
- [ ] **Negative tests** (each must be rejected at admission):
  - [ ] an **unsigned** image pushed to ECR by hand
  - [ ] a root container
  - [ ] `nginx:latest`
  - [ ] an image from Docker Hub
  - [ ] a pod without resource limits
- [ ] Kyverno failure mode chosen and documented (fail-closed for app namespaces; system namespaces excluded so a Kyverno outage can't block recovery)

> The same rule now lives in three layers (guide §1): the Dockerfile sets
> `USER 10001` (L1), hadolint/Checkov fail the PR if it's missing (L2), and
> PSA + Kyverno reject a root pod even if applied by hand (L3).

---

## P8-7 NetworkPolicy on EKS

**Type:** Story

- [ ] VPC CNI network policy enabled (P4-6 `enableNetworkPolicy`)
- [ ] The policies proven in P3-7 applied via Argo CD; Kyverno generates default-deny for new namespaces
- [ ] **Negative tests:** `front-end` → `carts-db` blocked; cross-namespace traffic blocked; DNS works
- [ ] The CUJ passes with all policies enforced

---

## P8-8 Progressive delivery with Argo Rollouts

**Type:** Story · *Blocked by P10-1 (needs application metrics in Prometheus)*

- [ ] Argo Rollouts installed; `front-end` converted from a Deployment to a `Rollout`
- [ ] Canary steps: 10 % (5 min) → 50 % (5 min) → 100 %, traffic split with the Argo Rollouts **Gateway API plugin** (it adjusts the `HTTPRoute` backend `weight`s, so the ALB shifts traffic between stable and canary Services)
- [ ] `AnalysisTemplate` querying **Amazon Managed Prometheus** (SigV4 via the Rollouts controller's Pod Identity; P10-1): error rate < 1 %, p95 < 500 ms
- [ ] **Happy path:** a good version promotes on its own
- [ ] **Failure path:** a deliberately broken version **rolls back automatically**, with no human action; rollback time measured
- [ ] Normal rollback for everything else documented: `git revert` the gitops commit → Argo CD syncs
- [ ] **Keep the failed canary around** for a few minutes after an abort (`abortScaleDownDelaySeconds`), so the team can `kubectl exec`/`logs` into it before it's gone; logs stay searchable in CloudWatch Logs afterwards
- [ ] Know the chain for the interview (reference 09): **Prometheus only stores metrics; it never rolls anything back.** Argo Rollouts queries it each analysis interval; on a breach it aborts and sets the route weights back to 100 % stable; the **AWS Load Balancer Controller** (not Argo CD, not Rollouts) calls the AWS API to change the ALB's weighted target groups, so the split happens **per request at the ALB**, not by pod count. Then the notification fires (P8-3a)

---

## P8-9 [OPT] Multi-environment / multi-cluster with ApplicationSets

> **Use when:** you manage more than one environment or cluster (e.g. staging + prod) and are tired of copy-pasting Application objects.

- [ ] One `ApplicationSet` generating `sockshop-dev`, `-staging`, `-prod` from the `envs/` folders (and, later, from a list of clusters)

---

## P8-10 [OPT] vCluster: virtual clusters inside the shared cluster

> **Use when:** you need throwaway preview environments per PR, give developers their own cluster-admin sandbox, or need stronger tenant isolation than namespaces. See the table below; for normal team separation, namespaces are enough.

**Type:** Story · *From the user's plan (SOCKPLAT-59) and workstation notes (plane 3)*

**What it is.** vCluster runs a **full Kubernetes control plane** (its own API
server and data store) as a pod inside **one namespace** of the host cluster.
Users get what looks like their own cluster (they can create namespaces,
CRDs, cluster-wide RBAC), while their pods are actually scheduled on the host's
nodes. Think of a namespace as a **room** in a building and a vCluster as an
**apartment** with its own front door.

| Use | Why vCluster | Plain namespace instead? |
|---|---|---|
| **Per-PR preview environments** | Each PR gets a throwaway cluster: install the whole app + its CRDs, run the CUJ and ZAP, delete. Seconds to create, no new EKS bill | Works only if the app needs no cluster-scoped objects |
| **Per-developer dev clusters** (the inner-loop target from P3-9) | Each dev can be cluster-admin of their own vCluster (install operators, break things) without touching anyone else; Skaffold/Tilt/mirrord point at it | Namespaces can't give someone CRDs or cluster-admin safely |
| **Stronger multi-tenancy** | Tenants can't see each other's resources or the host's | Namespaces + RBAC + NetworkPolicy + quotas are enough for most teams |
| Testing a Kubernetes upgrade or a new CRD/operator | A vCluster can run a different Kubernetes version | — |

- [ ] vCluster installed via Argo CD (or the `vcluster` CLI for experiments); the host namespace gets ResourceQuota, LimitRange, NetworkPolicy and PSA `restricted` (P6-5, P8-5, P8-7), because the vCluster's pods still run there
- [ ] **Isolation test:** from inside the vCluster, `kubectl get ns` shows only its own namespaces; host resources are invisible; a NetworkPolicy blocks traffic to other host namespaces
- [ ] **Preview environments:** a PR workflow creates `vcluster create pr-<number>`, deploys the app by digest, runs the CUJ (P2-8) and a ZAP baseline (P10-6), then deletes it on merge/close. Uses a scoped CI role (P5-2) whose only right is managing vClusters in a `previews` namespace
- [ ] **Kyverno and Falco still apply:** they run on the host and see the synced pods; prove an unsigned image is rejected inside a vCluster too
- [ ] ADR: namespaces vs vCluster for your teams (vCluster adds a control plane per tenant to run, upgrade and back up)

> **Interview line:** "Namespaces are rooms with shared walls; a vCluster is
> an apartment with its own front door. I use namespaces for teams and
> vClusters for disposable preview environments and per-developer clusters."

---

## P8-11 [OPT] Service mesh and mTLS: when and how

**Type:** Story · *Reference 07, SOCKPLAT-60/62*

**What it is.** A service mesh adds a proxy next to (or underneath) every pod
that handles **service-to-service** traffic for you: it encrypts it with
**mutual TLS (mTLS)**, so both sides prove their identity with certificates
the mesh issues and rotates automatically; it can **retry**, **time out** and
**split** traffic; and it reports golden metrics (requests, errors, latency)
for every call. Job 5 in the networking article.

**Use when** (any one is enough):

- a compliance rule or customer requires **encryption in transit inside the cluster**, not just at the load balancer
- you want **zero-trust service identity**: "`orders` may call `payment`" enforced by cryptographic identity, not by IP-based NetworkPolicy alone
- you have **many services** (roughly 10+) and need consistent retries, timeouts and per-call metrics without changing every app
- you need **L7 traffic policy between services** (header routing, fault injection, per-route canaries)

**Don't use it yet when:** you have a handful of services, the team is new to
Kubernetes, or NetworkPolicy + TLS at the ALB meets your requirements. A mesh
is another control plane to run, upgrade and debug, and every request now
passes through extra proxies.

| Option | Model | Pick when |
|---|---|---|
| **Linkerd** | Small Rust sidecar per pod | **Simplest to learn and run**; mTLS on by default. Note: the open-source project ships *edge* releases; versioned *stable* releases come from Buoyant's distribution (check its current licence terms) |
| **Istio (ambient mode)** | No sidecars: a per-node L4 proxy + optional L7 "waypoint" proxies | Most features, lower per-pod overhead than sidecars |
| **Cilium service mesh** | eBPF in the kernel + per-node Envoy | You already use Cilium as the CNI |

**How (Linkerd, the learning default):**

- [ ] Linkerd installed by Argo CD; its trust anchor certificate managed by **cert-manager** (not the 1-year default left to expire)
- [ ] `sock-shop` namespace annotated `linkerd.io/inject: enabled`; pods restarted; every pod shows the proxy container
- [ ] **Prove mTLS:** `linkerd viz edges deploy -n sock-shop` shows every connection `SECURED`; a `tcpdump` on a node shows encrypted traffic between pods
- [ ] **Authorization policy:** only `orders` may call `payment` (Linkerd `Server` + `AuthorizationPolicy`); **negative test:** a debug pod's call to `payment` is denied even though NetworkPolicy would allow it
- [ ] Keep NetworkPolicy too: L3/L4 policy and mesh identity are two layers (defence in depth)
- [ ] Measure the cost: p95 latency and memory per pod, before vs after; record them
- [ ] ADR: sidecar vs sidecarless (Linkerd vs Istio ambient vs Cilium), using your measurements; the conditions under which you'd switch

---

## Prove it

- [ ] `grep -r "kubectl\|helm upgrade" .github/` in the app repo → nothing
- [ ] All five admission negative tests rejected; all three self-heal drills reverted
- [ ] A manually pushed, unsigned image with a valid tag **cannot run**, even though it's in your own registry

## Common mistakes

| Mistake | Seen in |
|---|---|
| CI deploys with kubectl; no GitOps | 02, 04 |
| Argo CD `project: default`, installed once by hand, never hardened | 06, 01, 05 |
| Admission control "future work" | every reference |
| Policies straight to Enforce on day one (breaks the cluster) | (guide warns) |
| NetworkPolicy on a CNI that doesn't enforce it | — |
| Rollback = rebuild an old commit (new, untested image) | — |
