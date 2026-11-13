# 14 — Security and secrets

## The one rule

**Anything Terraform manages ends up in state, in plain text.**

Not hashed. Not encrypted at the field level. If a provider returns a database
password, it is a string in your `.tfstate`.

`sensitive = true` hides a value from console output. It does nothing to the
file.

Everything below follows from that.

## Keeping secrets out of Terraform

### Manage the container, not the value

The cleanest answer is not to put the secret in Terraform at all. Terraform
creates the *place* a secret lives; something else fills it in.

That is `modules/secrets/main.tf` in full:

```hcl
# Secrets Manager secrets: the containers only. Terraform manages each secret's name,
# description, encryption key and tags. It never manages the secret value
# (no aws_secretsmanager_secret_version), so values stay out of code and state.
# Set or rotate values outside Terraform, for example in the AWS console.

resource "aws_secretsmanager_secret" "this" {
  for_each = var.secrets

  name        = each.key
  description = each.value.description
  kms_key_id  = each.value.kms_key_id

  tags = var.tags
}
```

The absence is the design. There is no `aws_secretsmanager_secret_version`, so
no value can ever reach state. Terraform owns naming, encryption and tagging —
the things that benefit from review — and a human or a rotation Lambda owns the
value.

**Generalise this:** manage the lifecycle, not the contents. The same shape
works for SSM parameters, Kubernetes secrets and KMS keys.

### Read a secret at apply time when you must

```hcl
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db.secret_string
}
```

> This **does** put the value in state. It is better than hardcoding, but worse
> than not managing the password in Terraform at all. Prefer generating a
> password in the provider and letting AWS own it (`manage_master_user_password`
> on RDS), or setting it out of band.

### Never do these

```hcl
variable "db_password" {
  default = "hunter2"            # in git forever
}

provider "aws" {
  access_key = "AKIA..."         # in git forever
  secret_key = "..."
}
```

Credentials come from the environment — a profile, an instance role, OIDC. Never
from a `.tf` file.

If a credential does reach git, **rotate it**. Rewriting history does not help:
it was cloned, cached and indexed the moment it was pushed.

## Protecting state

Because state holds secrets, the state bucket is a security boundary.
`bootstrap/state.tf` is a worked example:

| Control | Why |
|---------|-----|
| `encrypt = true` in the backend | Encrypted at rest |
| `aws_s3_bucket_server_side_encryption_configuration` | Enforced on the bucket |
| `aws_s3_bucket_public_access_block` (all four true) | Never public |
| `aws_s3_bucket_ownership_controls` → `BucketOwnerEnforced` | ACLs disabled |
| `aws_s3_bucket_versioning` | Recovery from corruption |
| Bucket policy denying insecure transport | No plain HTTP, no TLS < 1.2 |
| `prevent_destroy = true` | Cannot be deleted by accident |
| IAM scoped to `envs/*` | CI cannot read bootstrap state |

The transport rule matches the platform's own guardrail:

```hcl
# Matches RCP's S3 guardrail on the bucket itself: no plain HTTP, and no TLS older than 1.2.
statement {
  sid     = "DenyInsecureTransport"
  effect  = "Deny"
  actions = ["s3:*"]
  # ...
}
```

### SSE-S3 vs KMS

`arifa-infra` uses SSE-S3 (`AES256`) and documents the gap rather than
pretending it is fine:

```hcl
# SSE-S3 until the account's KMS key rules are known: a customer-managed key
# needs a key policy that says which roles may decrypt state (docs/roadmap.md).
#trivy:ignore:AVD-AWS-0132
```

A customer-managed KMS key is better — it lets you say *which roles may decrypt
state* and gives you an audit trail of every decrypt. But a KMS key with a wrong
key policy locks everyone out of their own state, so doing it deliberately
beats doing it quickly.

Be honest about accepted risk. An inline ignore with a reason and a roadmap link
is a record; a global suppression is a cover-up.

## Least privilege

### Scope by path

```hcl
statement {
  sid       = "ReadEnvironmentState"
  actions   = ["s3:GetObject"]
  resources = ["${aws_s3_bucket.state.arn}/envs/*"]
}
```

CI reads environment state. It cannot read `bootstrap/*`, which contains the
IAM configuration.

### Deny what the job never needs

```hcl
statement {
  sid       = "DenySecretValues"
  effect    = "Deny"
  actions   = ["secretsmanager:GetSecretValue", "secretsmanager:BatchGetSecretValue"]
  resources = ["*"]
}
```

A plan reads *configuration*, never data. Since Terraform here manages secret
containers only, nothing legitimate ever needs a secret value — so deny it
outright. An explicit `Deny` overrides any `Allow`, including `ReadOnlyAccess`.

### Short-lived credentials

OIDC (chapter 13) means no stored keys and a token scoped to one workflow run:

```hcl
max_session_duration = 3600
```

### Separate read from write

CI plans; humans apply. The pipeline's credentials cannot change anything, so
compromising the pipeline does not compromise the infrastructure.

### Permissions boundaries

```hcl
resource "aws_iam_role" "github_plan" {
  permissions_boundary = local.permissions_boundary_arn
}
```

A ceiling on what the role can ever be granted, even if someone later attaches a
broader policy. Common in managed platform accounts.

## Guarding the account

```hcl
provider "aws" {
  region              = local.region
  allowed_account_ids = [local.account_id]
}
```

Terraform refuses to run if the credentials resolve elsewhere. One line, and it
prevents the classic "I had the wrong profile exported" incident.

## Secret scanning

```yaml
scan:
  scanners:
    - misconfig
    - secret
```

trivy scans every file for credential patterns, not just `.tf`. Run it in
pre-commit so a key is caught before the commit exists:

```yaml
- id: detect-private-key
```

Enable GitHub push protection as the backstop.

`arifa-infra` explains why it skips one common hook:

> **No `detect-aws-credentials` hook:** it only finds keys in local AWS CLI
> files, which SSO users do not have.

Knowing why a control does not apply is as useful as enabling it.

## Resource-level security

The notebook module hardcodes IMDSv2:

```hcl
instance_metadata_service_configuration {
  minimum_instance_metadata_service_version = "2"
}
```

IMDSv1 is vulnerable to SSRF: a request through a vulnerable application can
read instance credentials. IMDSv2 requires a session token, which blocks that.

It is **not a variable**, because it should never vary. Making a security
default configurable invites someone to configure it off.

The roadmap's remaining items are the rest of the hardening: disable root
access, move into a private subnet, disable direct internet access, add a
security group, use a customer-managed KMS key for the volume. Two of those
replace the instance — which is why they are planned rather than rushed.

## Checklist

**Secrets**
- [ ] No credential in any `.tf` file, including defaults
- [ ] Secret values not managed by Terraform where avoidable
- [ ] `sensitive = true` on anything that must pass through
- [ ] Secret scanning in pre-commit and CI
- [ ] Push protection enabled

**State**
- [ ] Remote backend with `encrypt = true`
- [ ] Versioning on
- [ ] Public access blocked
- [ ] `prevent_destroy` on the bucket
- [ ] IAM scoped per path
- [ ] `*.tfstate` in `.gitignore`

**Access**
- [ ] OIDC rather than stored keys
- [ ] CI cannot apply, or applies only a reviewed saved plan
- [ ] Trust policy pins the repository with `StringEquals`
- [ ] Explicit denies for data reads
- [ ] `allowed_account_ids` set

**Resources**
- [ ] Encryption at rest
- [ ] No public exposure by default
- [ ] Security defaults hardcoded, not configurable
- [ ] Misconfiguration scanning in CI

## Try it

```bash
cd arifa-infra

grep -rn "secretsmanager_secret_version" .     # expect nothing — that is the point
grep -rn "sensitive" .
grep -rn "allowed_account_ids" .
grep -rn "trivy:ignore" .

bash scripts/check.sh scan                     # needs trivy
```

## Check yourself

1. What does `sensitive = true` protect, and what does it not?
2. Why does `modules/secrets` deliberately omit a resource?
3. Why is the state bucket a security boundary?
4. Why SSE-S3 here, and what would make KMS the better choice?
5. Why can an explicit `Deny` override `ReadOnlyAccess`?
6. Why is IMDSv2 hardcoded instead of being a variable?
7. A key is committed and pushed. What is the first thing you do?

---

Previous: [13 — CI/CD](13-cicd-and-oidc.md) ·
Next: [15 — Best practices](15-best-practices.md)
