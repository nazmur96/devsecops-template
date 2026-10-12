# Phase 11: Present it

**Goal:** someone who has never seen the project understands what you built,
why, and how well it works in **5 minutes** from the README, and you can tell
the story in an interview in 60 seconds, then go deep on any part.

**Why:** a working platform nobody can evaluate is invisible. References 04
and 05 end with resume and interview prep for this reason. **Evidence beats
claims**: a screenshot of Kyverno rejecting an unsigned image says more than
"implemented supply-chain security".

---

## P11-1 The README

**Type:** Story

Structure:

1. **One-paragraph summary**: what the app is, what you built around it
2. **Architecture diagram**: user → Route 53 → WAF → ALB → pods → data stores; plus CI → ECR → Git → Argo CD → cluster
3. **The three layers table** (baseline / pipeline / enforcement / runtime) with the tools in each
4. **How to run it**: `devpod up .` → `make cluster-up deploy` (local) → `make infra-up` (AWS), with the cost per hour
5. **Evidence** (P11-2), linked
6. **Decisions**: link to `docs/adr/` with one line per ADR
7. **Trade-offs and what I'd do with more time**: honest list (P11-4)

- [ ] A teammate reads it cold and can explain the system back to you

---

## P11-2 Evidence folder

**Type:** Task · `docs/evidence/`

One screenshot or log excerpt per claim, with a caption:

- [ ] PR blocked by Gitleaks / Trivy / Checkov / quality gate (P7)
- [ ] Image never reached ECR because the scan failed (P7-3)
- [ ] `cosign verify` output for a running image (P7-3)
- [ ] Kyverno rejecting an unsigned image, a root pod, `:latest` (P8-6)
- [ ] Argo CD reverting a manual `kubectl` change (P8-4)
- [ ] Zero dropped requests during a rolling update / node drain (P3-8, P6)
- [ ] Canary auto-rollback (P8-8)
- [ ] Falco alert for a shell in a pod (P10-5)
- [ ] SLO dashboard + a burn-rate alert firing (P10-4)
- [ ] Namespace restore with the measured RTO (P10-9)
- [ ] Load test result with HPA scaling (P10-7)
- [ ] Monthly cost before/after Spot and scale-to-zero (P6-6, P10-10)

---

## P11-3 Numbers for your resume

**Type:** Task · *Reference 04 stresses quantified impact; reference 05 has a resume section*

Measure these during the project; they become resume bullets:

| Metric | Example |
|---|---|
| DORA metrics (P10-10b) | deploys **daily** (was monthly); lead time **12 min**; change failure rate **8 %**; time to restore **14 min** |
| Deploy lead time | commit → prod in **12 min** (was manual) |
| Image size | Java images **640 MB → 180 MB**; Go **< 20 MB** distroless |
| Vulnerabilities | **0 HIGH/CRITICAL** in prod images; N blocked at the gate |
| Recovery | namespace restore RTO **14 min** (target < 1 h); Postgres failover **22 s** |
| Availability | zero failed requests across drain / AZ / Spot drills |
| Cost | **–58 %** with Spot + consolidation + dev scale-to-zero |
| Security | **0** long-lived credentials; 100 % of images signed and verified at admission |

Bullet pattern: **verb + what + how + measurable result**

> Built a GitOps delivery platform on EKS (Terraform, Argo CD, Kyverno) where
> every image is scanned, SBOM-attested and Cosign-signed in CI and verified
> at admission; reduced commit-to-production time to 12 minutes with zero
> long-lived credentials.

> Hardened 8 polyglot microservices (Go, Java, Node) to non-root,
> read-only-root-filesystem images, cutting image size by 72 % and removing
> all HIGH/CRITICAL CVEs.

> Designed SLOs with multi-window burn-rate alerting and proved recovery with
> Velero DR drills (14-minute RTO) and Chaos Mesh experiments under load.

- [ ] 3–5 bullets with **your real numbers**; no number you can't explain

---

## P11-4 Trade-offs you made on purpose

**Type:** Task

Interviewers trust candidates who know where their project cuts corners. List
each with the production alternative:

| In this project | In production I'd… | ADR |
|---|---|---|
| Public subnets, no NAT in dev | Private subnets + NAT per AZ + VPC endpoints | 0005 |
| One account, namespaces per env | Separate AWS accounts per environment (Organizations + SCPs) | 0003 |
| MongoDB/RabbitMQ in-cluster | Managed services or operators with backups and PITR | 0006 |
| AWS-managed observability (AMP, AMG, CloudWatch, X-Ray) | Same, or self-hosted Grafana stack when cost at scale or portability (on-prem, other clouds) matters; OTel instrumentation makes it a collector change | 00xx |
| Single region | Multi-region only with a data replication design | 00xx |

---

## P11-5 The interview story

**Type:** Task

**60-second version** (the guide's three-layer pitch, now with your numbers):

> "I inherited a polyglot microservice app and took it to production on EKS.
> I started with a plan: inventory, a critical user journey as the universal
> test, sizing from measured load, and ADRs for every big decision.
>
> **The baseline** is secure by design: hardened non-root images, Terraform
> for everything including EKS Access Entries and add-ons, Pod Identity for
> every service, secrets from Secrets Manager, NetworkPolicies, and workloads
> with probes, HPA, PDBs and zone spread, proven with drain and AZ drills.
>
> **The pipeline** checks every change cheapest-first, builds each image
> once, scans it *before* pushing, attaches an SBOM, signs it with Cosign, and
> promotes the same digest from dev to prod. CI never touches the cluster.
>
> **Enforcement:** Argo CD deploys from Git and reverts drift, and Kyverno
> only admits images signed by our release workflow. In production, SLO
> burn-rate alerts, Falco and tested Velero restores cover what scanners can't."

**"Walk me through your end-to-end CI/CD pipeline"** (asked in almost every
DevOps interview; reference 09). Don't list tools. Pick one stack and walk
the path, mentioning security, tagging, safe deployment and rollback:

> "Developers work on feature branches; pre-commit hooks run Gitleaks
> locally, but hooks can be skipped, so the **PR pipeline** runs lint, unit
> tests, Gitleaks, SAST with a quality gate (coverage on new code ≥ 80 %, no
> new criticals), dependency and IaC scans. Nothing merges without passing
> checks and an approval.
>
> On merge, GitHub Actions assumes an AWS role through **OIDC**, so there are
> no stored keys. It builds a **multi-stage** image with cached dependencies
> and layers, **scans it with Trivy before pushing**, generates an SBOM,
> **signs** it, and pushes it to ECR with **immutable tags = the commit SHA**.
> Inspector keeps rescanning it.
>
> CI never touches the cluster. A bot opens a PR in the **separate GitOps
> repo** with the new digest for staging; it auto-merges, Argo CD syncs it,
> and staging runs integration tests and a ZAP scan. A **promotion PR** copies
> the **same digest** to prod, and CODEOWNERS makes it need a lead's approval:
> build once, deploy many.
>
> In prod, **Argo Rollouts** does a canary: 10 % of traffic, split per request
> at the ALB by the Load Balancer Controller. Rollouts queries Prometheus;
> if the error rate or p99 latency breaches the threshold, it **aborts
> automatically** and shifts traffic back. Schema changes follow
> **expand/contract**, so old and new versions run side by side. Secrets come
> from Secrets Manager via External Secrets, never the image. Kyverno only
> admits images signed by our pipeline, and Slack gets the result."

Then let them pick a thread. Prepare a 5-minute deep dive for each:

- [ ] Why OIDC and not keys (the 4-scenario story from reference 04 / P5-2)
- [ ] aws-auth vs Access Entries (P4-4)
- [ ] Tags vs digests; signing; promotion without rebuilding (P7)
- [ ] The probe trap and `startupProbe` (P3-3)
- [ ] PDB + HPA + spread, and the 5-minute eviction delay (P6)
- [ ] Zero-downtime at the ALB: readiness gates, preStop, deregistration delay (P9-5); why Gateway API over Ingress (P9 ADR)
- [ ] A real failure you diagnosed from traces (P10)
- [ ] The trade-offs table (P11-4)

**"Tell me about a challenge"**: pick a real one with a number, e.g. *"JVM
pods were OOM-killed while reporting free heap; the JVM didn't respect the
cgroup limit. Fixed with `MaxRAMPercentage`; verified with a 512 Mi memory
test, then later recognised the same fault (F3) in TrainTicket."*

### Scenario bank: practise thinking out loud

Interviewers use scenarios to test **judgement about where a responsibility
sits**, not trivia. Answer with this structure:

1. **Say which constraint is really binding** ("the binding constraint isn't scale, it's the compliance boundary").
2. **Say what you'd check first, and why that order.**
3. **Give the fix, then its ongoing cost.**
4. **Say what would change your answer.** This is the strongest signal that you're reasoning, not reciting.

Try each one aloud before opening it.

<details><summary><b>1. A pipeline role has AdministratorAccess but gets "AccessDenied … because of a Service Control Policy".</b></summary>

An SCP is an organisation-level **ceiling**, not a permission; account IAM can only grant inside it. The fix is in AWS Organizations (amend the SCP or move the account), not in the role. First ask *why* the SCP denies it: if it's a deliberate guardrail, change the pipeline instead. (P5-8, P12)
</details>

<details><summary><b>2. Traffic goes from 100 to 50,000 req/s in two minutes; pods scale at 70 % CPU and fall over first.</b></summary>

CPU is a lagging signal: scale on requests per target, or pre-scale on a schedule if the event time is known. Karpenter adds nodes in about a minute. Push static assets to a CDN (P9-7). Check connection pooling: a sudden fleet of new pods opening unpooled connections takes the database down instead (P4-9). (P6-2, P10-7)
</details>

<details><summary><b>3. Developers want permanent access keys and an SSH bastion to debug a private service.</b></summary>

Decline both by **leading with the alternative**: SSO for short-lived credentials with the same SDK code locally and in the pod; Session Manager or `kubectl debug` for a shell, with no open ports and every session logged. If getting access is genuinely slow, fix that too, or the workaround comes back. (P5-1)
</details>

<details><summary><b>4. Every release drops connections for about 30 seconds.</b></summary>

Two causes, usually together: the rollout removes old pods before new ones take traffic (readiness gates, maxUnavailable, canary; P9-5, P8-8), and the drain time is wrong (the deregistration delay must exceed the app's drain time, and the **app must handle SIGTERM**). The last part is the developers', so it's a shared fix. (P0-2b, P3-8)
</details>

<details><summary><b>5. A migration deploys fine and the site breaks instantly.</b></summary>

Running code can't read the new schema. Enforce **expand/contract**: add without removing, deploy code that reads both, drop the old column in a later change, and agree this as a contract in advance. Require an undo path in every schema PR. (P8-3b, P0-11b)
</details>

<details><summary><b>6. The bill doubled. Find the waste now.</b></summary>

Cost by service and tag first, then the five-minute audit: NAT processing (gateway endpoints are free), cross-AZ transfer, orphaned volumes and snapshots, idle load balancers, log ingestion without retention. Then structure: is the floor on a Savings Plan, and is burst work still On-Demand? (P10-10c)
</details>

<details><summary><b>7. A fintech: 6 services, 4 engineers, one client requiring SOC 2. How many clusters?</b></summary>

High compliance, low headcount. Keep most things in one well-isolated setup on managed services, but if one service touches regulated data it gets its own cluster and account however small. Name the trigger for every extra cluster. (Template README "Scale the template", P0-6, P12)
</details>

<details><summary><b>8. A partner bank needs to allowlist your IP and requires mutual TLS terminated by your app. ALB or NLB?</b></summary>

NLB: fixed Elastic IPs, and TLS passthrough so mTLS ends in your app. The ALB's IPs rotate and it terminates TLS itself. (P9, "ALB, NLB or API Gateway?")
</details>

<details><summary><b>9. Your team runs 12 services on ECS Fargate; someone proposes moving to EKS. Argue both sides.</b></summary>

Use the P0-13 framework. Concede the case for staying first ("ECS works for our shape"); justify a move only with requirements that exist (admission policy for an auditor, several teams needing isolation, progressive delivery); state the ongoing cost (upgrades, more failure modes); name your own bias; give the reversal cost. (P0-13)
</details>

<details><summary><b>10. A video-transcoding service sits at 100 % CPU by design and gets bursts of thousands of jobs.</b></summary>

CPU is meaningless here: scale on queue depth, to zero when idle, on Spot with interruption handling (KEDA, P6-2). Then ask if Lambda fits better; it depends on whether a job can exceed 15 minutes. Workers must be idempotent. (P0-13)
</details>

<details><summary><b>11. Two VPCs joined by a Transit Gateway: traffic works one way and times out the other.</b></summary>

Asymmetric routing. Attaching isn't routing: both VPC route tables and the TGW route table need entries in both directions. ([DAY-2-OPERATIONS.md](DAY-2-OPERATIONS.md) incident 16)
</details>

<details><summary><b>12. The pod's role explicitly allows <code>s3:GetObject</code> and every request returns 403.</b></summary>

An explicit deny always wins: check the bucket policy, the KMS key policy if the bucket is encrypted, and the VPC endpoint policy. All of them must agree. (P5-8)
</details>

---

## P11-6 [OPT] Publish it

> **Use when:** you want a public portfolio or to help others learn from your build.

- [ ] Write-ups per phase as blog posts (the user's `~/projects/syndicate` cross-posts Markdown to dev.to and LinkedIn; wire the repo once with `scripts/install-into.sh`, opt in per post with a `crosspost:` block, use absolute links)
- [ ] A 3-minute demo video: a bad PR blocked → a good PR flows to prod → an attack detected → a restore

---

## Prove it

- [ ] README read cold by someone else → they can explain the architecture and three trade-offs
- [ ] Every resume bullet traces to an evidence file
- [ ] You can give the 60-second story without notes, and survive "why?" three times on any part of it
