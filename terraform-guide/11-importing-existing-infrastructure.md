# 11 — Importing existing infrastructure

Most Terraform work does not start on a blank account. Things already exist,
created by hand, and you need Terraform to adopt them **without changing them**.

That is exactly where `arifa-infra` is right now: one SageMaker notebook and
three secrets, all created in the console, being brought under management.

## The goal

A successful import changes **nothing** in AWS. It only teaches Terraform that
a resource exists and which code block owns it. The state file grows; reality
does not move.

If your import plan shows modifications you did not intend, your code does not
match AWS. Fix the code, not the resource.

## `import` blocks

The modern way. Declarative, committed, reviewed:

```hcl
import {
  to = module.notebook.aws_sagemaker_notebook_instance.this
  id = "arifa-df-sagemaker-nb-dev"
}
```

- `to` — the address the resource should occupy
- `id` — how the provider identifies it, which varies per resource type

For `for_each` resources, the key is part of the address:

```hcl
import {
  to = module.secrets.aws_secretsmanager_secret.this["arifa-df-jentic-api-key"]
  id = "arn:aws:secretsmanager:eu-central-1:483075907105:secret:arifa-df-jentic-api-key-ySR0cr"
}
```

Note the ID formats differ. The notebook imports by **name**; the secret imports
by **ARN**, because Secrets Manager ARNs carry a random six-character suffix.
`envs/dev/imports.tf` even records how to find it:

```hcl
# Secrets import by ARN, which ends in a random 6-character suffix. To look one up
# (metadata only, never the value):
#   aws secretsmanager describe-secret --profile arifa --region eu-central-1 --secret-id <name> --query ARN --output text
```

Check the provider documentation for each resource type's import ID format.

### The old CLI way

```bash
terraform import aws_s3_bucket.state my-bucket-name
```

Still works, but it mutates state immediately from someone's laptop with no
review and no record. Prefer `import` blocks: they appear in the pull request,
run through `plan` first, and leave a git trail.

## Generating configuration

You cannot import into a block that does not exist. Writing that block by hand
means guessing every attribute. Instead, let Terraform write a draft:

```bash
terraform plan -generate-config-out=generated.tf
```

Or via the Makefile:

```bash
make generate ENV=dev
```

Terraform reads each `import` block, fetches the real resource, and writes
matching HCL into `generated.tf`.

**Treat that file as a draft, not an answer.** It includes every attribute,
including defaults and read-only fields. Your job is to:

1. Move the meaningful settings into your module call
2. Drop the noise
3. Delete `generated.tf`

It is listed in `.gitignore` for that reason — it is scaffolding.

## The procedure

```bash
# 1. Write the module call with what you know
#    (envs/dev/main.tf)

# 2. Add the import block
#    (envs/dev/imports.tf)

# 3. See the real settings
make generate ENV=dev

# 4. Fold them into the module call, delete generated.tf

# 5. Plan until it is clean
make plan ENV=dev

# 6. Apply
make apply ENV=dev

# 7. Confirm
make plan ENV=dev        # expect: No changes.

# 8. Delete the applied import blocks
```

Step 8 matters. An `import` block is a one-time instruction; once applied it is
dead weight. `envs/dev/imports.tf` says so at the top:

```hcl
# Pending import blocks only. Delete each block once its import has been applied;
# git history keeps the record.
```

## Reading an import plan

```
Plan: 4 to import, 0 to add, 3 to change, 0 to destroy.
```

Break it down:

- **4 to import** — correct, that is the point
- **0 to add** — good; an "add" means Terraform did not match your block to the
  real resource
- **3 to change** — needs justification
- **0 to destroy** — anything else, stop immediately

The README justifies the three changes:

> Each secret shows only `+ force_overwrite_replica_secret = false` and
> `+ recovery_window_in_days = 30`: Terraform-only settings that AWS never
> returns, so applying them writes state only.

Those two attributes exist in Terraform's schema but not in the AWS API
response, so the first plan after import always proposes them. Applying writes
state and calls nothing.

The temptation is to silence them:

```hcl
# Don't.
lifecycle {
  ignore_changes = [recovery_window_in_days]
}
```

The repo explicitly forbids it:

```hcl
# After an import, the first plan adds recovery_window_in_days = 30 and
# force_overwrite_replica_secret = false to state; AWS never returns them. Apply it,
# and never ignore_changes them.
```

Apply once and the diff is gone forever. `ignore_changes` would hide the
attribute permanently, including real future drift.

## Keeping the import a true no-op

The subtlest import trap is **provider `default_tags`**. If your provider adds
tags automatically, every import is also a tag change, and the plan no longer
says "nothing happens".

`arifa-infra` solves it with a second provider that adds no tags:

```hcl
provider "aws" {
  alias  = "imported"
  region = local.region
  # no default_tags
}

module "notebook" {
  source    = "../../modules/sagemaker-notebook"
  providers = { aws = aws.imported }
  tags      = local.imported_tags   # exactly what is in AWS today
}
```

`local.imported_tags` is the tags the resources carry right now:

```hcl
imported_tags = {
  Project_Name = "ARIFA"
}
```

So the import changes nothing, and adopting the standard tags becomes a
separate pull request with its own reviewable plan.

Descriptions need the same care. The comment in `envs/dev/main.tf`:

```hcl
# The descriptions are the values in AWS, byte for byte (sanity plan, 2026-11-02).
# Rewording one changes the secret in AWS.
```

During an import you copy reality, including wording you might not like.
Improve it afterwards, deliberately.

## `moved` blocks — refactoring without destroying

Rename a resource and Terraform sees one address disappear and another appear:
destroy and create. A `moved` block says they are the same thing:

```hcl
moved {
  from = aws_s3_bucket.bucket
  to   = aws_s3_bucket.state
}

moved {
  from = module.old_name
  to   = module.new_name
}

moved {
  from = aws_instance.web[0]
  to   = aws_instance.web["primary"]   # count -> for_each
}
```

That last form is how you migrate from `count` to `for_each` safely — otherwise
every instance is destroyed and recreated.

Apply, confirm `No changes.`, then delete the `moved` block.

Better than `terraform state mv` for the same reason `import` blocks beat
`terraform import`: it is in the pull request.

## `removed` blocks — forgetting without destroying

To stop managing something but leave it in AWS:

```hcl
removed {
  from = aws_s3_bucket.legacy

  lifecycle {
    destroy = false
  }
}
```

`destroy = false` is the whole point: remove it from state, leave it running.
The declarative equivalent of `terraform state rm`.

## Common problems

**"Resource already managed"** — it is already in state. `terraform state list`
to confirm.

**"Cannot import non-existent remote object"** — wrong ID or wrong region.

**The plan wants to replace the imported resource.** Your config disagrees with
reality on an attribute that forces replacement. Compare carefully with
`generated.tf`. Do not apply.

**The plan shows unexpected changes.** Your code does not match AWS. Fix the
code.

## A safety net before you have a backend

You can sanity-check an import plan without touching the shared state. Create a
temporary local backend override:

```hcl
# envs/dev/backend_override.tf — gitignored
terraform {
  backend "local" {}
}
```

Then `init -reconfigure` and plan. State goes to your laptop, not the bucket.
**Delete the file and re-init before doing anything real**, or your apply will
write state to the wrong place. `arifa-infra`'s README documents this as the
"optional sanity plan".

## Try it

```bash
cd arifa-infra

cat envs/dev/imports.tf          # four pending imports
cat docs/importing.md            # the repo's own, longer guide

# with AWS access:
make generate ENV=dev            # writes envs/dev/generated.tf
# read it, then delete it
```

## Check yourself

1. What should a correct import plan change in AWS?
2. Why do the notebook and the secrets use different ID formats?
3. What is `generated.tf` for, and why is it gitignored?
4. Why do the three secrets show a change on first plan? Why not `ignore_changes`?
5. How does the `aws.imported` provider keep imports clean?
6. When do you delete an `import` block?
7. Difference between `moved` and `removed`?
8. Why prefer `import` blocks over `terraform import`?

---

Previous: [10 — The workflow and the CLI](10-workflow-and-cli.md) ·
Next: [12 — Quality gates and testing](12-quality-gates-and-testing.md)
