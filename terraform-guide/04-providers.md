# 04 — Providers

A **provider** is a plugin that knows how to talk to one API. `hashicorp/aws`
knows AWS. `hashicorp/azurerm` knows Azure. Terraform core knows neither — it
only knows how to build a dependency graph and call providers.

That is why the same language manages everything: learn HCL once, then learn
each provider's resources as you need them.

## Declaring one

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.71"
    }
  }
}
```

- `aws` — the **local name**, used as the prefix on resource types
  (`aws_s3_bucket`) and in `provider` blocks
- `source` — where to download it, `<namespace>/<type>` on the public registry
- `version` — a constraint, not an exact version

## Configuring one

`required_providers` says *which plugin*. A `provider` block says *how to use it*:

```hcl
provider "aws" {
  region              = local.region
  allowed_account_ids = [local.account_id]

  default_tags {
    tags = local.tags
  }

  ignore_tags {
    keys = ["owner", "Owner"]
  }
}
```

That is `arifa-infra/envs/dev/providers.tf`, and three of those settings are
worth copying into any AWS project.

**`allowed_account_ids`** — Terraform refuses to run if the credentials resolve
to a different account. The cheapest possible guard against applying dev code to
production.

**`default_tags`** — every resource this provider creates gets these tags
automatically, so you never repeat them:

```hcl
tags = {
  Project      = "Arifa"
  Environment  = "dev"
  ManagedBy    = "terraform"
  Repository   = "roche-innersource/arifa-infrastructure"
}
```

**`ignore_tags`** — tells Terraform not to manage certain tag keys. Here the
platform team's guardrails deny changes to `owner`, so Terraform must not try.
Without this, every plan would show a tag change it can never apply.

## Version constraints

| Constraint | Allows | Use for |
|------------|--------|---------|
| `= 6.71.0` | exactly 6.71.0 | rarely; the lock file does this better |
| `>= 6.0` | 6.0 and anything newer | modules |
| `~> 6.71` | `>= 6.71, < 7.0` | root modules — patch and minor updates |
| `~> 6.71.0` | `>= 6.71.0, < 6.72.0` | patch updates only |

The rule: **`~>` moves the rightmost component you specified.** `~> 6.71` lets
the minor move; `~> 6.71.0` pins the minor and lets only the patch move.

`arifa-infra` applies a deliberate split:

```hcl
# envs/dev/versions.tf — a root module, pins hard
required_version = "~> 1.17.0"
version          = "~> 6.71"

# modules/secrets/versions.tf — a module, states a minimum
required_version = ">= 1.11.0"
version          = ">= 6.0"
```

The comment in the module explains it: *"Modules state a minimum only; the
calling root pins exact versions."* A module that pins hard is painful to reuse,
because any caller on a different version is locked out. The root is where you
decide.

## The lock file

Constraints allow a range. `.terraform.lock.hcl` records what was actually
chosen, so everyone gets the same plugin:

```hcl
provider "registry.terraform.io/hashicorp/aws" {
  version     = "6.71.0"
  constraints = "~> 6.71"
  hashes      = [
    "h1:...",
    "zh:...",
  ]
}
```

**Commit this file.** It is the Terraform equivalent of `package-lock.json`.

| Command | Effect |
|---------|--------|
| `terraform init` | Creates or updates the lock file |
| `terraform init -lockfile=readonly` | **Fails** if the lock would change |
| `terraform init -upgrade` | Deliberately moves to newer allowed versions |

`arifa-infra` uses `-lockfile=readonly` everywhere — in the Makefile, in CI, in
`scripts/check.sh`. That turns "a provider upgrade" into a deliberate, reviewed
commit rather than something that happens because a colleague ran `init` on a
Tuesday.

### Hashes are per-platform

A lock file records hashes for the platforms it has seen. A lock written on an
Apple Silicon Mac has no Linux hashes, so CI's `-lockfile=readonly` fails.

The fix is to generate both up front:

```makefile
PLATFORMS := -platform=darwin_arm64 -platform=linux_amd64

lock:
	@for root in $(ROOTS); do \
	  terraform -chdir=$$root init -backend=false -input=false >/dev/null && \
	  terraform -chdir=$$root providers lock $(PLATFORMS) || exit 1; \
	done
```

That is `make lock`. Run it whenever you change a provider version. Add
`-platform=windows_amd64` if Windows contributors join.

## Multiple providers: aliases

Sometimes you need the same provider configured two ways — two regions, two
accounts, or two different behaviours. That is an **alias**:

```hcl
provider "aws" {
  region = "eu-central-1"
  default_tags { tags = local.tags }
}

provider "aws" {
  alias  = "imported"
  region = "eu-central-1"
  # deliberately no default_tags
}
```

The one without `alias` is the default. Pass the other explicitly:

```hcl
module "secrets" {
  source    = "../../modules/secrets"
  providers = { aws = aws.imported }
  # ...
}
```

### Why `arifa-infra` does this

This is the most interesting thing in the repository, and it is worth
understanding because the reasoning generalises.

The team is importing resources that already exist in AWS. An import should
change **nothing** — it only teaches Terraform that a resource exists.

But the default provider adds `default_tags` to everything it manages. So every
import would also become a tag change, and the "this changes nothing" plan would
instead be "4 to import, 3 to change". Reviewers could no longer tell a safe
import from a real modification.

The alias has no `default_tags`, so imports through it are genuine no-ops.
Adding the standard tags becomes a separate, reviewable pull request later.

The comment says exactly that:

```hcl
# Imported resources keep their current tags. With default tags, every import
# would also be a tag change, so imports use this provider until the team
# decides to add the standard tags as a separate, reviewed change.
```

**The generalisable lesson:** when a mechanical change and a semantic change
would land in the same plan, split them so a human can review each one.

### Aliases in modules

A module that needs an alias declares it:

```hcl
# inside the module
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      configuration_aliases = [aws.replica]
    }
  }
}
```

A module should **never** contain its own `provider` block. Providers are
configured by the root and passed in. A module with a hardcoded region cannot be
reused in another region — which is why
`modules/sagemaker-notebook/main.tf` says:

```hcl
# A SageMaker notebook instance. Region-agnostic: it runs in the region of the
# provider the caller passes in.
```

## Where providers are installed

`terraform init` downloads them into `.terraform/providers/` in that directory.
That folder is **gitignored** — it is a cache, often hundreds of megabytes.

This is why you must run `init` once per root per clone, and why this error
appears if you skip it:

```
Error: there is no package for registry.terraform.io/hashicorp/aws 6.71.0
cached in .terraform\providers
```

The fix is `terraform init`, not a code change.

## Try it

```bash
cd arifa-infra

# What does this root require?
cat envs/dev/versions.tf

# What got locked?
grep -A2 'provider "registry' envs/dev/.terraform.lock.hcl

# Which providers does each root use?
terraform -chdir=envs/dev providers
```

Then compare `modules/secrets/versions.tf` with `envs/dev/versions.tf` and
articulate why the constraints differ.

## Check yourself

1. What versions does `~> 6.71` allow? And `~> 6.71.0`?
2. Why is `.terraform.lock.hcl` committed but `.terraform/` is not?
3. What does `-lockfile=readonly` protect against?
4. Why do `arifa-infra`'s modules use `>= 6.0` instead of `~> 6.71`?
5. Explain the `aws.imported` alias to someone reviewing the pull request.
6. Why should a module never contain a `provider` block?

---

Previous: [03 — The HCL language](03-hcl-language.md) ·
Next: [05 — Resources and data sources](05-resources-and-data-sources.md)
