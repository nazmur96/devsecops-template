# Phase 10: Observe & operate

**Goal:** you can see what the system is doing (metrics, logs, traces), you
get paged for what users feel (SLO burn), you detect attacks while they
happen (runtime security), and you've **proven** you can survive failures and
restore from backup.

**Why:** only references 04 and 06 had observability at all, and none had
SLOs, runtime detection, backups, chaos or DAST. The user's plan (reference
07) is the only source for the drills here. A control you haven't seen fire
is a control you don't know works.

---

## P10-0 Choose the observability path

**Type:** Decision (ADR)

**The rule that makes this decision cheap:** instrument the apps with
**OpenTelemetry** (the OTel Operator's auto-instrumentation, or OTel SDKs) and
send everything through a **collector**. Then the backend is a **collector
configuration**, not a code change, and you can switch later.

The options seen across the reference projects:

| | **A. AWS-managed** (reference 06, retail store) | **B. Self-hosted Prometheus + Loki** (reference 08) | **C. Self-hosted Grafana stack (LGTM)** (reference 04, user plan 07) |
|---|---|---|---|
| Collector | **ADOT** (AWS Distro for OpenTelemetry, EKS add-on) / CloudWatch agent | Promtail + Prometheus scrape | OTel Collector / Alloy + Prometheus scrape |
| Metrics | **Amazon Managed Service for Prometheus (AMP)** | kube-prometheus-stack | kube-prometheus-stack (or Mimir at scale) |
| Logs | **CloudWatch Logs** | Loki via `loki-stack` (⚠️ deprecated chart; Promtail deprecated) | Loki (`grafana/loki`, S3 storage) via Alloy |
| Traces | **AWS X-Ray / CloudWatch Application Signals** | ❌ declared, not built | Tempo (S3 storage) or Jaeger |
| Dashboards | **Amazon Managed Grafana (AMG)** | Grafana | Grafana |
| Alerts | AMP alert manager → SNS; CloudWatch alarms; AMG alerting | Alertmanager | Alertmanager |
| You operate | Almost nothing: AWS runs storage, HA, scaling, upgrades | Prometheus, Loki, Grafana, storage | Everything |
| Cost model | Per metric sample ingested, per GB of logs, per trace, per Grafana user | Nodes + EBS + S3 | Nodes + EBS + S3 |
| Lock-in | High | Low | Low |

**Template default: A (AWS-managed)**, so you learn observability *concepts*
(signals, SLOs, burn rates, trace-driven diagnosis) without also operating
a storage system. **Track C comes twice:** a free demo-grade version on KinD first (P3-10), and production-grade on EKS later (P10-13): the same OTel
instrumentation pointed at a self-hosted stack, which teaches what the managed
services hide and runs on your Talos homelab too. B is C without traces;
don't stop there.

Whichever you pick:

- [ ] Every app instrumented with OpenTelemetry (no vendor SDKs, no X-Ray SDK), so switching to C later is a collector change
- [ ] Logs, metrics and traces share attributes (`service.name`, `k8s.namespace.name`, `deployment.environment`) so you can jump between them
- [ ] Collector identities via **Pod Identity**, scoped to the one AMP workspace / log group prefix / bucket they write to
- [ ] Alert destinations (SNS topics, Slack webhooks, PagerDuty keys) never in Helm values in Git (reference 08 kept a webhook placeholder there); SNS topic ARNs from Terraform, webhook secrets from Secrets Manager via ESO
- [ ] Retention set for every signal on day one (CloudWatch log groups default to **never expire**)
- [ ] **Budget alarm for observability spend**: managed observability is billed by volume, so a chatty debug log or a high-cardinality metric shows up on the bill
- [ ] ADR with the cost estimate for your traffic

---

## P10-1 Metrics (Amazon Managed Prometheus + Amazon Managed Grafana)

**Type:** Story · *Reference 06, section 20*

- [ ] **AMP workspace** in Terraform
- [ ] Metrics collection, pick one (ADR):
  - **AMP managed collector** (agentless scraper that AWS runs against your cluster): nothing to run in-cluster; configure scrape jobs for kube-state-metrics, node-exporter (both EKS add-ons, reference 06) and your services
  - **ADOT collector** (EKS add-on, needs cert-manager) with the Prometheus receiver → `prometheusremotewrite` exporter with SigV4 auth to AMP; Pod Identity role with `aps:RemoteWrite` on that workspace only
- [ ] Apps expose **application** metrics (requests, errors, latency histograms), not just CPU, via OTel or a `/metrics` endpoint
- [ ] **Amazon Managed Grafana** workspace, logins through **IAM Identity Center** (P5-1), AMP and CloudWatch added as data sources (service-managed permissions)
- [ ] Dashboards **as code** (Grafana JSON in Git, applied by Terraform or the Grafana provider), never only clicked together: **RED** per service (Rate, Errors, Duration), **USE** per node (Utilisation, Saturation, Errors)
- [ ] **Cost watch:** AMG bills per active editor/viewer per month; AMP per sample ingested; drop unused high-cardinality metrics at the collector
- [ ] **Prove it:** a PromQL query in AMG shows the p95 latency of `orders` per pod

---

## P10-2 Logs (CloudWatch Logs)

**Type:** Story

- [ ] Container logs shipped to CloudWatch, pick one (ADR):
  - the **Amazon CloudWatch Observability** EKS add-on (CloudWatch agent + Fluent Bit, also gives Container Insights), with Pod Identity
  - or the **ADOT collector** with a logs pipeline
- [ ] One log group per environment/namespace (e.g. `/sockshop/dev/application`), **retention set** (7 days dev, 30 days prod)

> **Other log stack you'll hear about: EFK / ELK** (Fluent Bit or Fluentd →
> Elasticsearch → Kibana; reference 09). The AWS-managed version is **Amazon
> OpenSearch Service** (OpenSearch Dashboards instead of Kibana), fed by Fluent
> Bit or the ADOT collector. **Use it instead of CloudWatch Logs when** you need
> heavy full-text search and aggregations over large log volumes, long
> retention at a lower cost per GB, or the team already knows Kibana. It's a
> cluster you size and pay for even when idle (or use OpenSearch Serverless);
> CloudWatch Logs needs no sizing. Pick one per environment (ADR).
- [ ] Apps log structured JSON to stdout, including the **trace ID** (P10-3), so a log line links to its trace
- [ ] **Prove it:** one CloudWatch Logs Insights query returns all errors for `orders` across every pod in the last hour:
  ```
  fields @timestamp, kubernetes.pod_name, log
  | filter kubernetes.container_name = "orders" and log like /ERROR/
  | sort @timestamp desc
  ```

---

## P10-3 Traces (X-Ray / CloudWatch Application Signals)

**Type:** Story · *Blocks the hard-mode track; diagnosing many services without traces is guesswork (reference 07)*

- [ ] **Auto-instrumentation** for Java and Node via the ADOT/OTel Operator (`Instrumentation` resource + pod annotation, reference 06); Go via the OTel SDK
- [ ] Traces exported to **AWS X-Ray** through the ADOT collector (Pod Identity role allowed to write trace segments only); or enable **CloudWatch Application Signals**, which builds on the same OTel data and adds service maps and **SLOs** (useful for P10-4)
- [ ] Sampling: 100 % in dev; ~10 % in prod (keep all errors)
- [ ] **Filter health-check spans** in the collector before export (reference 06 cut trace volume ~85 % this way; with managed tracing, that's money)
- [ ] The full checkout CUJ produces **one connected trace** in the service map, including across the RabbitMQ hop (context propagated in message headers)
- [ ] **Drill:** inject 500 ms latency into one service (Chaos Mesh, P10-7) → find it from the trace alone

---

## P10-3b Synthetic monitoring (the CUJ, every 5 minutes, in prod)

**Type:** Task · *Use when: always*

Metrics tell you pods are healthy; only a real request tells you **checkout
works**. A synthetic check runs your CUJ from outside, all the time, so you
find out before users do (even at 3 a.m., when there's no traffic to alert on).

- [ ] **CloudWatch Synthetics canary** (Terraform `aws_synthetics_canary`) running a script version of the CUJ (P2-8) against prod every 5 minutes, from outside the VPC
- [ ] Uses a dedicated **test account** in the app; never creates real orders (or creates ones flagged as test)
- [ ] Alarm on 2 consecutive failures → SNS (P10-4); canary artifacts (screenshots, HAR files) kept 7 days
- [ ] Also a simple availability check per environment (dev/staging every 15 minutes is enough)
- [ ] Canary success rate is a ready-made **availability SLI** for P10-4
- [ ] **Prove it:** break the checkout route in staging → the canary alarm fires within 10 minutes

---

## P10-4 SLOs, burn-rate alerts and runbooks

**Type:** Story

| SLI | SLO (from P0-10) |
|---|---|
| front-end availability (non-5xx / total) | 99.9 % / 30 days |
| front-end p95 latency | < 500 ms |
| checkout CUJ success rate | 99.5 % |

| Alert window | Burn rate | Severity | Action |
|---|---|---|---|
| 1 h (+ 5 m) | 14.4× | Critical | Page |
| 6 h (+ 30 m) | 6× | Warning | Ticket |
| 3 d | 1× | Info | Review |

- [ ] SLOs as code: the Prometheus recording + burn-rate alerting rules you wrote locally in P3-10 (or generated with Sloth), uploaded to **AMP** with `aws_prometheus_rule_group_namespace` in Terraform. Alternative: **CloudWatch Application Signals SLOs** if you enabled it in P10-3 (ADR)
- [ ] Error-budget dashboard in Amazon Managed Grafana
- [ ] **AMP alert manager** (`aws_prometheus_alert_manager_definition`) routes critical → an **SNS topic** → page (email/SMS or a phone push for learning), warning → SNS → Slack (via AWS Chatbot / Amazon Q Developer in chat applications)
- [ ] **Every alert links to a runbook** in the repo (`docs/runbooks/<alert>.md`). No runbook, no alert
- [ ] Also alert on: certificate expiry < 14 days, PVC > 80 % full, Argo CD app degraded/out-of-sync, Kyverno policy violations, node not ready
- [ ] **Prove it:** inject errors → the fast-burn alert fires within its window

---

## P10-5 Runtime threat detection (GuardDuty + Falco)

**Type:** Story

Matching the managed-first choice, two layers:

| | **Amazon GuardDuty** (EKS Protection + Runtime Monitoring) | **Falco** |
|---|---|---|
| What | AWS-managed: analyses EKS audit logs and runs a managed runtime agent (EKS add-on) | Open-source eBPF syscall rules you write and tune |
| You get | Findings like "credentials used from a malicious IP", "crypto-miner process", "privileged container launched", with no rules to maintain | Your own rules, tuned to your app; alerts in seconds |
| Cost | Per vCPU monitored + audit-log volume | Nodes only |
| Learning value | Low (it's a switch) | High (you learn what "suspicious" looks like) |

- [ ] **GuardDuty** enabled with EKS audit-log monitoring and **Runtime Monitoring** (agent managed as an EKS add-on); findings → EventBridge → SNS (P10-4)
- [ ] **Drill:** run GuardDuty's sample findings (or the AWS tester scripts) and confirm the alert path end to end
- [ ] Falco as a DaemonSet (modern eBPF driver), managed by Argo CD
- [ ] Rules: shell spawned in a container, write below `/etc`, read of `/etc/shadow` or cloud credential files, unexpected outbound connection, `kubectl exec` into prod
- [ ] Alerts via **Falcosidekick** → SNS / Slack / CloudWatch Logs
- [ ] **Simulated attacks** (each alert must fire within seconds):
  - [ ] `kubectl exec -it deploy/catalogue -- sh` (or into a debug container)
  - [ ] `touch /etc/evil` in a container with a writable `/etc`
  - [ ] `cat /etc/shadow`
- [ ] False positives from normal operation tuned out (document each exception)
- [ ] One runbook per rule: what it means, how to contain (cordon node, isolate pod with a NetworkPolicy), how to investigate
- [ ] Falco **detects, it doesn't block.** Note **Tetragon** if you need in-kernel enforcement

---

## P10-6 Dynamic testing (DAST)

**Type:** Story

- [ ] **ZAP baseline** (`zap-baseline.py`, passive: spider + observe, no attacks) runs after every staging deploy; fails on high-risk findings
- [ ] **ZAP full scan** (`zap-full-scan.py`, active attacks) nightly against **staging only, never prod**; or `zap-api-scan.py` with the OpenAPI spec
- [ ] Authenticated scan configured (otherwise ZAP only tests the login page)
- [ ] Findings triaged like other scanners (P7-7)

---

## P10-7 Load and chaos

**Type:** Story

- [ ] **k6** script of the full CUJ (from P2-8); profile: ramp 0→50 users (2 min), sustain (5 min), spike to 200 (2 min), recover
- [ ] Pass: p95 < 500 ms throughout, zero 5xx, **HPA scales before latency degrades**; bottleneck identified and written down
- [ ] The pass conditions written as **k6 thresholds** (`http_req_duration: ['p(95)<500']`, `http_req_failed: ['rate<0.01']`), so k6 itself exits non-zero when the performance budget is broken
- [ ] **k6 in CI on demand:** a `workflow_dispatch` (and nightly) workflow runs the load test against staging; a smaller smoke-load (e.g. 20 users for 2 minutes) runs after every staging deploy, so a slow query is caught before prod. Results kept as artifacts so you can compare runs
- [ ] **Chaos Mesh** (or AWS FIS for node/AZ faults) experiments, each run **under load**:

| Experiment | Expected |
|---|---|
| Kill a random `catalogue` pod | Replaced < 30 s, no user-visible errors |
| All `carts` replicas down | Cart degrades; browsing still works |
| +500 ms latency `orders` → `orders-db` | p95 rises; no cascading failure |
| Partition `rabbitmq` | Shipping queues locally; drains on recovery |
| Drain a node / an AZ | Pods reschedule; PVCs reattach (same AZ) |

- [ ] Observed vs expected recorded; worse-than-expected → a follow-up ticket
- [ ] **Did the alerts fire?** Silence during an outage is a finding

---

## P10-8 Post-mortem exercise

**Type:** Story

- [ ] Someone else (or a script with a hidden choice) breaks something
- [ ] You investigate using only the telemetry: Amazon Managed Grafana (AMP metrics), CloudWatch Logs Insights and X-Ray traces (or Grafana/Loki/Tempo locally)
- [ ] Blameless post-mortem: impact, timeline from telemetry, root cause, contributing factors, action items
- [ ] Observability gaps that slowed you down become tickets
- [ ] **Also write an RCA for every real problem you hit while building** (`docs/rca/YYYY-MM-DD-<title>.md`). Reference 08 does this well: "PVCs pending because no EBS CSI driver for gp3", "Falco crash on small nodes". These files are your best interview stories

---

## P10-8b End-of-life watch

**Type:** Task · *Reference 08's EOL-check workflow*

- [ ] Weekly scheduled workflow that checks runtimes (Node, Java, Go), base images, the EKS version, add-on and chart versions against their end-of-life dates (e.g. the endoflife.date API) and opens an issue for anything within 90 days of EOL
- [ ] Two of the reference projects shipped already-old pins (EKS 1.30, MySQL 5.7, Node 18): this catches that automatically

---

## P10-9 Backup and disaster recovery

**Type:** Story

- [ ] **Velero** with an S3 location + EBS snapshots (Pod Identity); daily schedule, 30-day retention, the app namespace + its PVs
- [ ] Managed data stores: automated backups + PITR enabled (P4-9)
- [ ] Git is the backup for cluster *configuration* (Argo CD can rebuild it); Velero covers *state*
- [ ] **DR drill:** delete the whole `sock-shop` namespace → restore → **all data present** → CUJ passes. RTO measured against the P0-10 target
- [ ] **Full-rebuild drill:** `terraform destroy` everything except the state bucket and backups → rebuild from Git + restore → CUJ passes
- [ ] Runbooks for both

---

## P10-10 Day-2 operations

**Type:** Story

- [ ] **Upgrades:** EKS minor version (control plane → add-ons → nodes), one version at a time, in dev first; PDBs keep it safe; check deprecated APIs first (`kubent` or `pluto`)
- [ ] **Registry rescans:** ECR enhanced scanning or a nightly Trivy job over running images; a new CRITICAL in prod → a ticket with an SLA
- [ ] **Cost:** budget alerts, Cost Explorer by tags, `kube-downscaler` / scheduled scale-to-zero for dev at night, OpenCost per namespace; monthly review of oversized requests, NAT data, cross-AZ traffic, idle nodes
- [ ] **Certificates and secrets rotation** schedule written down

---

## P10-10b Delivery metrics (DORA)

**Type:** Task · *Use when: always. It's how teams (and interviewers) measure delivery*

The four **DORA metrics** say how well your delivery system works. You
already have the data: the gitops repo's history (P1-1), Argo CD sync events,
and incidents (P10-8).

| Metric | Question | How to measure here | Good target |
|---|---|---|---|
| **Deployment frequency** | How often do we deploy to prod? | Commits to `envs/prod/` per week | Daily to weekly |
| **Lead time for changes** | Commit → running in prod | App commit timestamp → prod sync timestamp of that digest | < 1 day |
| **Change failure rate** | What share of deploys cause a problem? | Prod deploys followed by a revert, rollback or incident / all prod deploys | < 15 % |
| **Time to restore** | How fast do we recover? | Incident start → resolved (from your RCAs and post-mortems) | < 1 hour |

- [ ] A small script or scheduled workflow computes all four from Git + Argo CD + `docs/rca/`, and writes them to a dashboard or a monthly Markdown report
- [ ] Recorded at the start (likely "manual, monthly") and at the end of the project: the **before/after** numbers are your strongest resume bullets (P11-3)

---

## P10-11 [OPT] Multi-region failover

> **Use when:** the business needs to survive a **whole region** going down, or serves users globally. Multi-AZ (P6) already covers most outages; multi-region roughly doubles cost and is mostly a data-replication problem.

- [ ] Minimal standby stack in a second region from the same Terraform (a different tfvars)
- [ ] Route 53 failover records with health checks; failover drill timed; failback tested
- [ ] **Data replication strategy written down, including its gaps.** Routing is the easy part; data is the hard part (reference 03 skipped it)

---

## P10-12 [OPT] Hard mode: diagnosis at scale

> **Use when:** tracing works (P10-3) and you want diagnosis practice on a system much bigger than yours.

From reference 07's plan, once tracing works:

- [ ] **TeaStore** (5 services + a service registry): deploy, wire tracing, inject one fault, diagnose from traces
- [ ] **TrainTicket** (~40 services) + its fault-replication benchmark: diagnose F1 (async ordering race), F3 (JVM limit above the cgroup limit, the `MaxRAMPercentage` lesson), F5 (thread-pool exhaustion cascade) **before** reading their write-ups; record time to diagnosis

---

## P10-13 [OPT, later] Production-grade self-hosted observability on EKS

> **Use when:** you're comfortable with the managed path and want the 'without AWS' skill, run on-prem or another cloud, or managed observability gets expensive at your volume.

**Type:** Epic · *Do this once the managed path (P10-0..4) is familiar. It's the "without AWS" skill: what AMP, CloudWatch, X-Ray and AMG do for you*

You already know the components from the local demo (P3-10). The difference
is everything a demo skips:

- [ ] Installed by **Argo CD** from pinned chart versions (current charts: `kube-prometheus-stack`, `grafana/loki`, `grafana/tempo`, `grafana/alloy`; not the deprecated `loki-stack`/Promtail)
- [ ] **Object storage:** Loki and Tempo store chunks/blocks in **S3** (one bucket each, lifecycle rules, Pod Identity roles scoped per bucket), so data survives pod and node loss
- [ ] **HA:** Prometheus 2 replicas (or Prometheus agent mode → Mimir/Thanos for long-term, deduplicated storage); Loki simple-scalable with ≥ 2 replicas per component; Alertmanager clustered (3)
- [ ] **Retention** per signal and environment (e.g. metrics 15 d local + long-term in object storage; logs 7/30 d; traces 7 d)
- [ ] **Cardinality guardrails:** label limits in Loki, metric relabelling to drop unused series, per-tenant limits
- [ ] **Security:** Grafana SSO + admin secret from ESO; no public endpoints; alert receivers from Secrets Manager
- [ ] **Self-monitoring:** alerts on the observability stack itself (ingestion errors, disk full, compaction failing). Who watches the watchers?
- [ ] **Upgrades and backups:** a chart upgrade rehearsed in dev; Grafana dashboards and alert rules in Git (they *are* the backup)
- [ ] **Switch:** point the OTel/ADOT collector's exporters at the self-hosted endpoints; **no app changes** (that's the OTel payoff)
- [ ] ADR comparing your measured monthly cost and operating time with the managed path

---

## Prove it

- [ ] One dashboard shows an SLO, its error budget, and a link from a slow request to its trace and its logs
- [ ] Falco alerts on a shell in a pod within seconds; a GuardDuty sample finding reaches the SNS alert path
- [ ] Namespace deleted and fully restored within the RTO target
- [ ] Chaos results recorded, including whether the alerts fired

## Common mistakes

| Mistake | Seen in |
|---|---|
| No observability at all | 01, 02, 03, 05 |
| OTel demo stack shipped but never used | 05 |
| Alerting on CPU instead of user-facing symptoms | (user's Section 12 list) |
| Alerts with no runbook | — |
| Backups never restored | everyone |
| DAST with `zap-baseline` described as "active attacks" | (original notes) |
| Multi-region presented as routing only | 03 |
