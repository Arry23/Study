# 06 — Variables, outputs and locals

Three ways to name a value. They are not interchangeable.

| Block | Direction | Set by |
|-------|-----------|--------|
| `variable` | **in** | the caller — a person, a tfvars file, or a parent module |
| `output` | **out** | the module, for its caller or for humans |
| `locals` | internal | the module itself; cannot be overridden |

## Variables

```hcl
variable "instance_type" {
  description = "ML compute instance type, for example ml.t3.medium."
  type        = string
}
```

No `default` means **required**. With a default it is optional:

```hcl
variable "volume_size" {
  description = "Size of the ML storage volume, in GB."
  type        = number
  default     = 5
}
```

Always write a `description`. It appears in error messages, in `terraform
console`, and in generated documentation. `arifa-infra` describes every
variable, including the awkward ones:

```hcl
variable "subnet_id" {
  description = "Subnet to run the notebook in, or null to run outside a VPC. Required when direct_internet_access is Disabled."
  type        = string
  default     = null
}
```

That description encodes a rule a type cannot.

### Types

```hcl
type = string
type = number
type = bool
type = list(string)
type = set(string)
type = map(string)
type = object({ name = string, size = optional(number) })
type = any                      # avoid: disables checking
```

Be as specific as you can. `any` turns off the only free correctness check you
get.

### Validation

A `validation` block rejects bad input at plan time, with your message:

```hcl
variable "direct_internet_access" {
  type    = string
  default = "Enabled"

  validation {
    condition     = contains(["Enabled", "Disabled"], var.direct_internet_access)
    error_message = "direct_internet_access must be Enabled or Disabled."
  }

  validation {
    condition     = var.direct_internet_access == "Enabled" || var.subnet_id != null
    error_message = "direct_internet_access = Disabled needs subnet_id and security_group_ids."
  }
}
```

Both are from `modules/sagemaker-notebook/variables.tf`. Note the second one
references **another variable** — that is allowed since Terraform 1.9, and it is
how you express "if A then B is required".

The payoff: a mistake fails in one second with a sentence you wrote, instead of
in ninety seconds with an AWS API error.

Validate with `can()` and a regex when the shape matters:

```hcl
validation {
  condition     = alltrue([for name in keys(var.secrets) : can(regex("^[A-Za-z0-9/_+=.@-]{1,512}$", name))])
  error_message = "Secret names use letters, digits and /_+=.@- only, up to 512 characters."
}
```

### Sensitive variables

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

Terraform then prints `(sensitive value)` instead of the value in plans and
outputs.

> **Danger.** `sensitive = true` hides it from *output*. It does **not** encrypt
> it, and the value is stored in plain text in the state file. The real answer
> is to keep secrets out of Terraform entirely — chapter 14, and see how
> `modules/secrets` manages containers without ever touching a value.

### Setting values

```bash
terraform plan -var="instance_type=ml.t3.large"      # CLI
terraform plan -var-file="dev.tfvars"                # file
export TF_VAR_instance_type=ml.t3.large              # environment
```

Precedence, lowest to highest: defaults → `TF_VAR_*` → `terraform.tfvars` →
`*.auto.tfvars` → `-var-file` → `-var`.

### `arifa-infra` deliberately uses none of them

There is no `tfvars` file anywhere. Environment values are hardcoded in
`envs/dev/main.tf`:

```hcl
locals {
  environment = "dev"
  account_id  = "483075907105"
  region      = "eu-central-1"
}
```

With the stated reason:

```hcl
# Its values live here, not in tfvars, so CI plans exactly what a laptop plans.
```

This is a real pattern, sometimes called *rooted environments*. Each environment
is a directory whose values are committed. The trade-off:

- **Cost:** some duplication between environments.
- **Benefit:** no hidden inputs. A plan cannot differ because someone forgot
  `-var-file`, and reviewing the diff shows the actual value changing.

Variables remain the right tool for **modules**. The modules in this repo are
full of them; the roots are not.

## Outputs

```hcl
output "arns" {
  description = "ARNs of the managed secrets, keyed by name."
  value       = { for name, secret in aws_secretsmanager_secret.this : name => secret.arn }
}
```

Outputs serve two audiences:

1. **A parent module** — `module.secrets.arns`
2. **Humans** — printed after `apply`, readable with `terraform output`

A module's outputs are its public API. Expose what callers need and nothing
more, because removing an output later is a breaking change.

```hcl
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true     # required, or Terraform errors
}
```

Marking an output `sensitive` is mandatory if the value came from a sensitive
source. It still lives in state in plain text.

Read them:

```bash
terraform output                 # all
terraform output arns            # one
terraform output -json           # machine-readable
terraform output -raw bucket_name   # unquoted, for scripts
```

## Locals

A named expression inside one module. Computed once, used many times, and
**cannot be set from outside**.

```hcl
locals {
  environment = "dev"
  account_id  = "483075907105"

  tags = {
    Project     = "Arifa"
    Environment = local.environment      # locals may reference locals
    ManagedBy   = "terraform"
  }
}
```

Reference as `local.name` — singular, even though the block is `locals`.

Use them for:

**Values repeated in several places.** Change `local.region` once.

**Expressions that need a name.** `bootstrap/main.tf`:

```hcl
locals {
  state_bucket_name = "arifa-infrastructure-tfstate-${local.account_id}-${local.region}"
}
```

**Distinguishing two sets of the same kind of thing.** `envs/dev/main.tf` has
two tag maps, and the names carry the meaning:

```hcl
tags          = { Project = "Arifa", Environment = "dev", ... }   # local.tags
imported_tags = { Project_Name = "ARIFA" }                        # local.imported_tags
```

`local.imported_tags` is what the resources carry in AWS today; `local.tags` is
where the team is heading. Giving both a name makes the migration reviewable.

## Choosing between them

```
Does the caller need to set it?         -> variable
Does something outside need to read it? -> output
Neither — just avoiding repetition?     -> local
```

A frequent mistake is a variable with a default that nobody ever overrides.
That is a local wearing the wrong hat: it widens the module's API for no gain.

## Documenting a module

Variables and outputs *are* the documentation. `arifa-infra` writes a README per
module by hand. Generating it is better, because hand-written docs drift:

```yaml
# .pre-commit-config.yaml
- id: terraform_docs
```

## Try it

```bash
cd arifa-infra

# every variable, with its type
grep -A3 '^variable' modules/sagemaker-notebook/variables.tf

# the locals a root uses
sed -n '/^locals/,/^}/p' envs/dev/main.tf

# evaluate an expression
terraform console
> { for k, v in { a = { d = "x" }, b = { d = "y" } } : k => v.d }
```

Then: find a variable in `modules/sagemaker-notebook` whose `description`
states a rule the `type` cannot enforce, and find the `validation` block that
does enforce it.

## Check yourself

1. When is a `variable` required rather than optional?
2. Why can a `validation` block reference another variable?
3. What does `sensitive = true` actually protect, and what does it not?
4. Why does `arifa-infra` avoid tfvars files?
5. `local.tags` vs `local.imported_tags` — what is the difference for?
6. You have a variable with a default nobody overrides. What should it be?

---

Previous: [05 — Resources and data sources](05-resources-and-data-sources.md) ·
Next: [07 — State and backends](07-state-and-backends.md)
