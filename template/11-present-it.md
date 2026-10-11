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
