# 03 — The HCL language

HCL (HashiCorp Configuration Language) has a small grammar. Learn the block
shape and the expression rules and you can read any Terraform file.

## Everything is a block

```hcl
block_type "label_one" "label_two" {
  argument = value

  nested_block {
    argument = value
  }
}
```

A real one, from `arifa-infra/modules/sagemaker-notebook/main.tf`:

```hcl
resource "aws_sagemaker_notebook_instance" "this" {
  name          = var.name
  instance_type = var.instance_type

  instance_metadata_service_configuration {
    minimum_instance_metadata_service_version = "2"
  }

  tags = var.tags
}
```

- `resource` — the block type
- `"aws_sagemaker_notebook_instance"` — which kind of thing
- `"this"` — your name for it, used to reference it elsewhere
- `instance_metadata_service_configuration` — a nested block

The block types you will meet:

| Block | Purpose | Chapter |
|-------|---------|---------|
| `terraform` | Version constraints and the backend | 04, 07 |
| `provider` | How to talk to AWS, Azure, etc. | 04 |
| `resource` | Something Terraform creates and owns | 05 |
| `data` | Something Terraform only reads | 05 |
| `variable` | An input | 06 |
| `output` | A return value | 06 |
| `locals` | Named intermediate values | 06 |
| `module` | A call to reusable code | 09 |
| `import` / `moved` / `removed` | State surgery | 11 |

## File layout

Terraform reads **every `.tf` file in one directory** and treats them as a
single configuration. File names carry no meaning to Terraform — they are for
humans. The usual convention, which `arifa-infra` follows:

```
envs/dev/
├── main.tf        the actual resources and module calls
├── variables.tf   inputs
├── outputs.tf     outputs
├── providers.tf   provider configuration
├── versions.tf    required_version and required_providers
├── backend.tf     where state lives
└── imports.tf     pending import blocks
```

Order does not matter either. Terraform builds a dependency graph from
references, not from line numbers.

## Types

**Primitives**

```hcl
name    = "arifa-df-sagemaker-nb-dev"   # string
size    = 5                              # number
enabled = true                           # bool
key     = null                           # absent — not the same as "" or false
```

`null` means *not set*, so the provider uses its own default. In
`modules/sagemaker-notebook/variables.tf`:

```hcl
variable "kms_key_id" {
  description = "KMS key that encrypts the storage volume, or null for the AWS managed key."
  type        = string
  default     = null
}
```

**Collections**

```hcl
list(string)   ["a", "b", "a"]           ordered, duplicates allowed
set(string)    ["a", "b"]                unordered, unique
map(string)    { env = "dev", tier = "1" }   string keys
```

**Structural**

```hcl
object({ name = string, size = number })
tuple([string, number, bool])
```

`modules/secrets/variables.tf` combines a map with an object:

```hcl
variable "secrets" {
  type = map(object({
    description = optional(string)
    kms_key_id  = optional(string)
  }))
}
```

`optional(...)` means the caller may omit that attribute; it arrives as `null`.

## Strings

```hcl
plain       = "hello"
interpolated = "arn:aws:iam::${local.account_id}:role/service-role/Foo"
```

`${...}` embeds an expression. From `envs/dev/main.tf`:

```hcl
role_arn = "arn:aws:iam::${local.account_id}:role/service-role/AWSNeptuneNotebookRole-1782226041594"
```

Multi-line uses a heredoc:

```hcl
policy = <<-EOT
  {
    "Version": "2012-10-17"
  }
EOT
```

`<<-` strips the leading indentation. Plain `<<` keeps it.

> Do not build JSON by hand in a heredoc when a structured alternative exists.
> `arifa-infra` builds IAM policies with `data "aws_iam_policy_document"`,
> which is validated at plan time; a heredoc typo is only caught by AWS.

## Referencing things

```hcl
var.name                       # an input variable
local.region                   # a local value
aws_s3_bucket.state.arn        # an attribute of a resource
data.aws_caller_identity.current.account_id   # an attribute of a data source
module.secrets.arns            # an output of a module
each.key / each.value          # inside for_each
count.index                    # inside count
```

Referencing a resource creates a dependency. Terraform sees that
`aws_s3_bucket_versioning.state` uses `aws_s3_bucket.state.id` and orders the
bucket first, with no instruction from you.

## Operators

```hcl
a == b    a != b    a < b    a >= b
a && b    a || b    !a
a + b     a - b     a * b    a / b    a % b
condition ? if_true : if_false
```

The ternary is how you do conditionals:

```hcl
count = var.enable_github_ci ? 1 : 0
```

That line in `bootstrap/github_oidc.tf` is the whole feature flag: one when
enabled, zero resources when not.

## Functions

Terraform ships ~100 built-ins. There is no way to define your own.

```hcl
merge({ a = 1 }, { b = 2 })            # { a = 1, b = 2 }
lookup(map, "key", "fallback")
coalesce(null, "", "first non-empty")  # "first non-empty"
try(risky.expression, "fallback")
length([1, 2, 3])                      # 3
join("-", ["a", "b"])                  # "a-b"
split(",", "a,b")                      # ["a", "b"]
upper / lower / trimspace / replace
contains(["a", "b"], "a")              # true
keys(map) / values(map)
toset(list) / tolist(set) / tostring(x)
jsonencode(obj) / jsondecode(str)
file("${path.module}/policy.json")
can(expression)                        # true if it evaluates without error
alltrue(list) / anytrue(list)
```

`can()` and `alltrue()` together make validation rules readable —
`modules/secrets/variables.tf`:

```hcl
validation {
  condition     = alltrue([for name in keys(var.secrets) : can(regex("^[A-Za-z0-9/_+=.@-]{1,512}$", name))])
  error_message = "Secret names use letters, digits and /_+=.@- only, up to 512 characters."
}
```

Test any expression interactively:

```bash
terraform console
> merge({a=1}, {b=2})
> upper("dev")
```

## Loops in expressions

**`for` over a list**

```hcl
[for s in var.names : upper(s)]                 # list
[for s in var.names : s if length(s) > 3]       # with a filter
```

**`for` over a map**

```hcl
{ for k, v in var.secrets : k => v.description }
```

The `[...]` or `{...}` wrapper decides whether you get a list or a map.

**Splat** — shorthand for pulling one attribute off every instance:

```hcl
aws_instance.web[*].id
```

`modules/secrets/outputs.tf` uses a `for` to build a map of ARNs:

```hcl
output "arns" {
  value = { for name, secret in aws_secretsmanager_secret.this : name => secret.arn }
}
```

These are **expression** loops. Creating many resources is a different thing —
`count` and `for_each`, chapter 08.

## Comments

```hcl
# preferred
// also works
/* block
   comment */
```

`arifa-infra` comments *why*, not *what*. Compare:

```hcl
# Bad: says what the next line already says
# Set prevent_destroy to true
prevent_destroy = true

# Good: says why, which the code cannot
# Losing this bucket loses track of everything Terraform manages.
prevent_destroy = true
```

## Formatting

There is one canonical style and a tool that enforces it:

```bash
terraform fmt -recursive          # rewrite files
terraform fmt -check -recursive   # fail if anything is unformatted (CI)
```

Never argue about alignment. Run `fmt`.

## Try it

```bash
cd arifa-infra
terraform console          # Ctrl-D to exit
```

In the console:

```hcl
> merge({ Project = "Arifa" }, { Environment = "dev" })
> [for s in ["a", "bb", "ccc"] : upper(s) if length(s) > 1]
> can(regex("^[a-z]+$", "abc"))
> "arn:aws:iam::${123456789012}:role/x"
```

Then open `envs/dev/main.tf` and find one example each of: a local value, a
string interpolation, a map literal, and a module call.

## Check yourself

1. What is the difference between `null`, `""` and `false`?
2. `list(string)` vs `set(string)` — when does it matter?
3. What does `optional(string)` do to a caller's object?
4. Rewrite `[for k, v in m : k]` using a built-in function.
5. Why does `arifa-infra` build IAM policies with a data source instead of a heredoc?

---

Previous: [02 — Install and set up](02-install-and-setup.md) ·
Next: [04 — Providers](04-providers.md)
