# DevSecOps Project Template: from inheriting a repo to production

A step-by-step path for taking an application you have just inherited to a
secure, observable, production-grade platform on **AWS EKS**. Written so that
someone new can follow it **without needing other resources**: each step
says what to do, why, how to check it worked, and what usually goes wrong.

**Scope:** EKS only. No ECS, no Fargate-as-platform, no other clouds.
Alternatives appear only as one-line ADR notes, never as parallel tracks.

It is the merge of:

- the user's own **Sock Shop plan** (`reference/07`): its epic → ticket → acceptance-criteria format and its drills
- the **best part of each reference project** (`reference/01`–`06`)
- the **three-layer model** from [`../DEVSECOPS-GUIDE.md`](../DEVSECOPS-GUIDE.md)
- the user's **workstation rules** (`~/orca/projects/gitops-workstation`): mise or devenv, devcontainer, DevPod, Skaffold/Tilt + mirrord

---

## How to use this template

1. **Do the phases in order.** Each one assumes the previous one is done.
   The [dependency graph](#dependency-graph) shows which phases can run in parallel.
2. **Every phase has the same shape:**
   - **Goal**: one sentence
   - **Why**: what breaks if you skip it
   - **Steps**: tickets, each with acceptance criteria you can tick off
   - **Prove it**: the drill or negative test that shows it really works
   - **Common mistakes**: seen in the reference projects
   - **Cost and time**: so nothing surprises you
3. **Copy tickets into your tracker** (Jira, GitHub Projects, a Markdown file). Ticket IDs are `P<phase>-<n>` so you can rename them to your project key.
4. **Write an ADR** whenever a ticket says *Decision*. Template in [Phase 0](00-inherit-and-plan.md#p0-8-start-the-adr-log).
5. **Never skip "Prove it".** A control you haven't seen fail is a control you don't know works.

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
| 10 | [Observe & operate](10-observe-and-operate.md) | Metrics, logs, traces, SLOs, runtime detection, DR, chaos, cost | Runtime | 06, 07 |
| 11 | [Present it](11-present-it.md) | README, evidence, ADRs, resume bullets, interview story | — | 04, 05 |
| — | [Common mistakes](COMMON-MISTAKES.md) | Everything the reference projects got wrong, in one list | — | all |

---

## Dependency graph

```
P0 Plan
 └─→ P1 Dev environment
      └─→ P2 Containerise ─────────────┐
           ├─→ P3 Local Kubernetes     │
           │    └─────────────┐        │
           │                  ▼        ▼
           │   P4 Cloud foundation ←── (can start after P1, in parallel with P2/P3)
           │                  │
           │                  ▼
           │   P5 Identity & secrets
           │                  │
           ▼                  ▼
      P7 Pipeline ──→ P6 Run it well (on EKS; designed in P3)
           │                  │
           └────────┬─────────┘
                    ▼
           P8 GitOps & enforcement
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
       P9 Edge         P10 Observe & operate
          └─────────┬─────────┘
                    ▼
              P11 Present it
```

- **P4 does not depend on P3.** Terraform for the VPC/IAM/registry can be written while you learn Kubernetes locally.
- **P7 can start as soon as P2 exists.** Lint, scans and image builds don't need a cluster.
- **P6 is designed in P3** (probes, resources, shutdown are proven locally) and **re-verified on EKS**.

---

## Decisions this template makes for you

Change any of them with an ADR; the template still works.

| Decision | Choice | Why (and where the reference projects differ) |
|---|---|---|
| Cloud | AWS, one region, 3 AZs | All references use AWS/EKS |
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

Reference 06's author reports **$30–50 for the whole course** with
destroy-after-each-section discipline. Set an **AWS Budget alert** in
[P0](00-inherit-and-plan.md#p0-7-set-the-budget-and-cost-guardrails) before you create anything.

---

## Layers (from the guide)

| Layer | Question | Phases |
|---|---|---|
| **0 Plan** | What are we building, for whom, at what risk and cost? | P0 |
| **1 Baseline** | Is it built securely and reliably? | P1–P6, P9 |
| **2 Pipeline** | Did this change introduce a problem? | P7 |
| **3 Enforcement** | Can anything get around layers 1 and 2? | P8 |
| **Runtime** | What is happening right now, and can we recover? | P10 |
