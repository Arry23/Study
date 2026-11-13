# 05 — Resources and data sources

Two block types do the real work. `resource` creates and owns something.
`data` only reads something.

## Resources

```hcl
resource "aws_s3_bucket" "state" {
  bucket = local.state_bucket_name
}
```

- `aws_s3_bucket` — the **type**. The prefix before the first underscore names
  the provider.
- `state` — the **local name**. Yours to choose. Unique per type per directory.
- Together, `aws_s3_bucket.state` is the **address**, used for references, in
  state, and in CLI commands.

Terraform now owns this bucket. It will create it, keep it matching the config,
and **destroy it if you delete the block**.

### Naming

The local name is a Terraform identifier, not an AWS name. Two conventions:

```hcl
resource "aws_s3_bucket" "state" { }    # descriptive: there are several buckets
resource "aws_s3_bucket" "this" { }     # "this": the module's single main resource
```

`arifa-infra` uses both correctly. In `bootstrap/state.tf` every bucket resource
is named `state` because they relate to one subject. In
`modules/secrets/main.tf` it is `this`, because a module that manages one kind
of thing has an obvious subject.

Do not put the type in the name. `aws_s3_bucket.state_bucket` reads as
"s3 bucket state bucket".

### Attributes you set vs attributes you read

```hcl
resource "aws_s3_bucket" "state" {
  bucket = local.state_bucket_name   # you set this
}

# you read these — AWS fills them in
aws_s3_bucket.state.arn
aws_s3_bucket.state.id
aws_s3_bucket.state.region
```

Attributes you cannot know until after creation (ARNs, IDs, IP addresses) show
as `(known after apply)` in a plan. That is normal.

### One concern per resource

Modern AWS resources are deliberately small. A bucket's versioning, encryption,
public-access block, lifecycle and policy are each a separate resource:

```hcl
resource "aws_s3_bucket"                              "state" { }
resource "aws_s3_bucket_versioning"                   "state" { }
resource "aws_s3_bucket_server_side_encryption_configuration" "state" { }
resource "aws_s3_bucket_public_access_block"          "state" { }
resource "aws_s3_bucket_ownership_controls"           "state" { }
resource "aws_s3_bucket_lifecycle_configuration"      "state" { }
resource "aws_s3_bucket_policy"                       "state" { }
```

Seven resources for one bucket — which is exactly the `Plan: 7 to add` that
`arifa-infra`'s README tells you to expect from `make bootstrap-plan`.

Each takes `bucket = aws_s3_bucket.state.id`, which creates the dependency.

## Dependencies

Terraform builds a graph from references. It does not read top to bottom.

```hcl
resource "aws_s3_bucket" "state" {
  bucket = local.state_bucket_name
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id     # <-- implicit dependency
  versioning_configuration { status = "Enabled" }
}
```

Terraform cannot know `.id` until the bucket exists, so it orders them. It also
parallelises everything unrelated — by default ten operations at once.

### When references are not enough

Sometimes an ordering requirement exists that no reference expresses. Then say
so explicitly:

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "state" {
  # ...
  # Lifecycle rules on noncurrent versions need versioning to be on first.
  depends_on = [aws_s3_bucket_versioning.state]
}

resource "aws_s3_bucket_policy" "state" {
  # ...
  # The public access block must exist before a bucket policy is attached.
  depends_on = [aws_s3_bucket_public_access_block.state]
}
```

Both are from `bootstrap/state.tf`, and both carry a comment saying *why* —
because `depends_on` without a reason is impossible to review later.

> Use `depends_on` sparingly. If you need it often, you are probably missing a
> reference you could use instead.

## Data sources

A `data` block reads something Terraform does not manage:

```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

locals {
  account_id = data.aws_caller_identity.current.account_id
}
```

Reference them with a `data.` prefix: `data.aws_caller_identity.current.account_id`.

Data sources are read during plan, never created or destroyed.

### Use them for things other teams own

The pattern matters more than the syntax. When another team owns the VPC, do
not create one — look it up:

```hcl
data "aws_vpc" "main" {
  tags = { Name = "rcp-main" }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  tags = { Tier = "private" }
}

resource "aws_sagemaker_notebook_instance" "this" {
  subnet_id = data.aws_subnets.private.ids[0]
}
```

`arifa-infra` has this on its roadmap (item 6, "Read RCP network via data
sources") and the notebook module already anticipates it:

```hcl
# Networking belongs to the platform team: pass existing IDs in, never create them here.
subnet_id       = var.subnet_id
security_groups = var.security_group_ids
```

### Building IAM policies

The most common data source in AWS projects is not a lookup at all:

```hcl
data "aws_iam_policy_document" "state_bucket" {
  statement {
    sid     = "DenyInsecureTransport"
    effect  = "Deny"
    actions = ["s3:*"]
    resources = [
      aws_s3_bucket.state.arn,
      "${aws_s3_bucket.state.arn}/*",
    ]
    # ...
  }
}

resource "aws_s3_bucket_policy" "state" {
  bucket = aws_s3_bucket.state.id
  policy = data.aws_iam_policy_document.state_bucket.json
}
```

This generates JSON, validated at plan time. A hand-written heredoc is only
validated when AWS rejects it.

Note `not_resources` and `not_actions`, which express "everything except" —
`bootstrap/github_oidc.tf` uses that to deny reading every S3 object except the
state path:

```hcl
statement {
  sid           = "DenyDataReads"
  effect        = "Deny"
  actions       = ["s3:GetObject", "s3:GetObjectVersion"]
  not_resources = ["${aws_s3_bucket.state.arn}/envs/*"]
}
```

## `lifecycle`

A nested block that changes how Terraform treats the resource. Full detail in
chapter 08, but one rule belongs here:

```hcl
resource "aws_s3_bucket" "state" {
  bucket = local.state_bucket_name

  lifecycle {
    # Losing this bucket loses track of everything Terraform manages.
    prevent_destroy = true
  }
}
```

`prevent_destroy = true` makes `terraform destroy` — and any plan that would
replace this resource — fail with an error instead of proceeding.

**Put it on anything whose loss is unrecoverable:** state buckets, production
databases, KMS keys with encrypted data behind them.

## Replacement

Some attributes cannot be changed in place. Change one and Terraform destroys
and recreates the resource. The plan says so:

```
# aws_sagemaker_notebook_instance.this must be replaced
-/+ resource "aws_sagemaker_notebook_instance" "this" {
      ~ subnet_id = null -> "subnet-0abc123" # forces replacement
```

**`# forces replacement` is the most important comment in any plan.** Read every
plan for it. For the notebook it means the instance is deleted and rebuilt, so
anything on its volume is gone — which is why `arifa-infra`'s roadmap flags
items 9 and 10 as replacements and says the volume must be backed up first.

## Try it

```bash
cd arifa-infra

# every resource in the bootstrap root
grep -E '^resource' bootstrap/*.tf

# every data source
grep -rE '^data' bootstrap/ envs/

# the dependency graph
terraform -chdir=bootstrap graph
```

Then answer from the code alone: which resource must exist before
`aws_s3_bucket_policy.state`, and how do you know?

## Check yourself

1. Difference between `resource` and `data`?
2. What is the address of `resource "aws_s3_bucket" "state"`?
3. How does Terraform decide ordering without `depends_on`?
4. When is `depends_on` genuinely required?
5. What does `prevent_destroy = true` stop?
6. You see `# forces replacement` in a plan. What do you check before applying?
7. Why build IAM policy JSON with a data source?

---

Previous: [04 — Providers](04-providers.md) ·
Next: [06 — Variables, outputs and locals](06-variables-outputs-locals.md)
