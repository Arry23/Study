# 17 — Cheat sheet

One page. Keep it open.

## Core commands

```bash
terraform init                    # providers, modules, backend
terraform init -lockfile=readonly # fail if the lock would change
terraform init -backend=false     # no backend — for validate
terraform init -upgrade           # move to newer allowed versions
terraform init -migrate-state     # move state to a new backend
terraform init -reconfigure       # change backend, do not migrate

terraform plan                    # what would change
terraform plan -out=tfplan        # save it
terraform plan -refresh-only      # drift only
terraform plan -no-color          # for logs
terraform plan -generate-config-out=generated.tf   # draft config for imports

terraform apply                   # plan, confirm, apply
terraform apply tfplan            # apply a saved plan
terraform destroy                 # destroy everything in this root

terraform fmt -recursive          # rewrite
terraform fmt -check -recursive   # fail if unformatted
terraform validate                # syntax and references
terraform console                 # evaluate expressions
terraform output [-raw|-json]     # read outputs
terraform show [-json]            # state or a saved plan
terraform graph                   # dependency graph
terraform providers               # what each module requires
terraform test                    # run *.tftest.hcl

terraform -chdir=envs/dev plan    # run as if from that directory
```

## State

```bash
terraform state list
terraform state show <address>
terraform state mv <from> <to>
terraform state rm <address>
terraform state pull > backup.json      # ALWAYS before mv/rm/push
terraform state push backup.json
terraform force-unlock <LOCK_ID>
```

## arifa-infra make targets

```bash
make help                 # list targets
make check                # all four gates, no AWS needed
make fmt                  # rewrite files
make fmt-check            # fail if unformatted
make validate
make lint
make scan
make lock                 # provider hashes for macOS + Linux

make bootstrap-plan
make bootstrap-apply

make init     ENV=dev
make plan     ENV=dev
make generate ENV=dev     # draft config for pending imports
make apply    ENV=dev
```

## Block types

```hcl
terraform { required_version = "~> 1.17.0" }
provider "aws" { region = "eu-central-1" }
resource "aws_s3_bucket" "state" { }
data "aws_caller_identity" "current" { }
variable "name" { type = string }
output "arn" { value = aws_s3_bucket.state.arn }
locals { region = "eu-central-1" }
module "secrets" { source = "../../modules/secrets" }
import { to = ...  id = "..." }
moved { from = ... to = ... }
removed { from = ...  lifecycle { destroy = false } }
```

## References

```hcl
var.name                       # variable
local.region                   # local
aws_s3_bucket.state.arn        # resource attribute
aws_s3_bucket.state[0].arn     # with count
aws_s3_bucket.state["k"].arn   # with for_each
data.aws_region.current.name   # data source
module.secrets.arns            # module output
each.key / each.value          # inside for_each
count.index                    # inside count
path.module / path.root / path.cwd
terraform.workspace
```

## Version constraints

| Constraint | Allows |
|------------|--------|
| `= 1.2.3` | exactly 1.2.3 |
| `>= 1.2.0` | 1.2.0 and up |
| `~> 1.2` | `>= 1.2, < 2.0` |
| `~> 1.2.0` | `>= 1.2.0, < 1.3.0` |
| `>= 1.2, < 2.0` | explicit range |

`~>` moves the rightmost component you wrote.

## Meta-arguments

```hcl
count    = var.enabled ? 1 : 0        # toggle
for_each = var.secrets                # collection — prefer this
depends_on = [aws_s3_bucket_versioning.state]
provider   = aws.imported             # on a resource
providers  = { aws = aws.imported }   # on a module

lifecycle {
  prevent_destroy       = true
  create_before_destroy = true
  ignore_changes        = [tags["LastScanned"]]
  replace_triggered_by  = [aws_launch_template.this.latest_version]

  precondition {
    condition     = var.size > 0
    error_message = "size must be positive."
  }
}
```

## Types

```hcl
string  number  bool  null
list(string)  set(string)  map(string)
object({ name = string, size = optional(number) })
tuple([string, number])
any
```

## Common functions

```hcl
merge(a, b)              lookup(m, k, default)     coalesce(a, b)
try(expr, fallback)      can(expr)
length(x)                keys(m)       values(m)
join("-", list)          split(",", s)
upper(s)  lower(s)  trimspace(s)  replace(s, a, b)
contains(list, x)        toset(list)   tolist(set)
jsonencode(o)            jsondecode(s)
file("${path.module}/x") templatefile(path, vars)
alltrue(list)            anytrue(list)
cidrsubnet(cidr, n, i)
```

## Loops in expressions

```hcl
[for s in list : upper(s)]                 # list
[for s in list : s if length(s) > 3]       # filtered
{ for k, v in map : k => v.field }         # map
aws_instance.web[*].id                     # splat
```

## Validation

```hcl
variable "mode" {
  type = string

  validation {
    condition     = contains(["Enabled", "Disabled"], var.mode)
    error_message = "mode must be Enabled or Disabled."
  }
}
```

## Reading a plan

| Symbol | Meaning |
|--------|---------|
| `+` | create |
| `-` | destroy |
| `~` | change in place |
| `-/+` | **destroy and recreate** |

Always check:
1. `Plan: X to add, Y to change, Z to destroy` — is `Z` zero?
2. Search for `# forces replacement`
3. Does it match the expected plan in the runbook?

## Environment variables

```bash
TF_LOG=DEBUG            # TRACE DEBUG INFO WARN ERROR
TF_LOG_PATH=./tf.log
TF_IN_AUTOMATION=true   # suppress hints (CI)
TF_INPUT=0              # never prompt
TF_VAR_name=value       # set a variable
AWS_PROFILE=arifa
```

## AWS, day to day

```bash
aws sso login --profile arifa
aws sts get-caller-identity --profile arifa
aws secretsmanager describe-secret --profile arifa --region eu-central-1 \
    --secret-id <name> --query ARN --output text
aws iam list-open-id-connect-providers --profile arifa
```

## Safety rules

1. Read every plan. Check `to destroy` and `# forces replacement`.
2. `terraform state pull > backup.json` before any state surgery.
3. Never commit `*.tfstate`. Never commit credentials.
4. Never `-auto-approve` interactively.
5. `-target` is for debugging, never routine.
6. Never `force-unlock` unless you know nothing is running.
7. Never widen `required_version` to dodge an error.
8. Never `ignore_changes` a diff you do not understand.

## File layout

```
main.tf        resources and module calls
variables.tf   inputs
outputs.tf     outputs
providers.tf   provider configuration
versions.tf    required_version, required_providers
backend.tf     where state lives
imports.tf     pending import blocks
```

## Gitignore essentials

```
.terraform/
*.tfstate
*.tfstate.*
*.tfvars          # unless deliberately committed
generated.tf
backend_override.tf
```

Commit `.terraform.lock.hcl` for roots.

---

Previous: [16 — Troubleshooting](16-troubleshooting.md) ·
Next: [18 — Hands-on labs](18-hands-on-labs.md) ·
Back to the [index](README.md)
