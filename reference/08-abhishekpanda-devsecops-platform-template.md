# 08: DevSecOps Platform Template (abhishekpanda0620)

| | |
|---|---|
| **Source** | https://github.com/abhishekpanda0620/devsecops-platform-template (last commit 2025-12-28) + a Medium series (part 1) |
| **App** | One Node.js `user-service` (Express, prom-client metrics) |
| **Stack** | Terraform (VPC, EKS, ECR, GitHub OIDC, Argo CD module, split `foundation/` + `addons/`) · Kustomize base/overlays · Argo CD app-of-apps · GitHub Actions (CI, CD, Terraform, **EOL check**) · Gitleaks, Semgrep, Trivy, Checkov, detect-secrets (pre-commit) · **kube-prometheus-stack + loki-stack (Promtail) + Alertmanager → Slack**; Tempo declared in an umbrella chart but not deployed · Falco (from RCA) |
| **Why it's here** | The user's example of the **self-hosted observability** option, to compare with reference 06's **AWS-managed** one |

## Observability as built

| Signal | Tool | Notes |
|---|---|---|
| Metrics | kube-prometheus-stack (chart **55.5.0**, Dec 2023) via Argo CD | `ServiceMonitor` for the app; 15 d retention on 20 Gi gp3; dashboards by Grafana.com ID (7249, 1860) + a custom RED-style app dashboard |
| Logs | `loki-stack` chart (2.10.1) with **Promtail** | ⚠️ `loki-stack` is no longer maintained and Promtail is deprecated in favour of **Grafana Alloy**; use the `grafana/loki` chart |
| Traces | Tempo listed as an umbrella-chart dependency | ❌ not deployed by the Argo CD apps; app not instrumented |
| Alerts | Alertmanager → Slack; custom rules (error rate > 5 %, p95 > 1 s, crash-looping, memory > 90 %, PV > 90 %) | ⚠️ Slack webhook URL **inline in Helm values in Git** (placeholder, but the pattern leaks real ones); threshold alerts, not SLO burn rate |
| Access | Grafana + Alertmanager behind an Ingress subpath | RCA documents the `routePrefix`/`externalUrl` fix |

## Other notable things

- ✅ **RCA documents** (`docs/RCA/*.md`), real post-mortems: Alertmanager subpath redirect loop; Argo CD self-heal reverting a manual patch; **PVCs pending because there was no EBS CSI driver for gp3**; Falco crashing on small nodes (`mmap perf-buffer`). Great teaching material, and the format the template's P10-8 asks for.
- ✅ **EOL-check workflow**: weekly job flags runtimes and components past end-of-life.
- ✅ Terraform split into `foundation` (VPC/EKS/ECR/OIDC) and `addons` (Argo CD), with add-ons (`vpc-cni`, `coredns`, `kube-proxy`, `ebs-csi`) and optional KMS secrets encryption in the EKS module.
- ✅ Base manifests with probes, `runAsNonRoot`, `readOnlyRootFilesystem`, HPA (min 2), NetworkPolicy, ServiceMonitor.
- ⚠️ PDB `minAvailable: 1` with HPA min 2 (works, thin margin).
- ⚠️ GitHub OIDC trust `repo:<org>/<repo>:*` (any branch/PR), same as reference 06.
- ⚠️ `npm run lint || true`; `aquasecurity/trivy-action@master` (unpinned) ×3; image Trivy scan includes MEDIUM, but only the fs scan of CRITICAL actually fails the build.
- ⚠️ RCA fix for Argo CD reverting a manual patch was **disabling self-heal** on the parent app. The template's answer: fix the value in Git (self-heal is a security control, P8-4).
- ✅ Image SBOM via BuildKit (`sbom: true`) + a sign job (Cosign).

## Adopt

1. **RCA / post-mortem files in the repo** for every real incident during the build.
2. **EOL-check workflow**: scheduled check that runtimes, Kubernetes, charts and base images are within support.
3. `foundation` / `addons` Terraform split (cluster first, then things that need the cluster).
4. A RED-style app dashboard shipped as code (ConfigMap with the Grafana sidecar label).

## Compromises (don't copy)

`loki-stack` + Promtail (deprecated); tracing declared but not built; webhook URLs in Git values; old chart pins; `|| true` on lint; actions on `@master`; OIDC trust `:*`; disabling self-heal to make a manual patch stick.
