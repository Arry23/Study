# 08 — Meta-arguments

Meta-arguments work on **any** resource, data source or module, regardless of
provider. There are five.

| Meta-argument | Purpose |
|---------------|---------|
| `count` | Create N copies, or toggle one on and off |
| `for_each` | Create one per map key or set member |
| `depends_on` | Force ordering a reference cannot express |
| `lifecycle` | Change how Terraform creates, replaces and destroys |
| `provider` | Choose a non-default provider configuration |

## `count`

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-123"
  instance_type = "t3.micro"
  tags          = { Name = "web-${count.index}" }
}
```

Creates `aws_instance.web[0]`, `[1]`, `[2]`. Inside the block, `count.index` is
the position.

### The real use: a feature flag

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  count = var.enable_github_ci ? 1 : 0
  # ...
}
```

One when enabled, zero when not. That line in `bootstrap/github_oidc.tf` is the
entire on/off switch for CI, and it is the most common use of `count` in
production code.

Everything that depends on it needs the index:

```hcl
resource "aws_iam_role" "github_plan" {
  count              = var.enable_github_ci ? 1 : 0
  assume_role_policy = data.aws_iam_policy_document.github_plan_trust[0].json
}
```

Note `[0]` on the reference. A counted resource is always a list, even with one
element.

### Why `count` is dangerous for sets of things

`count` keys state by **position**:

```hcl
# before
count = 3   # names = ["alice", "bob", "carol"]
#   [0] alice   [1] bob   [2] carol

# remove "bob"
count = 2   # names = ["alice", "carol"]
#   [0] alice   [1] carol
```

Terraform compares by index. Index 1 was `bob` and is now `carol`, so it
**destroys bob and modifies carol** — rather than just destroying bob. On real
infrastructure that is an outage.

**Rule: use `count` for a toggle. Use `for_each` for a collection.**

## `for_each`

Keys state by a **string**, not a position:

```hcl
resource "aws_secretsmanager_secret" "this" {
  for_each = var.secrets

  name        = each.key
  description = each.value.description
  kms_key_id  = each.value.kms_key_id
  tags        = var.tags
}
```

That is `modules/secrets/main.tf`. With this input:

```hcl
secrets = {
  "arifa-df-jentic-api-key"             = { description = "Jentic API Key" }
  "arifa-df-galileo-api-key"            = { description = "Galileo Lite API key for ARIFA" }
  "arifa-df-anypoint-client-secret-dev" = { description = "Anypoint Application client secret to access Service APIs" }
}
```

you get three addresses, each keyed by name:

```
aws_secretsmanager_secret.this["arifa-df-jentic-api-key"]
aws_secretsmanager_secret.this["arifa-df-galileo-api-key"]
aws_secretsmanager_secret.this["arifa-df-anypoint-client-secret-dev"]
```

Remove one key and **only that one** is destroyed. The others are untouched,
because their state keys did not move.

Those stable addresses are what makes the import blocks in `envs/dev/imports.tf`
possible:

```hcl
import {
  to = module.secrets.aws_secretsmanager_secret.this["arifa-df-jentic-api-key"]
  id = "arn:aws:secretsmanager:eu-central-1:483075907105:secret:arifa-df-jentic-api-key-ySR0cr"
}
```

You could not write that against a `count`-based resource without pinning an
index that may later shift.

### `each.key` and `each.value`

| Input | `each.key` | `each.value` |
|-------|-----------|--------------|
| `map` | the key | the value |
| `set(string)` | the member | the member |

```hcl
for_each = toset(["a", "b"])   # each.key == each.value
```

`for_each` accepts only a map or a set of strings. Convert a list with
`toset()` — but only if duplicates are impossible and order genuinely does not
matter.

### Keys must be known at plan time

```hcl
# Fails: the key depends on something only known after apply
for_each = { for b in aws_s3_bucket.many : b.arn => b }
```

Terraform must know every address before it plans. Key by something static —
a variable, a literal, a name you control.

## `depends_on`

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "state" {
  # ...
  # Lifecycle rules on noncurrent versions need versioning to be on first.
  depends_on = [aws_s3_bucket_versioning.state]
}
```

Only for dependencies that exist in the real world but not in your references.
`bootstrap/state.tf` has two, both commented with the reason.

If you reach for `depends_on` often, look for a reference you could use
instead — those are self-documenting and cannot go stale.

## `lifecycle`

### `prevent_destroy`

```hcl
lifecycle {
  # Losing this bucket loses track of everything Terraform manages.
  prevent_destroy = true
}
```

Any plan that would destroy or replace this resource fails. Put it on state
buckets, production databases and KMS keys.

Note: it blocks the *plan*, so removing the resource block entirely also
requires removing this first — deliberately.

### `create_before_destroy`

Default order is destroy-then-create, which means downtime. Reverse it:

```hcl
lifecycle {
  create_before_destroy = true
}
```

Requires that both can exist at once — so names must be unique. Pair it with
`name_prefix` instead of `name` where the provider supports it.

### `ignore_changes`

```hcl
lifecycle {
  ignore_changes = [tags["LastScanned"]]
}
```

Tells Terraform to stop managing an attribute after creation. Use it when
something outside Terraform legitimately owns that field — an autoscaler setting
desired capacity, a tool stamping a tag.

> **Do not use `ignore_changes` to silence a diff you do not understand.** It
> hides the drift permanently rather than resolving it.
>
> `arifa-infra` makes this explicit for the secrets import:
>
> ```hcl
> # After an import, the first plan adds recovery_window_in_days = 30 and
> # force_overwrite_replica_secret = false to state; AWS never returns them. Apply it,
> # and never ignore_changes them.
> ```
>
> The right fix was to apply once and let state catch up. `ignore_changes` would
> have buried it forever.

`ignore_changes = all` exists. It almost always means the resource should be a
data source instead.

### `replace_triggered_by`

```hcl
lifecycle {
  replace_triggered_by = [aws_launch_template.this.latest_version]
}
```

Force replacement when something else changes.

### `precondition` and `postcondition`

```hcl
lifecycle {
  precondition {
    condition     = var.instance_type != "ml.p4d.24xlarge"
    error_message = "That instance type needs finance approval."
  }
}
```

Assertions checked at plan time. Use them for rules that span several values,
which a `variable` validation cannot see.

## `provider`

Selects a non-default provider configuration:

```hcl
module "secrets" {
  source    = "../../modules/secrets"
  providers = { aws = aws.imported }
  # ...
}
```

Note the difference:

- `provider = aws.imported` — singular, on a **resource**
- `providers = { aws = aws.imported }` — plural map, on a **module**

Chapter 04 explains why `arifa-infra` has an `imported` alias at all.

## `count` vs `for_each` — the summary

| | `count` | `for_each` |
|---|---|---|
| Address | `res[0]` | `res["name"]` |
| Keyed by | position | string |
| Remove a middle item | **shifts everything after it** | only that item |
| Input | number | map or set |
| Good for | on/off toggles, N identical things | collections of distinct things |

When unsure, use `for_each`.

## Try it

```bash
cd arifa-infra

grep -rn 'for_each\|count =' modules/ bootstrap/ envs/
grep -rn -B2 'depends_on' bootstrap/
grep -rn -A3 'lifecycle' bootstrap/ modules/
```

For each hit, say in one sentence why that meta-argument and not another.

## Check yourself

1. Why does removing a middle element from a `count` list cause extra changes?
2. What are `each.key` and `each.value` for a `set(string)`?
3. Why must `for_each` keys be known at plan time?
4. When is `depends_on` genuinely necessary?
5. What does `prevent_destroy` block, and at which stage?
6. Why does `arifa-infra` warn against `ignore_changes` on the secrets?
7. `provider` vs `providers` — which goes where?

---

Previous: [07 — State and backends](07-state-and-backends.md) ·
Next: [09 — Modules](09-modules.md)
