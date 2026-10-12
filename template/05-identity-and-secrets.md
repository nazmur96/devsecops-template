# Phase 5: Identity & secrets

**Goal:** no long-lived credential exists anywhere: not for humans, not for
CI, not for pods. Every identity can do only its own job, and every secret
comes from a secrets manager at runtime.

**Why:** stolen or over-broad credentials cause most cloud breaches. The
reference projects used admin IAM users with static keys (04, 05), long-lived
Docker Hub tokens (04, 05), Jenkins-stored credentials (02) and plain
Kubernetes Secrets (01). Reference 04's part 29 is the best explanation of
why each is dangerous; this phase builds the safe version.

Three kinds of identity, three mechanisms:

| Who | Mechanism | Never |
|---|---|---|
| **Humans** → AWS / cluster | SSO (IAM Identity Center) roles + EKS Access Entries | IAM users with access keys |
| **CI** → AWS | GitHub OIDC → `AssumeRoleWithWebIdentity` | Access keys or kubeconfig in GitHub secrets |
| **Pods** → AWS | EKS Pod Identity (IRSA as the older alternative) | The node's instance role; keys in env vars |

---

## P5-1 Human access through SSO roles

**Type:** Story

- [ ] IAM Identity Center (or your IdP) with permission sets: `PlatformAdmin`, `Developer` (read-only AWS)
- [ ] CLI login with `aws sso login` / `aws login`; sessions expire (1–12 h)
- [ ] EKS Access Entries per role (P4-4 pattern):

| Role | Access policy | Scope |
|---|---|---|
| PlatformAdmin | `AmazonEKSClusterAdminPolicy` | cluster |
| Developer (team X) | `AmazonEKSEditPolicy` | namespace `team-x` only |
| Auditor | `AmazonEKSViewPolicy` | cluster |

- [ ] **Negative test:** the Developer role can't create a Deployment in another team's namespace (`kubectl auth can-i create deploy -n team-y` → `no`)
- [ ] **Namespace = team = RBAC boundary** (reference 03)
- [ ] **No bastion, no SSH keys.** Offer this *before* someone asks for "a bastion and an access key": **SSM Session Manager** gives a shell on a node or private instance through the AWS CLI (`aws ssm start-session`), with no inbound ports, nothing to patch, and every session logged. For pods, use `kubectl exec` / `kubectl debug` through the Access Entry. Locally, developers use their own SSO session, so the same SDK code runs locally and in the pod with no credentials in config

> **The conversation you'll have:** *"Can you just make me an IAM user and
> send me the access key?"* Don't lead with "that's insecure". Lead with the
> alternative and why it's **less** work for them: the SDK finds credentials by
> itself (SSO locally, Pod Identity in the cluster), so they **delete**
> credential-handling code instead of adding it. Then answer the real
> objections before they come: "how do I test locally?" (`aws sso login`) and
> "how do I debug in prod?" (Session Manager, `kubectl debug`).

---

## P5-2 CI access through GitHub OIDC

**Type:** Story · *Reference 04 parts 29–30, with the trust policy tightened*

**Why OIDC, in four steps** (reference 04's lesson, worth retelling in interviews):

1. Admin kubeconfig as a GitHub secret: static, cluster-admin, every namespace. Worst.
2. A service-account token scoped to one namespace: smaller blast radius, but long-lived and never rotated.
3. A short-lived token minted at runtime: but minting it needs **AWS access keys** in GitHub, which exposes the whole account.
4. **OIDC:** GitHub signs a short-lived token saying *which repo, branch and workflow* is running; AWS verifies the signature and issues 1-hour credentials for one role. **Nothing secret is stored.** (The token is *signed*, not encrypted: anyone can read its claims, nobody can forge them.)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<account>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:<org>/sockshop:ref:refs/heads/main"
      }
    }
  }]
}
```

| Role | Trusted `sub` | Permissions |
|---|---|---|
| `sockshop-ci-build` | `repo:<org>/sockshop:ref:refs/heads/main` | ECR push to `sockshop/*` only |
| `sockshop-ci-plan` | `repo:<org>/sockshop-platform:pull_request` | read-only + state read (for `terraform plan` on PRs) |
| `sockshop-ci-apply` | `repo:<org>/sockshop-platform:environment:prod` | Terraform apply; requires the GitHub `prod` environment's reviewer |

- [ ] OIDC provider and roles **in Terraform**, not CLI commands (reference 04 did it imperatively)
- [ ] `sub` pinned to a branch or environment, **never `repo:<org>/<repo>:*`** (reference 06's trust allowed any branch or PR to assume the role)
- [ ] **Negative tests:** a workflow on another branch fails to assume the role; the build role can't call `ec2:*` or `iam:*`
- [ ] Workflows declare `permissions: { id-token: write, contents: read }`; nothing broader by default
- [ ] **Single account = IAM is your only wall between environments** (P0-6, Phase 12). So every CI and workload policy names its resources by environment (`arn:…:sockshop/dev/*`, `sockshop-dev-*`) or uses tag conditions (`aws:ResourceTag/Environment = dev`). **No `"Resource": "*"`** on anything that can write. **Negative test:** the dev CI role can't read a prod secret, push to a prod-only repository, or describe the prod cluster
- [ ] **No deploy role for app CI at all.** With GitOps (P8), CI only pushes images and commits to the GitOps repo; Argo CD deploys from inside the cluster. If you must push-deploy (references 02/04), scope it like reference 04: Access Entry + `AmazonEKSEditPolicy` limited to one namespace, one role per environment

---

## P5-3 GitHub Environments

**Type:** Story · *Reference 07, SOCKPLAT-16*

- [ ] Environments `dev`, `staging`, `production`; environment-scoped variables/secrets
- [ ] `production`: required reviewer + wait timer; deployment branch rule `main` only
- [ ] Non-secret config via `vars`, not `secrets`
- [ ] **Negative test:** a `dev` job can't read a `production` secret

---

## P5-4 Pod access through EKS Pod Identity

**Type:** Story · *Reference 06 is the model: one role per service, only what that service needs*

```hcl
resource "aws_iam_role" "carts" {
  name = "sockshop-${var.env}-carts"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "pods.eks.amazonaws.com" }
      Action    = ["sts:AssumeRole", "sts:TagSession"]
    }]
  })
}
resource "aws_eks_pod_identity_association" "carts" {
  cluster_name    = var.cluster_name
  namespace       = "sock-shop"
  service_account = "carts"
  role_arn        = aws_iam_role.carts.arn
}
```

- [ ] One role per service that calls AWS; policy limited to its own resources (one table, one queue, one secret path)
- [ ] **Prove it:** `kubectl exec` into the pod → `aws sts get-caller-identity` shows the service's role
- [ ] **Negative test:** a pod with a different ServiceAccount gets `AccessDenied` on that resource
- [ ] **Negative test:** with IMDS hop limit 1 (P4-5), a pod **cannot** get the node's credentials from `169.254.169.254`
- [ ] **[OPT] IRSA once, for contrast:** set up one role via the cluster OIDC provider + annotated ServiceAccount; note the differences (per-cluster trust policy edits vs a cluster-independent role). Don't confuse the **GitHub** OIDC provider (CI → AWS) with the **cluster** OIDC provider (IRSA, pods → AWS) (reference 04 explains this well)

---

## P5-4b ServiceAccounts: one per workload, sometimes several per service

**Type:** Story

**What a ServiceAccount is.** A ServiceAccount (SA) is a pod's **identity**
inside Kubernetes, the way an IAM role is an identity in AWS. Everything that
asks "who is this pod?" looks at its SA:

| Who asks | What they decide from the SA |
|---|---|
| **EKS Pod Identity** (P5-4) | Which **AWS role** the pod gets (associations are per namespace + SA) |
| **Kubernetes RBAC** | What the pod may do with the **Kubernetes API** (most app pods: nothing) |
| **External Secrets / OpenBao** (P5-5, P5-6) | Which **secrets** it may read |
| **Kyverno** (P8-6) | Policy decisions, e.g. "only `carts-migrate` may run the migration image" |
| **Logs, CloudTrail, Falco** | *Who* did something, in an investigation |

Every namespace has a `default` SA. If a pod doesn't name one, it gets
`default`, and so does every other pod that forgot. Then all of them share one
identity: give `default` an AWS role for one service and **every** pod in the
namespace gets it.

**Rule 1: one ServiceAccount per microservice.** `carts` runs as SA `carts`,
`orders` as `orders`. Never `default`.

**Rule 2: one ServiceAccount per *job* a service does, when the jobs need
different permissions.** A single service often runs more than one kind of
workload:

| Workload of the `orders` service | ServiceAccount | AWS role (Pod Identity) | DB credentials | K8s API |
|---|---|---|---|---|
| The API (Deployment) | `orders` | read/write its SQS queue | app user: SELECT/INSERT/UPDATE | none |
| Schema migrations (Argo CD PreSync Job, P8-3b) | `orders-migrate` | none | **migration user: DDL** (CREATE/ALTER) | none |
| Nightly export (CronJob) | `orders-export` | write to **one** S3 prefix | read-only user | none |
| Queue worker (separate Deployment) | `orders-worker` | read/delete from the queue only | app user | none |

Why split: the API pod is the one exposed to user input, so it's the most
likely to be compromised. It should **not** hold the DDL password or the S3
write permission it never uses. Split SAs mean a breach of one workload only
gets that workload's permissions.

When **not** to split: if two workloads need exactly the same permissions,
one SA is fine. Containers **in the same pod** (sidecars) always share the
pod's SA, so put a component with different needs in its own pod.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders            # the API
  namespace: sock-shop
automountServiceAccountToken: false     # this pod never calls the Kubernetes API
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-migrate    # the migration Job only
  namespace: sock-shop
automountServiceAccountToken: false
---
# In the Deployment:      spec.template.spec.serviceAccountName: orders
# In the migration Job:   spec.template.spec.serviceAccountName: orders-migrate
```

```hcl
# Each SA gets its own Pod Identity association and its own least-privilege role
resource "aws_eks_pod_identity_association" "orders" {
  cluster_name = var.cluster_name
  namespace = "sock-shop"
  service_account = "orders"
  role_arn = aws_iam_role.orders.arn             # SQS only
}
resource "aws_eks_pod_identity_association" "orders_export" {
  cluster_name = var.cluster_name
  namespace = "sock-shop"
  service_account = "orders-export"
  role_arn = aws_iam_role.orders_export.arn      # one S3 prefix only
}
```

- [ ] Every Deployment, StatefulSet, Job and CronJob names its own SA; **no workload uses `default`** (Kyverno policy `disallow-default-serviceaccount`, P8-6)
- [ ] `automountServiceAccountToken: false` on every SA whose pods don't call the Kubernetes API (almost all app pods). The few that do (operators, Argo CD, controllers) get a namespaced `Role` with only the verbs they need
- [ ] Separate DB users per SA where the permissions differ (app vs migrations vs read-only), each with its own secret in Secrets Manager (`sockshop/dev/orders/db-app`, `…/db-migrate`)
- [ ] A table like the one above in `docs/inventory.md` for every service
- [ ] **Negative tests:** the `orders` pod can't write to the export S3 prefix; the `orders-export` pod can't touch SQS; `kubectl auth can-i --as=system:serviceaccount:sock-shop:orders list secrets -n sock-shop` → `no`

---

## P5-5 Application secrets: default path (Secrets Manager + External Secrets)

**Type:** Story

> **Read first:** [Getting AWS secrets into EKS pods with ESO and Pod Identity](https://github.com/nazmur96/devops-sre-authentication/blob/main/posts/eso-eks-pod-identity.md).
> It walks through this ticket step by step (IAM role, association,
> `ClusterSecretStore`, `ExternalSecret`, verification, troubleshooting,
> cleanup), with the courier analogy: Secrets Manager is the vault, ESO the
> courier, the Pod Identity Agent the ID-card office, the IAM role the
> permission slip. Here it's done in Terraform + Argo CD instead of CLI commands.
>
> Gotchas from the article:
> - **Restart ESO after creating its Pod Identity association.** Credentials are wired in when a pod starts; old pods get `AccessDenied`
> - If the secret uses a **customer-managed KMS key**, the role also needs `kms:Decrypt` on it
> - **Env vars are read once.** A rotated secret reaches the app only after a restart: mount it as a file, or use **Reloader** to restart on change

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata: { name: aws-sm }
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-central-1          # credentials: ESO controller's Pod Identity role
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: { name: catalogue-db, namespace: sock-shop }
spec:
  refreshInterval: 1h
  secretStoreRef: { kind: ClusterSecretStore, name: aws-sm }
  target: { name: catalogue-db }
  data:
    - secretKey: password
      remoteRef: { key: sockshop/dev/mysql, property: password }
```

- [ ] Secrets created in Secrets Manager under `sockshop/<env>/<name>` (by Terraform with generated values, or by hand once; **never** in Git)
- [ ] ESO's IAM role can read **only** `sockshop/<env>/*`
- [ ] Every Kubernetes Secret in the app namespace is generated by an ExternalSecret; nothing hand-made
- [ ] **Prove it:** rotate a value in Secrets Manager → the Kubernetes Secret updates within `refreshInterval`; the app picks it up (restart via Reloader, or the app re-reads the file)
- [ ] Repo scan (Gitleaks over full history) confirms no credential was ever committed

> **Trade-off to know:** with one `ClusterSecretStore`, the ESO controller's
> identity can read every app secret it's allowed to. For **per-service**
> isolation use the **Secrets Store CSI driver + AWS provider** (reference 06:
> each pod mounts its secret with its own Pod Identity role), or OpenBao's
> per-role policies (P5-6).

---

## P5-6 [ADVANCED] OpenBao track

> **Use when:** you need **dynamic** database credentials, per-service secret policies stricter than ESO gives you, or the same secrets system on-prem and in several clouds. Skip if Secrets Manager + ESO meets your needs (it does for most teams).

**Type:** Story · *Reference 07's plan, SOCKPLAT-67–70*

For when you want Vault-style features: dynamic database credentials,
per-service policies, no cloud lock-in.

- [ ] OpenBao HA (3 replicas, Raft), KMS auto-unseal via Pod Identity, TLS from cert-manager
- [ ] **Drill:** delete all OpenBao pods → cluster comes back **unsealed with no human action**
- [ ] Kubernetes auth: one role per service bound to one ServiceAccount + namespace; token TTL 1 h
- [ ] KV v2 paths per environment; one policy per service with read on **only** its path
- [ ] **Negative test:** `carts`'s role can't read the MySQL path
- [ ] **[OPT] Dynamic DB credentials:** database secrets engine issues a short-lived MySQL user per pod
- [ ] Failure mode documented: what happens to running pods if OpenBao is down?

> Most guides assume HashiCorp Vault; OpenBao (the Linux Foundation fork) is
> compatible but expect to translate `vault` → `bao` in CLI examples.

---

## P5-7 [OPT] SOPS for GitOps-native secrets

> **Use when:** there's no cloud secrets manager (homelab/Talos), the team is small, and secrets rarely change. Not for EKS production, where Secrets Manager is the default.

For homelab/Talos setups without a cloud secrets manager: secrets live
**encrypted** in Git with SOPS + age, decrypted by Argo CD (KSOPS) at sync.
Fine for small teams; you lose central rotation and audit.

---

## P5-8 Debug an AccessDenied, and trim permissions

**Type:** Task · *The AWS skill you'll use for the rest of your career*

**Why:** least-privilege policies (P5-2, P5-4) mean you **will** hit
`AccessDenied`. The skill isn't writing policies, it's **naming which layer
denied the request** before changing anything. Guessing and widening the
policy is how `"Resource": "*"` creeps back in.

**The layers that can say no** (an explicit deny anywhere always wins):

| Layer | Lives in | Typical surprise |
|---|---|---|
| SCP | AWS Organizations (P12) | An `AdministratorAccess` role still denied. SCPs are a ceiling, not a grant |
| Permission boundary | On the role | The identity policy allows it, the boundary doesn't |
| Identity policy | The role or user | The action or resource ARN is slightly wrong |
| **Resource policy** | S3 bucket, KMS key, ECR repo, Secrets Manager secret | **S3 returns 403 while the role allows `s3:GetObject`**: the bucket policy denies it, or the **KMS key policy** doesn't let the role decrypt |
| VPC endpoint policy | The interface/gateway endpoint (P4-3) | Works from your laptop, fails from the pod that goes through the endpoint |
| Session policy / duration | The assumed session | A long CI job dies with an expired token after exactly 1 h: raise the role's max session duration or refresh mid-job |
| Trust policy | The target role | `AssumeRole` fails: the trust (*who may assume me*) and the caller's permission (*what I may try*) are **two different policies**; you probably fixed one |

- [ ] **Drill:** break one permission on purpose. Find the event in **CloudTrail** (`errorCode = AccessDenied`), read the principal and action, name the layer, then fix only that
- [ ] **Test as the real principal.** "Works for me" with your SSO admin proves nothing about the pod's role. Exec into the pod and run the call there
- [ ] **IAM Access Analyzer:** turn it on for the account; fix one finding (anything shared outside the account, or over-broad)
- [ ] **Unused permissions:** after a week of real traffic, check each workload role's **last accessed** data (IAM Access Advisor) and remove actions never used; re-run the CUJ
- [ ] Write the top 3 denials you hit and their layer into `docs/rca/` (P10-8)

---

## Prove it

- [ ] `aws iam list-users` → no human or CI user has access keys
- [ ] `grep -r` over all three repos and full Git history (Gitleaks) → no secrets
- [ ] Every pod's AWS identity is its own role (`sts get-caller-identity`), and the negative tests above all fail as expected

## Common mistakes

| Mistake | Seen in |
|---|---|
| Admin IAM user + `aws configure` with static keys | 04, 05 |
| Long-lived registry tokens in CI secrets | 04, 05 (Docker Hub) |
| Kubeconfig / tokens / AWS keys stored in CI | (04's scenarios 1–3) |
| OIDC trust `repo:org/repo:*` (any branch or PR) | 06 |
| Plain Kubernetes Secrets in Git ("base64 is not encryption") | 01 |
| One identity for every pod (the node role) | 05 (IRSA only for one controller) |
| App code with a hardcoded fallback password (`root`/`password`) | 04 (`server/config/db.js`) |
