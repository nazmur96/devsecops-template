# DevSecOps Project Template: from inheriting a repo to production

> 🗺️ **New here?** The [visual overview](../README.md) explains the whole plan with diagrams in about 5 minutes. This page is the detailed hub.

A step-by-step path for taking an application you have just inherited to a
secure, observable, production-grade platform on **AWS EKS**. Written so that
someone new can follow it **without needing other resources**: each step
says what to do, why, how to check it worked, and what usually goes wrong.

**Scope:** EKS only. No ECS, no Fargate-as-platform, no other clouds.
Alternatives are never parallel tracks; [P0-13](00-inherit-and-plan.md#p0-13-why-eks-the-compute-decision) teaches when they would be the better choice.

It is the merge of:

- the user's own **Sock Shop plan** (`reference/07`): its epic → ticket → acceptance-criteria format and its drills
- the **best part of each reference project** (`reference/01`–`09`)
- the **three-layer model** from [`../DEVSECOPS-GUIDE.md`](../DEVSECOPS-GUIDE.md)
- the user's **workstation rules** (`~/orca/projects/gitops-workstation`): mise or devenv, devcontainer, DevPod, Skaffold/Tilt + mirrord

---

## How to use this template

1. **Do the phases in order, 0 → 11.** Each one assumes the previous one is done.
   The [dependency graph](#dependency-graph) shows which phases can run in parallel.
   Phase 12 is for later, after a few projects.
2. **Read that phase's section of [Common mistakes](COMMON-MISTAKES.md) before you start it.** Five minutes there saves an afternoon of debugging.
3. **⏸ means "come back later".** A few tickets need something from a later phase. They're marked ⏸ and listed under [Come back later](#come-back-later). Read them when you meet them, build them when that list says.
4. **Every phase has the same shape:**
   - **Goal**: one sentence
   - **Why**: what breaks if you skip it
   - **Steps**: tickets, each with acceptance criteria you can tick off
   - **Prove it**: the drill or negative test that shows it really works
   - **Common mistakes**: seen in the reference projects
   - **Cost and time**: so nothing surprises you
5. **Copy tickets into your tracker** (Jira, GitHub Projects, a Markdown file). Ticket IDs are `P<phase>-<n>` so you can rename them to your project key.
6. **Write an ADR** whenever a ticket says *Decision*. Template in [Phase 0](00-inherit-and-plan.md#p0-8-start-the-adr-log).
7. **Never skip "Prove it".** A control you haven't seen fail is a control you don't know works.

### The worked example

The template is generic, but every phase has a **worked example** using
**Sock Shop** (Weaveworks: 8 services in Go/Java/Node, MongoDB, MySQL,
RabbitMQ), because it's polyglot, stateful, and has realistic rough edges.
Swap in your own app; the steps don't change.

---

## Phase map

| # | Phase | Goal | Layer | Best reference |
|---|---|---|---|---|
| 0 | [Inherit & plan](00-inherit-and-plan.md) | Know the app, the people, the risks and the budget before touching infra | L0 | 03, 07 |
| 1 | [Workstation & dev environment](01-dev-environment.md) | A reproducible per-project toolchain; run and understand the app locally | L0/L1 | workstation notes, 04 |
| 2 | [Containerise](02-containerise.md) | Hardened, small, reproducible images and a local Compose stack | L1 | 06, 07 |
| 3 | [Local Kubernetes bridge](03-local-kubernetes.md) | Learn and prove Kubernetes primitives with no cloud bill | L1 | 07 |
| 4 | [Cloud foundation (IaC)](04-cloud-foundation.md) | State, network, EKS, access, add-ons, registry, all in Terraform | L1 | 05, 06 |
| 5 | [Identity & secrets](05-identity-and-secrets.md) | No long-lived credentials anywhere: humans, CI, or pods | L1 | 04, 06 |
| 6 | [Run it well](06-run-it-well.md) | Probes, resources, scaling, disruption budgets, spread, graceful shutdown | L1 | 06, 07 |
| 7 | [Security pipeline (CI)](07-pipeline.md) | Every change is scanned, built once, signed, and fails closed | L2 | 02, 04 |
| 8 | [GitOps & enforcement](08-gitops-and-enforcement.md) | Git is the only way in; admission rejects anything that skipped CI | L3 | 06, 07 |
| 9 | [Edge](09-edge.md) | HTTPS, DNS, WAF, zero-downtime rollouts at the load balancer | L1 | 06, user Section 11 |
| 10 | [Observe & operate](10-observe-and-operate.md) | Metrics, logs, traces, SLOs, runtime and account security, auto-remediation, DR, chaos, cost, upgrades | Runtime | 06, 07 |
| 11 | [Present it](11-present-it.md) | README, evidence, ADRs, resume bullets, interview story | — | 04, 05 |
| 12 | [Multi-account (later)](12-multi-account.md) | One AWS account per environment: Organizations, SCPs, cross-account doors. **After a few projects** | L1 | 09, user notes |
| — | [Common mistakes](COMMON-MISTAKES.md) | Everything the reference projects got wrong, grouped by phase. **Read each phase's section before starting that phase**, not at the end | — | all |
| — | [Day-2 operations](DAY-2-OPERATIONS.md) | Running it after it's built: operating rhythm, troubleshooting path, 18 common incidents, drill calendar | Runtime | roadmap appendix |
| — | [When it all runs together](WHEN-IT-ALL-RUNS-TOGETHER.md) | The system view: where policy, identity, routing and scaling decisions overlap, and how to decide if a tool is worth adding | — | roadmap Part VIII |

---

## Dependency graph

```
P0 Plan
 └─→ P1 Dev environment
      └─→ P2 Containerise
           └─→ P3 Local Kubernetes          (all local, $0)
                └─→ P4 Cloud foundation     (Terraform can be written during P2/P3)
                     └─→ P5 Identity & secrets
                          └─→ P6 Run it well   (first deploy by hand, P6-0)
                               └─→ P7 Pipeline (lint/scan jobs can start during P2)
                                    └─→ P8 GitOps & enforcement (replaces the P6-0 hand deploy)
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
       P9 Edge         P10 Observe & operate
          └─────────┬─────────┘
                    ▼
              P11 Present it
                    │   (after a few projects)
                    ▼
              P12 Multi-account
```

The numbers are the safe order. If you want to overlap phases:

- **P4 does not depend on P3.** Terraform for the VPC/IAM/registry can be written while you learn Kubernetes locally (don't `apply` until you need it; the meter starts then).
- **P7's PR checks can start as soon as P2 exists.** Lint, scans and image builds don't need a cluster. The release workflow needs ECR (P4-7) and GitHub OIDC (P5-2).
- **P6 is designed in P3** (probes, resources, shutdown are proven locally) and **re-verified on EKS**. The app gets onto EKS by hand in [P6-0](06-run-it-well.md#p6-0-first-deploy-to-eks-temporary-by-hand); P8 replaces that with GitOps.

### Come back later

These tickets sit in an earlier phase because that's where they belong
logically, but they need something built later. Each is marked ⏸ in its phase.

| Ticket | Read it in | Build it after | Why it has to wait |
|---|---|---|---|
| [P7-5](07-pipeline.md#p7-5-promotion-by-digest-never-by-tag) final "Prove it" | P7 | P8, P10-6 | Needs staging running and the ZAP scan |
| [P8-4b](08-gitops-and-enforcement.md#p8-4b-post-deploy-smoke-test) against the public URL | P8 | P9-4 | Until then, run it against the in-cluster Service |
| [P8-4c](08-gitops-and-enforcement.md#p8-4c-integration-and-api-tests-in-staging) ZAP step | P8 | P10-6 | ZAP is set up in P10 |
| [P8-8](08-gitops-and-enforcement.md#p8-8-progressive-delivery-with-argo-rollouts) Argo Rollouts canary | P8 | P9 **and** P10-1 | Splits traffic at the ALB (P9) and decides from AMP metrics (P10-1) |
| [P6-2](06-run-it-well.md#p6-2-replicas-and-autoscaling) HPA against the SLO | P6 | P10-7 | P6 proves scaling with a simple load loop; P10-7 proves it against the p95 SLO with k6 |
| [P6-5b](06-run-it-well.md#p6-5b-platform-node-pool-and-priorityclass) platform node pool | P6 | P6-6 | Apps need Karpenter's NodePools before the managed node group can be tainted for platform tools |
| [P4-9](04-cloud-foundation.md#p4-9-managed-data-stores-where-the-app-allows) [OPT] RDS IAM auth | P4 | P5-4 | The pod needs its Pod Identity role before it can request a database token |

When you finish P10, come back to this table and close every row.

---

## Scale the template to your situation

As a learning project, do everything (skipping `[OPT]` freely). In a real
job, **knowing what to leave out is the skill.** What decides it isn't
company size but four separate things:

1. **Blast radius:** does a bad deploy lose money, data or trust, or is it an annoying rollback?
2. **Compliance:** is there a real external gate (SOC 2, PCI, HIPAA auditor), or just good instinct?
3. **Team vs system size:** can one platform engineer hold the whole system in their head?
4. **Traffic and cost:** does inefficiency cost real money?

High compliance with a tiny team (a 6-person fintech with one SOC 2 client)
is the hardest combination; there, managed services matter most.

| | **Tier 1: small** (< ~15 services, small team, no compliance mandate) | **Tier 2: growing** (~15–60 services, 2–5 platform engineers) | **Tier 3: enterprise / regulated** |
|---|---|---|---|
| First question | Is EKS even right? ([P0-13](00-inherit-and-plan.md#p0-13-why-eks-the-compute-decision): ECS or Lambda may be) | EKS usually starts paying for itself here | Compliance decides the topology |
| **Keep (cheap, high payoff)** | P0, P4, P5-1..5, P6-1..4, P7, P8-1..8, P9-1..6, P10-1..5, P10-9, the endpoint rule in P4-3 | All of Tier 1, plus P6-5b, P8-9, P10-5b, P10-8c, P10-10c, P10-10d | All of Tier 2, plus P12 (account per environment) and P8-11 (mTLS) |
| **Minimal version** | One cluster with namespaces; CloudWatch only; 2–3 SLIs, no full burn-rate framework | Two clusters (P0-6 default); Kyverno in Audit for new rules | The regulated service in **its own cluster and account**, however small |
| **Defer until it hurts** | P5-6 OpenBao, P8-10 vCluster, P8-11 mesh, P8-12 flags, P10-11 multi-region, P10-13 self-hosted stack, P3-7b Cilium | A second mesh next to Cilium; self-hosted logging | Active-active multi-region (start with warm standby) |
| **Consolidate instead of adding** | — | Secrets Manager + RDS IAM auth before OpenBao; Cilium's own L7 before adding a mesh; managed logs, spending the effort on log *quality* | Buy down every non-differentiating system (managed Prometheus, logs, Kafka, Vault) and invest in "who wins" docs ([WHEN-IT-ALL-RUNS-TOGETHER.md](WHEN-IT-ALL-RUNS-TOGETHER.md)) |
| Beyond this template | — | — | Internal developer platform (Backstage), SPIFFE/SPIRE, compliance evidence automation |

**The rule in one line:** adopt tools that **remove a manual, error-prone
step** (certificates, scoped IAM, consistent deploys); defer tools that **add
a new decision-making system** (a mesh, dynamic secrets, an event bus) until a
specific incident or requirement justifies them. "We might need it later"
isn't a reason; "this hurt us last quarter" is.

**Where there's no cheap middle ground:** real cluster separation has a fixed
operational floor (two control planes to patch and upgrade); active-active
multi-region is roughly double the infrastructure plus genuine
distributed-systems work; audit evidence always needs a human to maintain the
story. If you need these, budget honestly; don't fake them with namespaces.

---

## Decisions this template makes for you

Change any of them with an ADR; the template still works.

| Decision | Choice | Why (and where the reference projects differ) |
|---|---|---|
| Cloud | AWS, one region, 3 AZs | All references use AWS/EKS |
| Compute | **EKS**, for a learning reason stated in an ADR | [P0-13](00-inherit-and-plan.md#p0-13-why-eks-the-compute-decision) teaches when ECS/Fargate or Lambda would be the better architecture |
| IaC | **Terraform**, remote state in S3 with locking, never committed | 05/06 got state right; 06 committed a state backup with passwords |
| Cluster | **EKS** (current default version), managed node group + Karpenter later | 02/04 used `eksctl` CLI flags (not reproducible) |
| Local cluster | **KinD** (3 control plane + 3 workers) | Only 07 has a local bridge; it saves money and teaches more |
| Cluster access | **EKS Access Entries**, `authentication_mode = "API"` | 02 used legacy `aws-auth` |
| Pod → AWS | **EKS Pod Identity** (IRSA taught once, for contrast) | 06 does this everywhere; 05 only for one controller |
| CI → AWS | **GitHub OIDC**, trust pinned to repo + branch/environment | 04 explains why (4-scenario lesson); 06 pinned too loosely (`repo:*`) |
| CI | **GitHub Actions** | 02 used Jenkins; most new teams use Actions |
| CD | **Argo CD** (app-of-apps); CI never runs `kubectl apply` | 02/04 deployed from CI with `kubectl` |
| Packaging | Raw manifests first (learn), **Helm** after (reuse) | 07's order: you can only judge a template once you've written the YAML |
| Secrets | **AWS Secrets Manager + External Secrets Operator** (default); **OpenBao** as the advanced track | 04/06 used Secrets Manager; 07 planned OpenBao |
| Admission | **Kyverno** + Pod Security Admission | No reference implemented this |
| Signing | **Cosign keyless** from CI, verified by Kyverno | No reference implemented this |
| Observability | **Instrument with OpenTelemetry** (backend becomes a collector setting). Learn the open-source stack **locally for free** (Prometheus, Loki, Tempo, Grafana on KinD, P3-10); run **AWS-managed** on EKS (ADOT → AMP, CloudWatch Logs, X-Ray, Amazon Managed Grafana; P10). Production-grade self-hosted on EKS as a later track (P10-13) | [P10-0](10-observe-and-operate.md#p10-0-choose-the-observability-path) compares all three (06 managed; 04 and 08 self-hosted) |
| Artifact repository | Images: **ECR** (+ ECR **pull-through cache** for Docker Hub/GHCR/Quay). Language packages: **AWS CodeArtifact** as a proxy cache. **Nexus** as the self-hosted, enterprise-style option | [P4-8](04-cloud-foundation.md#p4-8-artifact-repository-dependency-proxy-ecr-pull-through-codeartifact-or-nexus) |
| Routing API | **Gateway API**: Envoy Gateway locally, the AWS Load Balancer Controller (ALB) on EKS; the same `HTTPRoute`s in both. Ingress only as a fallback if a controller feature you need is missing (ADR) | P3-6, P9 |
| Runtime | **Falco** | No reference implemented this |
| Dev env | `mise.toml` **or** `devenv.nix` (never both) + `devcontainer.json` + **DevPod** | User's workstation rules |
| Inner loop | **Skaffold or Tilt** (pick one) + **mirrord** | User's workstation rules; added once workloads exist |

---

## Explainers (read when a phase points to them)

Plain-language articles the user wrote; each one is linked from the phase that needs it.

| Article | Read before |
|---|---|
| [How code goes from your laptop to the user](https://github.com/nazmur96/devops-learning-journal/blob/main/foundations/how-code-goes-from-laptop-to-user.md): entry point, dependencies, build, output per language; why the output decides the Docker image | P1-6, P2 |
| [Kubernetes networking, explained simply](https://github.com/nazmur96/devops-learning-journal/blob/main/networking/kubernetes-networking-explained.md): the 5 networking jobs, L4 vs L7, Ingress vs Gateway API, the ingress-nginx retirement | P3-6, P9 |
| [Why can't my app connect to my database?](https://github.com/nazmur96/devops-learning-journal/blob/main/networking/why-cant-my-app-connect-to-my-database.md): security groups, SG-as-source, the 4 gates and their error messages | P4-3, P4-9 |
| [Getting AWS secrets into EKS pods with ESO and Pod Identity](https://github.com/nazmur96/devops-sre-authentication/blob/main/posts/eso-eks-pod-identity.md): the courier analogy, every command, verification, troubleshooting | P5-4, P5-5 |

---

## Time and cost (rough)

| Phase | Time (part-time learner) | AWS cost |
|---|---|---|
| P0–P3 | 3–5 weeks | **$0** (all local) |
| P4–P6 | 2–3 weeks | ~$3–5 per 4-hour session (EKS control plane ~$0.10/h + nodes + NAT if used); **destroy after each session** |
| P7 | 1 week | $0 (GitHub Actions free tier) + ECR storage pennies |
| P8–P10 | 3–4 weeks | ~$5–10 per session with the observability stack |
| P11 | 3–5 days | $0 |
| P12 (later) | 1–2 weeks | Organizations and SCPs are free; the cost is **one cluster per environment account** instead of one shared cluster, so roughly the P4–P6 session cost per account you run at the same time |

Reference 06's author reports **$30–50 for the whole course** with
destroy-after-each-section discipline. Set an **AWS Budget alert** in
[P0](00-inherit-and-plan.md#p0-7-set-the-budget-and-cost-guardrails) before you create anything.

---

## Layers (from the guide)

| Layer | Question | Phases |
|---|---|---|
| **0 Plan** | What are we building, for whom, at what risk and cost? | P0 |
| **1 Baseline** | Is it built securely and reliably? | P1–P6, P9; P12 (later) |
| **2 Pipeline** | Did this change introduce a problem? | P7 |
| **3 Enforcement** | Can anything get around layers 1 and 2? | P8 |
| **Runtime** | What is happening right now, and can we recover? | P10 |
| — | Can someone else understand it and trust it? | P11 |
