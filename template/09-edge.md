# Phase 9: Edge (load balancer, DNS, TLS, WAF) with the Gateway API

**Goal:** users reach the app at a real domain over HTTPS only, through one
ALB that's protected by a WAF and rate limits, with routing written in the
**Gateway API**, and rolling deployments cause **zero** 502s at the load
balancer.

**Why:** reference 05 served a public domain over plain HTTP; 02 used NodePort
on public nodes; nobody except 06 tuned health checks. The user's Section 11
list covers the gotchas courses skip: readiness gates, security-group
chaining, WAF, and CloudFront's lock-down.

> **Read first:** [Kubernetes networking, explained simply](https://github.com/nazmur96/devops-learning-journal/blob/main/networking/kubernetes-networking-explained.md).
> This phase is that article's "Project 1: let AWS do it" setup: VPC CNI for
> pods (job 1), the **ALB** as the L7 front door (job 2), **Gateway API** for the
> routing rules (job 3), and the **AWS Load Balancer Controller** as the rule
> follower (job 4). The controller is **not** in the traffic path; it only
> configures the ALB. Traffic goes ALB → pod.

---

## Why Gateway API (ADR)

| | Ingress | **Gateway API** (template default) |
|---|---|---|
| Status | Frozen; no new features | The Kubernetes standard for routing going forward |
| Who owns what | One object mixes infra and app concerns | **Role split**: platform team owns `GatewayClass` + `Gateway` (the ALB, its TLS, its WAF); app teams own `HTTPRoute`s in their own namespace |
| Traffic splitting, header matching, redirects | Vendor-specific annotations | Built into the standard (`weight`, `matches`, `filters`) |
| Portability | Annotations tie you to one controller | Same `HTTPRoute` works with LBC on EKS, Envoy Gateway locally (P3-6), Cilium, Traefik |
| Ecosystem | — | ExternalDNS, cert-manager, Argo Rollouts and Kyverno all understand Gateway API objects |

**Maturity note:** the AWS Load Balancer Controller's Gateway API support is
**newer than its Ingress support**. Use a current controller version, enable
its ALB Gateway feature, and check its docs for the exact CRD field names
below; they're the part most likely to have changed since this was written.
If a feature you need (e.g. readiness gates or WAF on Gateway-managed target
groups) isn't supported in your version, record that in the ADR and use the
fallback noted in that step.

---

## P9-1 Install the Gateway API CRDs and enable the controller's Gateway support

**Type:** Task

- [ ] Gateway API **standard-channel** CRDs installed (pinned version) **before** the controller, via Argo CD (sync wave −1)
- [ ] AWS Load Balancer Controller upgraded to a version with Gateway API support, with its **ALB Gateway** feature enabled (Helm value / feature gate, per its docs), running with its Pod Identity role (P4-6)
- [ ] `kubectl get gatewayclass` shows the class `Accepted: True`

---

## P9-2 Platform side: one `Gateway` = one ALB

**Type:** Story · *Owned by the platform team, lives in a `gateway` namespace*

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata: { name: aws-alb }
spec:
  controllerName: gateway.k8s.aws/alb          # the LBC's ALB controller
---
# ALB-level settings live in a controller-specific config object, not annotations
apiVersion: gateway.k8s.aws/v1beta1
kind: LoadBalancerConfiguration
metadata: { name: public-alb, namespace: gateway }
spec:
  scheme: internet-facing
  listenerConfigurations:
    - protocolPort: HTTPS:443
      defaultCertificate: <acm-certificate-arn>          # TLS terminates at the ALB (ACM)
      sslPolicy: ELBSecurityPolicy-TLS13-1-2-2021-06
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: { name: public, namespace: gateway }
spec:
  gatewayClassName: aws-alb
  infrastructure:
    parametersRef: { group: gateway.k8s.aws, kind: LoadBalancerConfiguration, name: public-alb }
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes: { namespaces: { from: Same } }               # only the redirect route below
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.sockshop.example.com"
      allowedRoutes:
        namespaces:
          from: Selector
          selector: { matchLabels: { gateway-access: public } }   # app namespaces opt in
```

- [ ] One `Gateway` (one ALB) for the whole app; **no** `type: LoadBalancer` per service (reference 05 explains why: cost and sprawl)
- [ ] `allowedRoutes` limits which namespaces may attach routes (the role split, enforced)
- [ ] `kubectl get gateway -n gateway` shows `Programmed: True` and the ALB's DNS name
- [ ] The ALB disappears when the `Gateway` is deleted (check the console; leaked ALBs cost money)

---

## P9-3 App side: `HTTPRoute`s, health checks and the HTTP → HTTPS redirect

**Type:** Story · *Owned by the app team, lives in `sock-shop`*

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: sockshop, namespace: sock-shop }
spec:
  parentRefs: [{ name: public, namespace: gateway, sectionName: https }]
  hostnames: ["sockshop.example.com"]
  rules:
    - matches: [{ path: { type: PathPrefix, value: /catalogue } }]
      backendRefs: [{ name: catalogue, port: 80 }]
    - matches: [{ path: { type: PathPrefix, value: / } }]
      backendRefs: [{ name: front-end, port: 8079 }]
---
# Plain HTTP only redirects to HTTPS (standard Gateway API filter, no annotation)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: https-redirect, namespace: gateway }
spec:
  parentRefs: [{ name: public, sectionName: http }]
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect: { scheme: https, statusCode: 301 }
---
# Target-group settings per Service (controller-specific CRD)
apiVersion: gateway.k8s.aws/v1beta1
kind: TargetGroupConfiguration
metadata: { name: front-end, namespace: sock-shop }
spec:
  targetReference: { name: front-end }               # the Service
  defaultConfiguration:
    targetType: ip                                   # ALB → pod IP directly
    healthCheckConfig:
      healthCheckPath: /health                       # the readiness endpoint (reference 06)
      healthyThresholdCount: 2
      unhealthyThresholdCount: 2
    targetGroupAttributes:
      - { key: deregistration_delay.timeout_seconds, value: "30" }
```

- [ ] `sock-shop` namespace labelled `gateway-access: public`
- [ ] Routes show `Accepted: True` and `ResolvedRefs: True` (`kubectl describe httproute`)
- [ ] Health checks point at the **readiness** endpoint; target type `ip`
- [ ] The path map is the same one you used locally with Envoy Gateway (P3-6); only `parentRefs` and the AWS-specific config objects differ

> **Security groups with target type `ip`** (fixes the bug in reference 07's
> spec): the ALB sends traffic **straight to pod IPs on the container port**,
> not to NodePorts. Let the controller manage SGs (a frontend SG for the ALB, a
> shared backend SG allowed into the node/pod SG on the target ports only).
> Verify that pods accept app traffic **only from the ALB's backend SG**.
> Pod-to-pod isolation inside the cluster is NetworkPolicy's job (P8-7).

---

## P9-4 DNS and certificates

**Type:** Story

| Option | How | Pick when |
|---|---|---|
| **ACM** certificate on the ALB | DNS-validated in Route 53, referenced in `LoadBalancerConfiguration` | **Default** (references 04, 06) |
| cert-manager + Let's Encrypt (DNS-01) | Certificate as a Kubernetes Secret on a Gateway listener | When TLS terminates in-cluster (e.g. Envoy Gateway locally, P3-6). How: [P9-4b](#p9-4b-opt-lets-encrypt-certificates-with-cert-manager) |

- [ ] Hosted zone in Route 53; registrar nameservers updated (the classic failure: an ACM cert stuck in *pending validation* because the NS records were never changed, from the user's Section 11 notes)
- [ ] ACM certificate (and its DNS validation record) **in Terraform**
- [ ] **ExternalDNS** (EKS add-on, Pod Identity) with the Gateway API sources enabled (`gateway-httproute`), so `hostnames` in an `HTTPRoute` become Route 53 records; `--policy=upsert-only` and a TXT ownership registry so it never deletes records it doesn't own
- [ ] `curl -I http://…` → 301 to HTTPS; `https://` serves a publicly trusted chain; TLS 1.2+ only (`nmap --script ssl-enum-ciphers -p 443 …` or SSL Labs)

---

## P9-4b [OPT] Let's Encrypt certificates with cert-manager

> **Use when:** TLS ends **inside** the cluster instead of at the ALB:
> Envoy Gateway or another in-cluster gateway behind an **NLB**, an internal
> tool (Argo CD UI, Grafana) on a private gateway, your **Talos homelab**, or
> any cluster with no ACM. **Skip it** for the public app on the ALB: the ALB
> can only serve ACM certificates, so ACM (P9-4) stays the default there.

**What it does, in plain English:** cert-manager is a controller that gets
certificates for you and **renews them before they expire**. You declare *"I
want a certificate for `argocd.example.com`"*; cert-manager proves to Let's
Encrypt that you control that domain, receives a free, publicly trusted
certificate (valid 90 days), stores it as a Kubernetes `Secret`, and renews it
about 30 days before expiry. Your gateway reads the Secret. You never touch a
certificate file. You already used cert-manager in P3-6 with a self-signed CA;
this is the same tool pointed at a public CA.

**How Let's Encrypt checks you own the domain (the "challenge"):**

| Challenge | How it proves ownership | Works for | Doesn't work for |
|---|---|---|---|
| **HTTP-01** | Let's Encrypt fetches `http://<domain>/.well-known/acme-challenge/<token>` from the internet | Public hosts with port 80 open | Wildcards (`*.example.com`), private/internal hosts, anything behind a VPN |
| **DNS-01** (recommended) | cert-manager writes a TXT record `_acme-challenge.<domain>` in Route 53; Let's Encrypt looks it up | Everything, including **wildcards** and **internal-only** hosts | Needs API access to your DNS provider |

**Steps:**

- [ ] cert-manager installed (already done in P4-6) with Gateway API support on (`config.enableGatewayAPI: true` in the Helm values)
- [ ] **IAM for DNS-01 (Pod Identity, P5-4):** a role for cert-manager's ServiceAccount allowing `route53:GetChange` on `arn:aws:route53:::change/*`, `route53:ChangeResourceRecordSets` + `route53:ListResourceRecordSets` on **your one hosted zone only** (optionally limited to `_acme-challenge.*` TXT records with the `route53:ChangeResourceRecordSetsNormalizedRecordNames` condition), and `route53:ListHostedZonesByName`
- [ ] **Two issuers, staging first.** Let's Encrypt production has strict rate limits (e.g. 5 duplicate certificates per week); a broken config can lock you out for days. The staging issuer has generous limits but its certificates aren't trusted by browsers, which is fine for testing:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: { name: letsencrypt-staging }
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: platform-team@example.com          # expiry warnings go here
    privateKeySecretRef: { name: letsencrypt-staging-account }
    solvers:
      - dns01:
          route53:
            region: eu-central-1
            hostedZoneID: Z0123456789ABC       # pin the zone; don't let it guess
        selector: { dnsZones: ["example.com"] }
# letsencrypt-prod: identical, with server: https://acme-v02.api.letsencrypt.org/directory
```

- [ ] **Get a certificate**, either way:
  - **Automatic (Gateway API):** annotate the `Gateway` with `cert-manager.io/cluster-issuer: letsencrypt-staging`; each HTTPS listener's `hostname` + `certificateRefs` name becomes a certificate (same pattern as P3-6)
  - **Explicit:** a `Certificate` object (`dnsNames: ["*.internal.example.com"]`, `secretName: internal-wildcard-tls`, `issuerRef: letsencrypt-staging`) and reference that Secret from the listener
- [ ] Check it: `kubectl get certificate` → `READY True`; `kubectl describe certificate` shows the events; `cmctl status certificate <name>` explains failures
- [ ] Switch the issuer to `letsencrypt-prod` → the browser shows a trusted padlock
- [ ] **Drill:** `cmctl renew <name>` → a new certificate is issued and the gateway serves it with **no restart**
- [ ] **Alert** on certificates close to expiry (cert-manager exposes `certmanager_certificate_expiration_timestamp_seconds`; alert if < 14 days), because a silently failing renewal is how certificates expire in production

**Debugging (follow the chain of objects):** `Certificate` → `CertificateRequest`
→ `Order` → `Challenge`. Run `kubectl describe` on each in that order; the
first one with an error tells you where it broke. Usual causes: the IAM
policy names the wrong hosted zone; the registrar's nameservers aren't
pointing at Route 53 (same trap as ACM in P9-4); you hit the production rate
limit because you skipped the staging issuer.

**Pairs with ExternalDNS:** ExternalDNS creates the `A`/`CNAME` record for the
hostname, and cert-manager gets the certificate for it. Together, adding a
hostname to a `Gateway` or `HTTPRoute` gives you DNS **and** HTTPS with no manual step.

---

## P9-5 Zero-downtime rollouts at the ALB

**Type:** Story · *User's Section 11/12 lists*

Three settings work together:

| Setting | What it fixes |
|---|---|
| **Pod readiness gates**: namespace label `elbv2.k8s.aws/pod-readiness-gate-inject=enabled` | A new pod isn't "Ready" until the **ALB** reports it healthy, so a rollout can't remove old pods before the new ones get traffic. Needs real readiness probes (P3), or there's nothing to wait on |
| **`preStop` sleep** (≈ 10–15 s) | The old pod keeps serving while the ALB deregisters it |
| **Deregistration delay** ~30 s (P9-3 `TargetGroupConfiguration`) | Default 300 s makes every rollout slow. Must be ≤ `terminationGracePeriodSeconds` |

- [ ] All three configured; `terminationGracePeriodSeconds` > preStop + drain time
- [ ] Confirm readiness-gate injection works for **Gateway-managed** target groups in your controller version (`kubectl get pod -o yaml` shows a `readinessGates` entry). If it doesn't, rely on `preStop` + deregistration delay and note it in the ADR
- [ ] **Prove it:** run a `curl` loop through the ALB (the P3-8 loop) during `kubectl rollout restart`, before and after. Count the 502s; after must be zero

---

## P9-6 WAF and rate limiting

**Type:** Story · *~$5/month per web ACL + ~$1 per rule + per-request charges; delete it after the session if you're on a budget*

- [ ] AWS WAF web ACL **in Terraform**: managed rule groups (Core rule set, Known bad inputs, SQL injection, Amazon IP reputation list) + a rate-based rule (e.g. 2,000 requests / 5 min per IP) and a stricter one for `/login`
- [ ] Associated with the Gateway's ALB through the controller's WAF setting in `LoadBalancerConfiguration` (check the field name for your version). Fallback: an `aws_wafv2_web_acl_association` in Terraform, looking up the ALB by the controller's tags
- [ ] **Count** mode first; review sampled requests for false positives; then **Block**
- [ ] **Prove it:** a `' OR 1=1--` query is blocked (403); a burst from one IP gets rate-limited; normal CUJ traffic unaffected
- [ ] WAF logs to CloudWatch/S3 with retention set

---

## P9-7 [OPT] CloudFront in front of the ALB

> **Use when:** the app serves lots of static assets or has users far from your region. Skip for mostly dynamic apps; it adds a second certificate and an ALB lock-down to manage.

**Type:** Story

- [ ] Static assets from a **private** S3 bucket through Origin Access Control
- [ ] CloudFront's certificate lives in **us-east-1** (a CloudFront requirement)
- [ ] **Lock the ALB to CloudFront** (the half most guides skip): ALB SG allows only the CloudFront origin-facing managed prefix list, **and** the ALB requires a secret custom header that CloudFront adds. Otherwise attackers bypass CloudFront and the WAF by hitting the ALB directly
- [ ] Shield Standard is automatic. Shield Advanced, NLB and PrivateLink are out of scope

---

## Prove it

- [ ] `https://` only; modern TLS; HTTP redirects with 301
- [ ] An app team can add an `HTTPRoute` in its namespace; a namespace **without** the `gateway-access` label can't attach one (status shows `NotAllowedByListeners`)
- [ ] Zero 502s during rollouts
- [ ] SQL injection probe blocked by the WAF; rate limit trips under a burst
- [ ] Pods reachable from the ALB only (SG check); app traffic between services governed by NetworkPolicy

## Common mistakes

| Mistake | Seen in |
|---|---|
| Public domain over HTTP only | 05 |
| NodePort on public nodes instead of an ALB | 02 |
| One LoadBalancer Service per microservice | (05 explains why not) |
| NodePort SG rules with target type `ip` | 07 (spec bug) |
| Readiness gates without readiness probes | (user's list; probes first) |
| CloudFront in front, but the ALB still open to the internet | (user's list) |
| Gateway API CRDs not installed (or wrong channel) before the controller | — |
| Any namespace allowed to attach routes to the public Gateway | — (use `allowedRoutes`) |
