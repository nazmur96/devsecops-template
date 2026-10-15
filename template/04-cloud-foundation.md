# Phase 4: Cloud foundation (Infrastructure as Code)

**Goal:** the AWS account holds a network, an EKS cluster, a registry and the
data stores, **all created by Terraform** from a reviewed repo, with state
stored safely, access explicit, and nothing clicked in the console.

**Why:** two of the six reference projects created EKS with `eksctl` command
flags (not reproducible, not reviewable), one installed controllers by hand,
one left the API endpoint open to the internet, and one committed Terraform
state containing database passwords to a public repo.

**Can start in parallel with P2/P3**: none of this needs your manifests.

---

## P4-1 State backend, done safely

**Type:** Story · *Do this before any other resource*

```hcl
# platform/bootstrap/main.tf: applied once, with local state, then never touched
resource "aws_s3_bucket" "tfstate" {
  bucket = "sockshop-tfstate-${data.aws_caller_identity.me.account_id}"   # globally unique
}
resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration { status = "Enabled" }
}
resource "aws_s3_bucket_server_side_encryption_configuration" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "aws:kms" }
  }
}
resource "aws_s3_bucket_public_access_block" "tfstate" {
  bucket                  = aws_s3_bucket.tfstate.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

```hcl
# platform/environments/dev/backend.tf
terraform {
  backend "s3" {
    bucket       = "sockshop-tfstate-<account-id>"
    key          = "dev/platform.tfstate"
    region       = "eu-central-1"
    encrypt      = true
    use_lockfile = true    # native S3 locking (Terraform ≥ 1.10); DynamoDB locking is the legacy way
  }
}
```

- [ ] Bucket versioned, KMS-encrypted, public access blocked, bucket policy limited to the admin role
- [ ] **State is a secret.** It contains every resource attribute, including generated passwords. Never commit it (P1-1 `.gitignore`); never print it in CI logs
- [ ] **Drill:** run `terraform apply` from two shells at once; the second waits for or fails on the lock
- [ ] Provider versions pinned; `default_tags` set (`Project`, `Environment`, `ManagedBy=terraform`)

> References 05 and 06 used DynamoDB for locking. That still works, but
> Terraform now locks natively in S3 and has deprecated the DynamoDB option.

---

## P4-2 Repository structure

**Type:** Task

```
platform/
├── bootstrap/                 # state bucket (local state, run once)
├── modules/
│   ├── vpc/
│   ├── eks/
│   ├── ecr/
│   └── data/                  # RDS / ElastiCache / DynamoDB / SQS, if used
└── environments/
    ├── dev/   { backend.tf, main.tf, terraform.tfvars }
    └── prod/  { backend.tf, main.tf, terraform.tfvars }
```

- [ ] Environments differ **only** in `*.tfvars` (sizes, CIDRs, endpoint mode, Multi-AZ). Reference 06's `env/dev|staging|prod.tfvars` is the model
- [ ] **Decision:** write your own modules (best for learning; references 05/06) or use `terraform-aws-modules/*` (best for real work). ADR

> **When it grows: split state into stacks.** One state per environment is
> right for this project. Once several people apply changes, or one bad apply
> could hit everything, split each environment into **stacks** with their own
> state key: `network` (VPC, endpoints) → `cluster` (EKS, add-ons, Karpenter)
> → `data` (RDS, caches, queues). A later stack reads an earlier one's
> **outputs** with `data "terraform_remote_state"`, and only those outputs
> are the contract, so you can refactor a stack's insides without breaking its
> readers. Benefits: a `destroy` in `data` can't touch the cluster, plans get
> faster, and teams can own a stack each. Keep outputs minimal. Apply in order,
> **non-prod first, never prod and non-prod from one plan**. Commit
> `.terraform.lock.hcl`; never commit state.

---

## P4-3 Network

**Type:** Story · *Decision (ADR) on NAT strategy (see P0-7)*

| Setting | dev (budget) | prod |
|---|---|---|
| AZs | 3 | 3 |
| Subnets | 3 public + 3 private | 3 public + 3 private |
| Nodes in | private (or public with ADR-0005 compensations) | **private** |
| NAT | **1** shared (or none + public nodes) | **1 per AZ** (an AZ outage doesn't cut egress) |
| VPC endpoints | S3 gateway (free) | S3/DynamoDB gateway + ECR api/dkr, STS, Logs interface |
| Flow logs | 7 days | 30 days |

> **Read first:** [Why can't my app connect to my database?](https://github.com/nazmur96/devops-learning-journal/blob/main/networking/why-cant-my-app-connect-to-my-database.md):
> security groups, why `0.0.0.0/0` and wide port ranges are risky, and
> **security group as the source** ("anything wearing the `app-sg` badge")
> instead of IP addresses. Every SG rule in this phase follows that pattern.

- [ ] **Endpoints are a cost decision too.** NAT charges per GB in both directions, even for traffic to AWS's own services, and every image pull, log line and STS call goes through it. **S3 and DynamoDB gateway endpoints: always** (they're free). **Interface endpoints** (ECR, STS, Logs, Secrets Manager; ~$0.01/h per AZ each) pay for themselves once a service pushes roughly **150–200 GB/month** through NAT. They're also more secure: no public path, and they support endpoint policies. Check the NAT line on the bill monthly (P10-10c)
- [ ] Subnet tags for load balancers: public `kubernetes.io/role/elb = 1`, private `kubernetes.io/role/internal-elb = 1`
- [ ] Flow logs on
- [ ] Outputs: VPC ID, subnet IDs by tier

---

## P4-4 EKS cluster with explicit access

**Type:** Story

```hcl
resource "aws_eks_cluster" "this" {
  name     = "sockshop-${var.env}"
  version  = var.kubernetes_version          # the minor chosen in P1-2 (same as kubectl + kind image); standard support only, never extended
  role_arn = aws_iam_role.cluster.arn

  access_config {
    authentication_mode                         = "API"   # Access Entries only; no aws-auth ConfigMap
    bootstrap_cluster_creator_admin_permissions = false   # no invisible admin
  }

  vpc_config {
    subnet_ids              = var.private_subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = var.env == "prod" ? false : true
    public_access_cidrs     = var.env == "prod" ? null : [var.admin_cidr]   # your IP/32, never 0.0.0.0/0
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator"]

  encryption_config {
    resources = ["secrets"]
    provider { key_arn = aws_kms_key.eks.arn }   # your own key: you control and audit it
  }
}

# Admin access is an explicit, reviewable entry
resource "aws_eks_access_entry" "admin" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = var.admin_role_arn        # your SSO role, not an IAM user
}
resource "aws_eks_access_policy_association" "admin" {
  cluster_name  = aws_eks_cluster.this.name
  principal_arn = var.admin_role_arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
  access_scope { type = "cluster" }
}
```

- [ ] `aws eks list-access-entries` shows exactly the entries in code, nothing else
- [ ] **Negative test:** from a role with no entry, `kubectl get pods` returns `Unauthorized`
- [ ] Control plane logs arriving in CloudWatch
- [ ] Endpoint **not** open to `0.0.0.0/0` (reference 05's default)

> Why Access Entries and not `aws-auth`: the ConfigMap lived inside the
> cluster (one bad indent locked everyone out), needed cluster access to
> edit, and the creator had invisible permanent admin. Access Entries are an
> AWS API: auditable in CloudTrail, scoped by namespace, and in Terraform.
> (Reference 02 used `aws-auth`; reference 04 explains Access Entries well.)

---

## P4-5 Node group

**Type:** Story

- [ ] Managed node group in **private** subnets across 3 AZs; instance type from P0-5 (`t3.large` ×3 dev floor for Sock Shop)
- [ ] Launch template with **IMDSv2 required** and `http_put_response_hop_limit = 1`, so pods can't reach the node's instance credentials (pods use Pod Identity instead, P5). Pass region/VPC ID explicitly to controllers that would otherwise read them from IMDS
- [ ] Node role: `AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryReadOnly`, `AmazonSSMManagedInstanceCore` (SSM instead of SSH). The CNI policy goes to the `vpc-cni` add-on's own Pod Identity role, not the node
- [ ] **No SSH key pair, no port 22** (reference 02 allowed SSH)
- [ ] Nodes `Ready`; `kubectl get nodes -L topology.kubernetes.io/zone` shows all 3 AZs

---

## P4-6 Add-ons and controllers as code

**Type:** Story · *Reference 06 is the model: nothing installed by hand*

```hcl
resource "aws_eks_addon" "vpc_cni" {
  cluster_name         = aws_eks_cluster.this.name
  addon_name           = "vpc-cni"
  configuration_values = jsonencode({ enableNetworkPolicy = "true" })   # needed for P8 NetworkPolicy
}
resource "aws_eks_addon" "pod_identity_agent" {
  cluster_name = aws_eks_cluster.this.name
  addon_name   = "eks-pod-identity-agent"
}
# + coredns, kube-proxy, aws-ebs-csi-driver, metrics-server
```

| Component | How | Identity |
|---|---|---|
| vpc-cni (network policy on), coredns, kube-proxy | `aws_eks_addon` | vpc-cni: Pod Identity |
| eks-pod-identity-agent | `aws_eks_addon` | — |
| aws-ebs-csi-driver + `gp3` StorageClass (encrypted, `WaitForFirstConsumer`) | `aws_eks_addon` | Pod Identity |
| metrics-server | `aws_eks_addon` | — |
| cert-manager (needed by the ADOT/OTel operator in P10, and by Linkerd if used) | `aws_eks_addon` (community add-on) or Helm | — |
| AWS Load Balancer Controller | `helm_release` (or Argo CD from P8) | Pod Identity |
| ExternalDNS (P9), Karpenter (P6), External Secrets (P5) | add-on / Helm, later phases | Pod Identity |

- [ ] Every add-on version pinned; `terraform plan` after apply shows **no changes**
- [ ] **Drill:** delete the LB Controller deployment by hand → `terraform apply` (or Argo CD self-heal in P8) restores it

> Reference 05 installed the LB Controller with `eksctl` + `helm` + a hand-patched
> IAM policy. That's drift from day one: nobody can rebuild it from the repo.

---

## P4-7 Container registry

**Type:** Story

```hcl
resource "aws_ecr_repository" "svc" {
  for_each             = toset(var.services)
  name                 = "sockshop/${each.key}"
  image_tag_mutability = "IMMUTABLE"
  image_scanning_configuration { scan_on_push = true }
  encryption_configuration { encryption_type = "KMS" }
}
```

- [ ] One repo per service; lifecycle policy (expire untagged after 1 day; keep the last N dev builds; keep release tags)
- [ ] **Drill:** pushing an existing tag fails (immutability)
- [ ] Optional: enable **ECR enhanced scanning** (Amazon Inspector) for continuous rescans of stored images (guide, Phase 4 "registry rescans")

---

## P4-8 Artifact repository (dependency proxy): ECR pull-through, CodeArtifact, or Nexus

**Type:** Decision (ADR) + Story

**What it is.** An artifact repository is a private store that sits between
your builds and the public internet. It **proxies and caches** public
registries (Maven Central, npm, PyPI, Go proxy, Docker Hub) and **hosts** your
own artifacts (internal libraries, JARs, Helm charts, images). Every build
then pulls dependencies from **one place you control**.

**Why it matters (and why none of the videos showed it):**

| Without one | With one |
|---|---|
| Every CI run downloads from npm/Maven/Docker Hub directly | Dependencies cached once; builds faster and still work when a public registry is down |
| Docker Hub rate limits break builds | Pull-through cache absorbs them |
| A package unpublished upstream (or a malicious new version) reaches your build instantly | You can block, pin or quarantine components centrally |
| Nodes and CI runners need open internet egress | Egress allow-list = your repository only (pairs with private subnets / no NAT) |
| "Which builds used log4j 2.14?" is a grep across CI logs | The repository knows what was downloaded, by whom |

**Options on EKS:**

| Option | What | Pros | Cons | Pick when |
|---|---|---|---|---|
| **ECR pull-through cache** | ECR mirrors Docker Hub, GHCR, Quay, ECR Public, `registry.k8s.io` | Managed, IAM-native, same scanning as your images; nodes pull everything from ECR | Container images only | **Always**, for upstream images (base images, Helm chart images) |
| **AWS CodeArtifact** | Managed proxy/host for npm, Maven, PyPI, NuGet, Cargo, generic | No servers; IAM auth (CI via OIDC, short-lived tokens); upstream connections to the public registries | Language packages only; per-request + storage cost; AWS-only | **Default** for language packages in this template |
| **Sonatype Nexus Repository** | Self-hosted universal repo: Maven, npm, PyPI, Go, Helm, Docker, raw | One tool for everything; very common in enterprises (especially Java shops); works on-prem and air-gapped; repository "firewall" features in the paid tier | **You run it**: a stateful app with storage, backups, upgrades, HA (paid); the free Community Edition has usage caps (check the current limits) | You want the **enterprise skill**, a multi-cloud/on-prem story, or one repo for every format. Common interview topic |
| JFrog Artifactory | Commercial equivalent of Nexus | Same | Licence | Mentioned for completeness |
| Harbor | Self-hosted **OCI/container** registry with scanning and signing | CNCF, great for images + Helm OCI charts | Not for language packages | Self-hosting images instead of ECR (reference 07, SOCKPLAT-85) |

**Default path (managed):**

- [ ] **ECR pull-through cache** rules for Docker Hub (credentials in Secrets Manager), GHCR, Quay and `registry.k8s.io`; manifests and Helm values reference `<account>.dkr.ecr.<region>.amazonaws.com/docker-hub/library/...`
- [ ] **CodeArtifact** domain + repositories (`npm-store`, `maven-store`, `pypi-store`) with upstream connections to the public registries; one `internal` repo for your own packages
- [ ] CI authenticates with **OIDC** (P5-2), then `aws codeartifact login --tool npm|pip|…` (12-hour token, never stored); Maven uses a token from `get-authorization-token` in `settings.xml` via an env var
- [ ] Developers use the same path from the devcontainer (`mise` task `login-artifacts`)
- [ ] **Prove it:** block the build runner's direct internet egress to `registry.npmjs.org` → the build still succeeds through CodeArtifact. Kyverno `restrict-registries` (P8-6) allows only your ECR, so upstream images *must* come through the pull-through cache

**[OPT] Nexus track (enterprise skill).** *Use when:* you are aiming at enterprise or Java-heavy employers, need on-prem or air-gapped builds, or want one repository for every package format. Skip if CodeArtifact + ECR cover you:

- [ ] Nexus Repository deployed **by Argo CD** as a StatefulSet (official Helm chart) with a `gp3` PVC for its database and an **S3 blob store** (Pod Identity role scoped to one bucket) for artifacts
- [ ] Repositories: `npm-proxy` + `npm-hosted` → `npm-group`; same for Maven, PyPI, Go; a Docker **proxy** for Docker Hub and a Docker **hosted** repo
- [ ] Admin password rotated on first boot (stored in Secrets Manager, synced by ESO); anonymous access disabled; one CI user/token per pipeline, stored as a GitHub **environment** secret (Nexus has no native AWS OIDC, a real trade-off vs CodeArtifact)
- [ ] TLS through the internal ALB; **not** internet-facing
- [ ] Cleanup policies (e.g. delete proxied components unused for 90 days) + scheduled compaction; **backup** of the DB and blob store included in the DR drill (P10-9)
- [ ] Builds switched over: `.npmrc` `registry=`, Maven `settings.xml` mirror, `pip.conf` `index-url`, `GOPROXY`
- [ ] ADR comparing Nexus vs CodeArtifact on cost, operations, auth, and lock-in

> **The idea to remember for interviews:** an artifact repository turns
> "the internet" into one controlled, cached, auditable dependency source.
> On AWS the managed pieces are ECR pull-through + CodeArtifact; Nexus (or
> Artifactory) is the self-hosted, any-format equivalent most enterprises run.

---

## P4-9 Managed data stores (where the app allows)

**Type:** Story · *Reference 06's data plane, with its compromises fixed*

| Store | Managed option | Notes |
|---|---|---|
| Relational | RDS / Aurora | `manage_master_user_password = true`: the password lives in Secrets Manager and **never enters Terraform state** |
| Cache | ElastiCache | `transit_encryption_enabled = true` (reference 06 had it off) |
| Document | DocumentDB, or keep MongoDB in-cluster | Sock Shop expects MongoDB; in-cluster StatefulSet is acceptable (ADR) |
| Queue | SQS / Amazon MQ | Sock Shop expects RabbitMQ → Amazon MQ, or in-cluster |

- [ ] `publicly_accessible = false`, storage encrypted, SG ingress **only from the cluster/pod SG** (never a CIDR); **one SG per data store, never shared**, so compromising one service doesn't give network reach to every database
- [ ] **When the app can't connect**, debug in gate order (from the database article above): `Connection timed out` = security group/network; `Connection refused` = nothing listening; `no pg_hba.conf entry` = DB access rules; `password authentication failed` = credentials. Test with `kubectl run -it --rm dbg --image=postgres:16 -- psql -h <endpoint> -U app -d mydb` from inside the cluster
- [ ] Dev vs prod in tfvars: `multi_az`, `deletion_protection`, `backup_retention_period`, `skip_final_snapshot = false` in prod
- [ ] **Connection limits:** sum `peak connections per pod × maxReplicas` across every service using the database (from the manifests, P0-2b) and compare it with the instance's `max_connections`. If a scale-out could exceed it, add pooling in the app, or **RDS Proxy** in front. "The database fell over" during a traffic spike is very often this
- [ ] **[OPT] RDS IAM authentication** (*use when:* you want no database password at all; ⏸ wire it after Pod Identity exists, P5-4): the pod's Pod Identity role gets `rds-db:connect`, and the app requests a 15-minute token instead of a stored password. Fewer secrets to rotate
- [ ] **Why you might leave RDS later** (write it in the ADR so you'd recognise it): avoiding cloud lock-in, cost at large scale *if* you already run Kubernetes maturely, data sovereignty or air-gapped rules, or running everything on one platform. Without one of those, RDS is the right default. (SQLite + Litestream is a third option, but only for single-writer apps on one node)
- [ ] **[OPT] Operator paradigm** (*use when:* you want to learn the operator pattern or can't use a managed database): CloudNativePG in-cluster vs RDS, with failover and PITR drills and an ADR comparing RTO/RPO, cost and on-call burden (reference 07, SOCKPLAT-52–55)

---

## P4-10 Keep it honest: tests, drift, cost

**Type:** Story

- [ ] `checkov -d platform/` clean (or justified skips): runs in CI from P7
- [ ] **[OPT] Terratest** (*use when:* your modules are reused by other teams or environments, so a broken module breaks many things): apply the VPC module to a throwaway workspace, assert subnets/tags/SG rules (e.g. "no data-store SG allows `0.0.0.0/0`"), destroy
- [ ] **CloudTrail trail** in Terraform: management events for the account to an encrypted S3 bucket with a lifecycle rule (e.g. 90 days), log-file validation on. The console's 90-day Event history works without it, but drift alerts below, EventBridge-driven automation (P10-5c) and any real investigation need a trail
- [ ] **Drift detection:** scheduled `terraform plan -detailed-exitcode` in CI (exit 2 = drift → alert); a CloudTrail/EventBridge rule for console changes to SGs. **Drill:** add an SG rule in the console, see the alert
- [ ] **Cost:** budget alert (P0-7); `make destroy` documented; optional scheduled scale-to-zero of dev node groups at night

---

## Prove it

- [ ] `terraform destroy && terraform apply` rebuilds the whole environment with **no manual step**, then the CUJ passes (re-check this once the app is deployed: by hand in P6-0, by Argo CD from P8)
- [ ] No IAM user access keys exist for humans or CI (`aws iam list-access-keys` empty for your users)
- [ ] `checkov` clean; `terraform plan` clean after apply

## Common mistakes

| Mistake | Seen in |
|---|---|
| `eksctl create cluster` with CLI flags (not in Git) | 02, 04 |
| Public API endpoint open to the world | 05 (default), 06 (dev) |
| `aws-auth` ConfigMap, invisible creator admin | 02; 06 kept `bootstrap…admin = true` |
| Controllers installed by hand / IAM policy patched by hand | 05 |
| Terraform state committed (with passwords) | 05 (instructor's state), 06 (state backup) |
| Admin IAM user with static keys for everything | 04, 05 |
| Old pinned Kubernetes version past standard support | 05 (1.30), 07 (1.30) |
| Data-store SG open to a CIDR instead of a source SG | — (06 got this right) |
