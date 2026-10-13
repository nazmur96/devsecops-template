# Phase 12 (later): Graduate to one AWS account per environment

**When:** after you've built a few projects with phases 0–11 in **one
account**. Not before: a second account adds cost and cross-account IAM, and
it's much easier once precise single-account IAM is second nature.

**Use when** (any one is enough): the system holds **real customer data**;
**more than one team** deploys to it; a **compliance** framework (SOC 2, ISO
27001, PCI) asks for environment isolation; or you want the cost of each
environment on its own bill.

**Goal:** dev, staging and prod live in separate AWS accounts, managed by AWS
Organizations, with guardrails nobody can switch off, and only the deliberate
"doors" between them that the pipeline needs.

---

## The idea, in plain English

- **One account = one house with locked rooms.** Dev, staging and prod are
  rooms. The locks are the IAM policies you write. Write one lock wrong (a
  policy with `"Resource": "*"`) and a key for the dev room also opens prod.
- **Separate accounts = separate houses.** By default nobody from the dev
  house can enter the prod house at all. To let something cross, you
  deliberately build a door, and you build it from **both sides**.

Most of this phase is **IAM roles and policies**: the same skill you used in
one account, plus explicit doors.

## What changes, side by side

| Question | One account (phases 0–11) | Separate accounts (this phase) |
|---|---|---|
| What stops dev from touching prod? | **Your IAM policies**, scoped by names and tags (`sockshop-prod-*`, `Environment=prod`). One mistake can cross environments | **The account boundary.** Nothing crosses unless you explicitly allow it |
| How does CI push images? | CI role → ECR in the same account | ECR lives in a shared **tooling** account; CI pushes there; each environment's nodes **pull across accounts** via an **ECR repository policy** ("accounts 222 and 333 may pull") |
| How does a role reach another account? | Not needed | **Cross-account role assumption**: the target role's **trust policy** says "I trust role X from account 111", **and** the caller's policy allows `sts:AssumeRole` on it. That's the two-sided door |
| Secrets and encryption keys | Policies limit who reads `sockshop/prod/*` | Each account has its own Secrets Manager and KMS keys; sharing across accounts needs a **resource policy** on the secret or key as well |
| Terraform | One state bucket, one set of credentials | State per account (or one state account), and Terraform **assumes a role per environment** |
| DNS | One hosted zone | The main zone in one account; **subdomains delegated** (`dev.example.com`, `staging.example.com`) to each environment's account |
| Guardrails | Only IAM | **SCPs** (Service Control Policies) from AWS Organizations sit above every IAM policy: "nobody in prod may disable CloudTrail or leave our regions", and not even an admin in that account can override them |
| Bills and limits | One bill, split by tags; service quotas shared by every environment | A separate bill per environment; a dev load test can't use up prod's quotas |
| Effort | Lowest | More accounts, roles and policies to write and debug |

---

## P12-1 Design the account structure

**Type:** Decision (ADR)

```
Organization (management account: billing + Organizations only, no workloads)
├── OU Security
│   ├── log-archive     CloudTrail + Config + VPC flow logs from every account, write-once
│   └── security-tools  GuardDuty / Security Hub delegated admin
├── OU Shared
│   └── tooling         ECR, CodeArtifact, CI roles, Route 53 root zone, (central Argo CD)
├── OU NonProd
│   ├── dev
│   └── staging
└── OU Prod
    └── prod
```

- [ ] Account list and OUs agreed; one email alias per account; naming `sockshop-<env>` (P0-12)
- [ ] The management account runs **no workloads**
- [ ] **AWS Control Tower** (guided setup with landing-zone guardrails) vs plain **Organizations + Terraform**: ADR. Control Tower is faster to start; Terraform is more transparent for learning

## P12-2 Human access across accounts

**Type:** Story

- [ ] **IAM Identity Center** in the management (or a delegated) account; permission sets assigned per account (`PlatformAdmin` in all; `Developer` read-only in staging/prod, edit in dev)
- [ ] `aws sso login` gives a profile per account (`sockshop-dev`, `sockshop-prod`); `mise.toml` sets the right `AWS_PROFILE` per repo/environment
- [ ] EKS Access Entries in each cluster reference that account's SSO roles (P4-4, P5-1)
- [ ] **Negative test:** the `Developer` permission set can't change anything in prod

## P12-3 Guardrails with SCPs

**Type:** Story

- [ ] SCPs on the OUs, starting with:
  - deny leaving the Organization
  - deny disabling CloudTrail, Config, GuardDuty
  - deny regions you don't use
  - deny creating IAM users / access keys (SSO and roles only)
  - prod OU: deny deleting the log-archive bucket, deny public S3 buckets
- [ ] SCPs tested in a sandbox OU first (an SCP mistake can lock out a whole account)
- [ ] **Negative test:** an admin in dev can't create an IAM user

## P12-4 Central logging and security

**Type:** Story

- [ ] Organization-wide **CloudTrail** to the `log-archive` account (S3 with Object Lock, so logs can't be altered)
- [ ] **GuardDuty** and **Security Hub** with a delegated admin in `security-tools`, auto-enabled for new accounts
- [ ] AWS Config with conformance packs (optional)

## P12-5 Shared registry and cross-account pulls

**Type:** Story

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowEnvAccountsToPull",
    "Effect": "Allow",
    "Principal": { "AWS": ["arn:aws:iam::222222222222:root", "arn:aws:iam::333333333333:root"] },
    "Action": ["ecr:BatchGetImage", "ecr:GetDownloadUrlForLayer", "ecr:BatchCheckLayerAvailability"]
  }]
}
```

- [ ] ECR repos (and the ECR pull-through cache, CodeArtifact) in **tooling**; a **repository policy** like the one above (or one keyed on the Organization ID with `aws:PrincipalOrgID`) lets the environment accounts pull
- [ ] The node role in each environment keeps `AmazonEC2ContainerRegistryReadOnly` (its half of the door)
- [ ] Images encrypted with a KMS key whose **key policy** also allows the environment accounts to decrypt
- [ ] CI's OIDC role lives in **tooling** and can only push there
- [ ] Kyverno's `verifyImages` and `restrict-registries` (P8-6) now point at the tooling account's registry
- [ ] **Prove it:** a prod pod pulls an image from the tooling account; a random other account can't

## P12-6 Terraform across accounts

**Type:** Story

```hcl
provider "aws" {
  region = "eu-central-1"
  assume_role {
    role_arn = "arn:aws:iam::${var.account_id}:role/terraform-deployer"   # one per environment account
  }
  default_tags { tags = local.tags }
}
```

- [ ] A `terraform-deployer` role in each account, **trusting only** the CI apply role (and the platform admins' SSO role)
- [ ] State: one bucket per account, or a single state account with per-environment prefixes and bucket policies
- [ ] `environments/{dev,staging,prod}` now differ in `account_id` as well as tfvars
- [ ] The CI apply role for prod requires the GitHub `prod` environment approval (P5-3)

## P12-7 DNS delegation

**Type:** Task

- [ ] Root zone `example.com` in tooling; `dev.example.com` and `staging.example.com` hosted zones in their own accounts, delegated with NS records in the root zone; prod uses the root zone (or `prod.` delegated)
- [ ] ExternalDNS and ACM in each account only touch their own zone

## P12-8 Argo CD across accounts

**Type:** Decision (ADR) + Story

| Model | How | Pick when |
|---|---|---|
| **One Argo CD per cluster** | Each cluster runs its own Argo CD pointing at the same gitops repo (its own folder) | **Simplest and safest**: a compromise of one Argo CD can't deploy to another account. Template default |
| **Hub-and-spoke** | One Argo CD in tooling manages all clusters; it reaches each cluster through an IAM role in that account + an EKS Access Entry for it | Many clusters; one place to see everything. The hub becomes very powerful: protect it accordingly |

- [ ] Model chosen; for hub-and-spoke: a role per spoke account trusting the hub's Pod Identity role, an Access Entry in each spoke cluster, and an ApplicationSet with a cluster generator (P8-9)

## P12-9 Secrets and keys per account

**Type:** Task

- [ ] Each account has its own Secrets Manager secrets and KMS keys; **don't share prod secrets** into other accounts
- [ ] Where something must be shared (e.g. a KMS key for images, P12-5), use **resource policies** limited to the specific accounts/roles

## P12-10 Migrate from the single account

**Type:** Story

- [ ] Order: tooling (ECR, CI) → dev → staging → prod last
- [ ] Rebuild rather than move: with everything in Terraform + GitOps + backups, a new account is `terraform apply` + Argo CD sync + data restore (your DR drill, P10-9, *is* the migration rehearsal)
- [ ] Old single-account resources destroyed after the CUJ passes in the new accounts; budgets set per account

---

## Prove it

- [ ] A dev-account admin can't see or change anything in prod (cross-account negative tests)
- [ ] Prod pulls images from tooling; a non-Organization account can't
- [ ] An SCP blocks an action even for an account admin
- [ ] CloudTrail from every account lands in log-archive and can't be deleted
- [ ] The CUJ passes in staging and prod accounts after the migration

## Common mistakes

| Mistake | Fix |
|---|---|
| Running workloads in the management account | Management = billing + Organizations only |
| Testing a new SCP directly on the prod OU | Sandbox OU first |
| Only one side of the door: trust policy without the caller's `sts:AssumeRole` permission (or the reverse) | Always check both sides |
| ECR repository policy allows the account, but the KMS key policy doesn't | Images encrypted with a customer key need both |
| Sharing prod secrets into lower accounts "for convenience" | Each environment gets its own secrets |
| A central Argo CD with admin rights everywhere and no extra protection | One Argo CD per cluster, or harden the hub |
