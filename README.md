# DevSecOps on EKS: project template

A step-by-step plan for taking an inherited application to a secure,
observable, production-grade platform on **AWS EKS**, written to be followed
by someone new without other resources.

## Start here

**[template/README.md](template/README.md)**: the phase map, dependency graph,
the decisions the template makes for you, and the time and cost of each phase.

| # | Phase |
|---|---|
| 0 | [Inherit & plan](template/00-inherit-and-plan.md) |
| 1 | [Workstation & dev environment](template/01-dev-environment.md) |
| 2 | [Containerise](template/02-containerise.md) |
| 3 | [Local Kubernetes bridge (KinD)](template/03-local-kubernetes.md) |
| 4 | [Cloud foundation (Terraform)](template/04-cloud-foundation.md) |
| 5 | [Identity & secrets](template/05-identity-and-secrets.md) |
| 6 | [Run it well](template/06-run-it-well.md) |
| 7 | [Security pipeline (CI)](template/07-pipeline.md) |
| 8 | [GitOps & enforcement](template/08-gitops-and-enforcement.md) |
| 9 | [Edge (Gateway API, DNS, TLS, WAF)](template/09-edge.md) |
| 10 | [Observe & operate](template/10-observe-and-operate.md) |
| 11 | [Present it](template/11-present-it.md) |
| 12 | [Multi-account (later)](template/12-multi-account.md) |
| — | [Common mistakes](template/COMMON-MISTAKES.md): read each phase's section before that phase |

## Also in this repo

- [DEVSECOPS-GUIDE.md](DEVSECOPS-GUIDE.md): the three-layer model (baseline, pipeline, enforcement) the template is built on
- [reference/](reference/README.md): reviews of the demo projects and courses the template was merged from, and what each got right or wrong
