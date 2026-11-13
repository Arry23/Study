# 07 — State and backends

**The most important chapter in this guide.** Nearly every confusing Terraform
problem is a state problem.

## What state is

A JSON file mapping each address in your code to a real resource ID.

```json
{
  "resources": [
    {
      "mode": "managed",
      "type": "aws_s3_bucket",
      "name": "state",
      "instances": [
        {
          "attributes": {
            "id":  "arifa-infrastructure-tfstate-483075907105-eu-central-1",
            "arn": "arn:aws:s3:::arifa-infrastructure-tfstate-...",
            "bucket": "arifa-infrastructure-tfstate-..."
          }
        }
      ]
    }
  ]
}
```

That record is the only link between `aws_s3_bucket.state` in your code and a
real bucket in AWS.

## Why it has to exist

Terraform could in principle query AWS for everything and compare. It does not,
for three reasons:

**Identity.** Your code says a bucket should exist. AWS has forty buckets. Which
one is *this* block? Only state knows.

**Deletion.** You delete a `resource` block. Without state, Terraform sees code
with no bucket and AWS with a bucket it has no opinion about. With state, it
sees "I used to manage this and it is gone from the code" — so it destroys it.

**Speed.** State caches attributes, so a plan reads far less from the API.

## The consequences

**Lose the state and Terraform forgets everything.** The resources keep running;
Terraform just no longer knows they are his. The next apply tries to create them
all again, and most fail with "already exists".

**State contains secrets in plain text.** Every attribute a provider returns is
stored, including database passwords and private keys. `sensitive = true`
affects console output, not the file.

> **Danger.** Treat the state file as a credential. Never commit it. Never put
> it in a public bucket. `arifa-infra`'s `.gitignore` excludes
> `*.tfstate` and `*.tfstate.*`, and the state bucket blocks all public access.

## Local vs remote

By default state is `terraform.tfstate` next to your code. That is fine for a
tutorial and wrong for a team:

- Only on your laptop, so nobody else can plan
- No locking, so two applies can corrupt it
- Not backed up
- Easy to commit by accident

A **backend** moves it somewhere shared.

## The S3 backend

```hcl
# envs/dev/backend.tf
terraform {
  backend "s3" {
    bucket       = "arifa-infrastructure-tfstate-483075907105-eu-central-1"
    key          = "envs/dev/terraform.tfstate"
    region       = "eu-central-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

| Argument | Meaning |
|----------|---------|
| `bucket` | Where state lives |
| `key` | Path within the bucket — one per root |
| `region` | The bucket's region |
| `encrypt` | Server-side encryption on the object |
| `use_lockfile` | **Native S3 locking** — no DynamoDB table needed |

### Backend blocks cannot use variables

```hcl
# This does NOT work
bucket = var.state_bucket
```

The backend is resolved before variables exist. Hence the comment in the file:
*"Backend blocks cannot use variables, so the values are literals."*

If you must parameterise, use partial configuration:

```hcl
terraform { backend "s3" {} }
```

```bash
terraform init -backend-config=dev.s3.tfbackend
```

`arifa-infra` chooses literals instead, for the same reason it avoids tfvars:
no hidden inputs.

### Locking

Two engineers apply at once; both read state, both write. One set of changes is
lost and state no longer matches reality.

A lock prevents it. Terraform writes a `.tflock` object, and anyone else waits:

```
Acquiring state lock. This may take a few moments...
```

Historically this needed a DynamoDB table. Since Terraform 1.11,
`use_lockfile = true` uses S3 conditional writes instead — which is why
`bootstrap/state.tf` says:

```hcl
# Locking uses S3 native lock files (use_lockfile), so there is no DynamoDB table.
```

If a run is killed mid-apply the lock can be left behind. Verify nothing is
running, then:

```bash
terraform force-unlock <LOCK_ID>
```

> Never `force-unlock` because you are impatient. If another apply is genuinely
> in flight, you will corrupt state.

## One state per root

A **root module** is a directory you run Terraform in. Each has its own state:

```
arifa-infra/
├── bootstrap/     -> s3://.../bootstrap/terraform.tfstate
└── envs/dev/      -> s3://.../envs/dev/terraform.tfstate
```

Benefits: a mistake in dev cannot touch bootstrap, plans stay fast, and the
blast radius of any single apply is bounded.

This is also why the CI role can read `envs/*` but not `bootstrap/*`:

```hcl
statement {
  sid       = "ReadEnvironmentState"
  actions   = ["s3:GetObject"]
  resources = ["${aws_s3_bucket.state.arn}/envs/*"]
}
```

## The bootstrap problem

The state bucket is itself Terraform-managed. So where does *its* state live
before it exists?

Answer: locally at first, then migrate. `bootstrap/backend.tf` is committed
**commented out**, with the procedure in the comment:

```
# Bootstrap creates the state bucket, so its own first apply has to use local state.
#
# Right after the first apply (README, demo step 2):
#   1. Uncomment the block below.
#   2. Run: terraform -chdir=bootstrap init -migrate-state
#   3. Answer "yes" to copy the local state into the bucket, then delete
#      bootstrap/terraform.tfstate* from your laptop.
#   4. Commit this file.
```

`-migrate-state` copies the existing state into the new backend. This is the
standard chicken-and-egg solution and worth recognising.

## Protecting the state bucket

`bootstrap/state.tf` is a checklist of what a state bucket needs:

| Resource | Why |
|----------|-----|
| `prevent_destroy = true` | Losing the bucket loses everything |
| `aws_s3_bucket_versioning` | **The recovery path** — restore an earlier state |
| `..._server_side_encryption_configuration` | State holds secrets |
| `aws_s3_bucket_public_access_block` | Never public |
| `aws_s3_bucket_ownership_controls` | Disable ACLs |
| `aws_s3_bucket_lifecycle_configuration` | Expire old versions after 90 days |
| `aws_s3_bucket_policy` | Deny plain HTTP and TLS < 1.2 |

**Versioning is the one that saves you.** When state is corrupted or a resource
is deleted by accident, you restore the previous object version. Without
versioning there is no undo.

## Reading and editing state

Inspect it:

```bash
terraform state list                    # every address
terraform state show aws_s3_bucket.state   # one resource's attributes
terraform show                          # everything
terraform show -json | jq               # machine-readable
```

Change it — carefully:

```bash
terraform state mv   <from> <to>    # rename without destroying
terraform state rm   <address>      # forget, but leave in AWS
terraform import     <address> <id> # adopt an existing resource (see ch. 11)
terraform state pull > backup.json  # download
terraform state push backup.json    # upload — last resort
```

> **Always `terraform state pull > backup.json` before any `state mv`, `state
> rm` or `state push`.** These commands have no undo. Bucket versioning gives
> you a second net, but a local copy is faster.

`state rm` is how you stop managing something without destroying it. `state mv`
is how you refactor — though `moved` blocks (chapter 11) are better, because
they are committed and reviewed rather than typed once on a laptop.

## Drift

Someone changes a resource in the console. Code and reality now disagree —
that is **drift**.

Terraform refreshes state during plan, sees the difference and offers to put it
back:

```
~ resource "aws_sagemaker_notebook_instance" "this" {
    ~ instance_type = "ml.t3.large" -> "ml.t3.medium"
```

Decide which is right: revert the drift by applying, or update the code to match
if the manual change was correct.

```bash
terraform plan -refresh-only    # show drift only, propose no changes
```

Drift is inevitable in a shared account. Some teams run a scheduled
`plan` purely to detect it — `arifa-infra` does not yet, which means drift
surfaces only when someone happens to open a pull request.

## Remote state between roots

One root can read another's outputs:

```hcl
data "terraform_remote_state" "bootstrap" {
  backend = "s3"
  config = {
    bucket = "arifa-infrastructure-tfstate-483075907105-eu-central-1"
    key    = "bootstrap/terraform.tfstate"
    region = "eu-central-1"
  }
}

resource "..." "..." {
  bucket = data.terraform_remote_state.bootstrap.outputs.state_bucket_name
}
```

Convenient, but it couples the roots and requires read access to the other
state — which contains secrets. Prefer a data source lookup by tag or name
where one exists.

## Try it

```bash
cd arifa-infra

cat envs/dev/backend.tf
cat bootstrap/backend.tf      # note it is commented out, and why

# With AWS access and after `make init ENV=dev`:
terraform -chdir=envs/dev state list
terraform -chdir=envs/dev state show module.secrets.aws_secretsmanager_secret.this[\"arifa-df-jentic-api-key\"]
```

Then read `bootstrap/state.tf` and list every protection on the bucket, with a
one-line reason for each.

## Check yourself

1. Why can Terraform not work out what it manages by querying AWS?
2. You delete `terraform.tfstate`. What happens to the resources? To the next apply?
3. Why must state be treated as a credential?
4. What does `use_lockfile = true` replace?
5. Why is `bootstrap/backend.tf` committed commented out?
6. Which protection on the state bucket is the actual recovery path?
7. Difference between `terraform state rm` and `terraform destroy`?
8. What is drift, and what are your two valid responses to it?

---

Previous: [06 — Variables, outputs and locals](06-variables-outputs-locals.md) ·
Next: [08 — Meta-arguments](08-meta-arguments.md)
