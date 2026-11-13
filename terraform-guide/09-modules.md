# 09 — Modules

A module is a directory of `.tf` files. That is the whole definition.

You are already using modules. The directory you run `terraform apply` in is the
**root module**. When it calls another directory, that one is a **child module**.

## Calling one

```hcl
module "secrets" {
  source = "../../modules/secrets"

  providers = { aws = aws.imported }
  tags      = local.imported_tags

  secrets = {
    "arifa-df-jentic-api-key"  = { description = "Jentic API Key" }
    "arifa-df-galileo-api-key" = { description = "Galileo Lite API key for ARIFA" }
  }
}
```

- `module` — the block type
- `"secrets"` — your name for this call
- `source` — where the code is; the only required argument
- everything else — values for the module's `variable` blocks

Read its outputs as `module.secrets.arns`.

Call the same module twice with different names and you get two independent
sets of resources.

## Anatomy

```
modules/secrets/
├── main.tf        resources
├── variables.tf   inputs  (the API)
├── outputs.tf     outputs (the API)
├── versions.tf    provider and Terraform requirements
└── README.md      documentation
```

`arifa-infra`'s `secrets` module in full — 19 lines of resource code:

```hcl
# modules/secrets/main.tf
resource "aws_secretsmanager_secret" "this" {
  for_each = var.secrets

  name        = each.key
  description = each.value.description
  kms_key_id  = each.value.kms_key_id

  tags = var.tags
}
```

```hcl
# modules/secrets/variables.tf
variable "secrets" {
  description = "Secrets to manage, keyed by secret name. Values are never set here."
  type = map(object({
    description = optional(string)
    kms_key_id  = optional(string)
  }))

  validation {
    condition     = alltrue([for name in keys(var.secrets) : can(regex("^[A-Za-z0-9/_+=.@-]{1,512}$", name))])
    error_message = "Secret names use letters, digits and /_+=.@- only, up to 512 characters."
  }
}

variable "tags" {
  description = "Tags for every secret, merged with the provider's default tags."
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/secrets/outputs.tf
output "arns" {
  value = { for name, secret in aws_secretsmanager_secret.this : name => secret.arn }
}
```

Small, single-purpose, no surprises. That is what a good module looks like.

## Sources

```hcl
source = "../../modules/secrets"                    # local path
source = "terraform-aws-modules/vpc/aws"            # public registry
source = "app.terraform.io/myorg/vpc/aws"           # private registry
source = "git::https://github.com/org/repo.git//modules/vpc?ref=v1.2.0"
source = "git::ssh://git@github.com/org/repo.git?ref=v1.2.0"
```

Local paths start with `./` or `../`. Anything else is treated as a registry
address.

**Always pin a remote source.** `?ref=v1.2.0` for git, `version = "5.1.0"` for
the registry. An unpinned module changes under you without a commit:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1"
}
```

`version` works only for registry sources. Git sources pin via `?ref=`.

## Version constraints inside a module

From chapter 04, but it matters most here:

```hcl
# modules/secrets/versions.tf
terraform {
  # Modules state a minimum only; the calling root pins exact versions.
  required_version = ">= 1.11.0"

  required_providers {
    aws = { source = "hashicorp/aws", version = ">= 6.0" }
  }
}
```

A module that pins `~> 6.71` is unusable by a caller on 6.80. State the minimum
you need and let the root decide. The root is where `.terraform.lock.hcl` lives,
so the root is where exactness belongs.

## Providers are passed in, never declared

A module must not contain a `provider` block. It declares what it needs and the
caller supplies it.

```hcl
# modules/sagemaker-notebook/main.tf
# A SageMaker notebook instance. Region-agnostic: it runs in the region of the
# provider the caller passes in.
```

If a module hardcoded `region = "eu-central-1"`, it could never be used
anywhere else. With the provider passed in, the same code serves every region
and account.

If a module needs *two* configurations, it declares aliases:

```hcl
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      configuration_aliases = [aws.primary, aws.replica]
    }
  }
}
```

and the caller supplies both:

```hcl
providers = {
  aws.primary = aws
  aws.replica = aws.us_east
}
```

## Designing the interface

The variables and outputs are a contract. Changing them breaks callers.

**Expose what varies; hard-code what does not.** The notebook module takes
`instance_type` as a variable because it differs per environment, but hardcodes
IMDSv2 because it should never vary:

```hcl
instance_metadata_service_configuration {
  minimum_instance_metadata_service_version = "2"
}
```

A variable for that would invite someone to turn it off.

**Default to the safe value.** `direct_internet_access` defaults to `"Enabled"`
here only because it matches what exists today during an import; the roadmap
moves it to `Disabled`. In new code, default to the restrictive option.

**Validate at the boundary.** The module cannot control its callers, so it
checks:

```hcl
validation {
  condition     = var.direct_internet_access == "Enabled" || var.subnet_id != null
  error_message = "direct_internet_access = Disabled needs subnet_id and security_group_ids."
}
```

**Never create what another team owns.** Take IDs as inputs:

```hcl
# Networking belongs to the platform team: pass existing IDs in, never create them here.
subnet_id       = var.subnet_id
security_groups = var.security_group_ids
```

## When to make a module

**Do** when the same shape is used in several places, when you want to enforce
a standard, or when a concept deserves a name.

**Do not** wrap a single resource in a module that only forwards variables. That
adds a layer and buys nothing:

```hcl
# pointless
module "bucket" {
  source = "./modules/bucket"
  name   = var.name
}
```

A good test: can you describe the module in one sentence without "and"?
*"Secrets Manager containers, without ever managing a value"* — yes.

## Nesting

Modules can call modules. Keep it shallow — two levels is usually plenty.
Deeper, and tracing a value from root to resource becomes archaeology, and
every layer has to re-declare and forward variables.

## Refactoring safely

Rename a module call and Terraform sees the old address gone and a new one
appear: destroy and create. Avoid that with a `moved` block:

```hcl
moved {
  from = module.old_name
  to   = module.new_name
}
```

Committed, reviewed, and applied like anything else — far better than
`terraform state mv` on a laptop. Chapter 11.

## Testing

```bash
cd modules/secrets
terraform init -backend=false
terraform validate
```

`-backend=false` because a module has no state of its own.

`arifa-infra` validates every module separately in `scripts/check.sh`:

```bash
for module in modules/*/; do
  terraform -chdir="$module" init -backend=false -input=false >/dev/null
  terraform -chdir="$module" validate
done
```

with the reasoning in a comment: *"Modules are validated on their own too, as
they will be once published. Their lock files are throwaway (gitignored); the
calling roots pin versions."*

Terraform also has a native test framework — `.tftest.hcl` files run by
`terraform test`. Chapter 12.

## Documenting

Variables and outputs are the API, so document them. Hand-written READMEs drift;
generate them:

```yaml
# .pre-commit-config.yaml
- id: terraform_docs
```

`arifa-infra` writes them by hand today, which is the main drift risk in those
two modules.

## Try it

```bash
cd arifa-infra

cat modules/secrets/*.tf        # the whole module, ~50 lines

# how it is called
sed -n '/^module "secrets"/,/^}/p' envs/dev/main.tf

# validate it standalone
terraform -chdir=modules/secrets init -backend=false
terraform -chdir=modules/secrets validate
```

Then: list every input `modules/sagemaker-notebook` accepts and decide, for
each, whether it should have been a variable at all.

## Check yourself

1. What is the difference between a root module and a child module?
2. Why do modules state `>= 6.0` while roots state `~> 6.71`?
3. Why must a module never contain a `provider` block?
4. Why is IMDSv2 hardcoded in the notebook module instead of being a variable?
5. When is wrapping a single resource in a module a mistake?
6. How do you rename a module call without destroying anything?
7. Why does `scripts/check.sh` validate modules separately from roots?

---

Previous: [08 — Meta-arguments](08-meta-arguments.md) ·
Next: [10 — The workflow and the CLI](10-workflow-and-cli.md)
