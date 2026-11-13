# 15 — Best practices

Patterns worth copying, and the mistakes that cost the most.

## Repository layout

```
arifa-infra/
├── bootstrap/          state bucket + CI role — applied once per account
├── envs/
│   └── dev/            one root per environment
├── modules/
│   ├── sagemaker-notebook/
│   └── secrets/
├── scripts/check.sh    quality gates, shared by make and CI
├── docs/               importing.md, roadmap.md
├── Makefile            local convenience
└── .github/workflows/  CI
```

**One root per environment**, each with its own state and backend. Dev and prod
cannot touch each other, and an apply's blast radius is one directory.

The alternative, workspaces, keeps one configuration and switches state:

```bash
terraform workspace select prod
```

It looks tidier but hides which environment you are in behind shell state, and
forces conditionals (`var.env == "prod" ? ... : ...`) through the code. Separate
directories cost duplication and buy obviousness. **Prefer directories.**

**Bootstrap is separate** because it creates the state bucket that everything
else depends on, and because the CI role belongs somewhere a bad environment
apply cannot reach:

> The CI role lives here, not in an environment root, so a bad environment apply
> cannot delete the pipeline's own credentials.

## Naming

**Resources:** `this` for a module's single main resource; a descriptive name
when there are several of a type.

```hcl
resource "aws_secretsmanager_secret" "this" { }   # modules/secrets
resource "aws_s3_bucket" "state" { }              # bootstrap
```

Do not repeat the type. `aws_s3_bucket.state_bucket` reads as "s3 bucket state
bucket".

**Real names:** build them from components so they are unique and self-describing.

```hcl
state_bucket_name = "arifa-infrastructure-tfstate-${local.account_id}-${local.region}"
```

S3 bucket names are globally unique; including account and region makes
collisions impossible and tells you at a glance what a bucket is.

**Files:** by role — `main.tf`, `variables.tf`, `outputs.tf`, `providers.tf`,
`versions.tf`, `backend.tf`. Terraform does not care; readers do.

## Pin everything

| Thing | Where | Why |
|-------|-------|-----|
| Terraform | `.terraform-version`, `required_version` | Same plan everywhere |
| Providers | `~> 6.71` plus `.terraform.lock.hcl` | Reproducible |
| Modules | `version` or `?ref=` | No silent upstream changes |
| tflint plugins | `.tflint.hcl` | Same rules on laptop and CI |
| GitHub Actions | full commit SHA | Tags can be moved |
| pre-commit hooks | full commit SHA | Same |
| Tool downloads | version + SHA-256 | Supply chain |

`arifa-infra` pins every one of these. The reason is always the same: a change
should come from a commit, not from the passage of time.

## Comment the why

The code shows what. Comments earn their place by explaining why.

```hcl
# Losing this bucket loses track of everything Terraform manages.
prevent_destroy = true

# Lifecycle rules on noncurrent versions need versioning to be on first.
depends_on = [aws_s3_bucket_versioning.state]

# RCP's guardrails deny changing the owner tag, so no plan may touch it.
ignore_tags { keys = ["owner", "Owner"] }
```

Each answers a question a reviewer would otherwise ask, and prevents a future
"this looks unnecessary" deletion.

Also comment what you deliberately did **not** do:

```hcl
# As it is today, on purpose: no VPC, direct internet access and root access on, and
# a volume key that AWS chooses. The import changes none of it.
# TODO: make it perfect, one pull request at a time: docs/roadmap.md, items 3, 4 and 7 to 10.
```

Without that, the next reader cannot tell a known gap from an oversight.

## Make plans reviewable

The plan is the artefact a human approves, so protect its signal.

**Split mechanical changes from semantic ones.** The `aws.imported` provider
exists so imports show no tag churn; tagging lands as its own pull request.

**Write down the expected plan.**

```
make bootstrap-plan  ->  Plan: 7 to add, 0 to change, 0 to destroy.
make plan ENV=dev    ->  Plan: 4 to import, 0 to add, 3 to change, 0 to destroy.
```

> Any other change, or any "must be replaced", means the code does not match
> AWS: stop and fix the code.

Review becomes comparison rather than judgement.

**Keep roots small.** A 300-resource plan does not get read.

## Avoid hidden inputs

`arifa-infra` has no tfvars files:

```hcl
# Its values live here, not in tfvars, so CI plans exactly what a laptop plans.
```

Hidden inputs mean a plan can differ between machines for reasons invisible in
the diff. Committing values trades duplication for a guarantee.

Variables still belong in modules — that is their interface. The rule applies to
roots.

## Module design

- Minimum version constraints, not pins — the root pins
- No `provider` blocks — passed in by the caller
- Never create another team's resources; take IDs as inputs
- Expose what varies; hardcode what must not
- Validate at the boundary, with a message that says what to do
- One sentence describes the module, without "and"

## Have a roadmap

`docs/roadmap.md` lists twelve ordered improvements, each with its dependencies
and its risk. Two are flagged as replacing the notebook, with a note that the
volume must be backed up first.

A roadmap turns "we know this is not ideal" into a plan someone can pick up. It
is also what lets the import phase ship without pretending it is finished.

## Anti-patterns

**`count` for collections.** Removing a middle element shifts every index after
it, so Terraform destroys and recreates resources you did not touch. Use
`for_each`.

**`ignore_changes` to silence a diff you do not understand.** It hides the
symptom permanently. Understand the diff; apply it if it is correct.

> After an import, the first plan adds `recovery_window_in_days = 30` and
> `force_overwrite_replica_secret = false` to state; AWS never returns them.
> Apply it, and never `ignore_changes` them.

**Hardcoded credentials.** Including "temporary" ones. They reach git, and git
is forever.

**Committed state.** `*.tfstate` in `.gitignore`, always. It contains secrets
and it will conflict.

**Unpinned module sources.** `source = "org/module/aws"` with no version changes
underneath you.

**`-auto-approve` interactively.** The prompt is where you catch `-/+`.

**`-target` as routine.** It produces an inconsistent view. Debugging only.

**Giant roots.** If `terraform state list` scrolls past a screen, split it.

**Terraform managing secret values.** They land in state. Manage the container.

**A quality gate that can silently pass.** Worse than no gate, because you
trust it. The `&&`-chain bug in `scripts/check.sh` let `make check` report
success while two roots failed to validate.

## Review checklist

**Plan**
- [ ] `to destroy` is zero, or every destruction is understood
- [ ] No unexpected `# forces replacement`
- [ ] Matches the expected plan in the runbook

**Code**
- [ ] Formatted (`terraform fmt`)
- [ ] All gates pass
- [ ] New variables have `description` and `type`
- [ ] Validation on constrained inputs
- [ ] Comments explain why, not what
- [ ] No secrets anywhere
- [ ] Lock files updated and committed if providers changed

**Structure**
- [ ] Modules take providers, do not declare them
- [ ] Modules state minimum versions
- [ ] Roots pin exact versions
- [ ] New remote sources pinned

## Try it

Review `arifa-infra` against the checklist. Then find:

1. A comment that prevents a future wrong "simplification"
2. A deliberate trade-off with its reasoning recorded
3. A place where a mechanical change was split from a semantic one

## Check yourself

1. Why directories per environment rather than workspaces?
2. Why does the account ID appear in the state bucket name?
3. Why pin a GitHub Action to a SHA instead of a tag?
4. Why avoid tfvars in a root module?
5. Why is `count` wrong for a set of named things?
6. When is `ignore_changes` legitimate?
7. Why write the expected plan into the README?

---

Previous: [14 — Security and secrets](14-security-and-secrets.md) ·
Next: [16 — Troubleshooting](16-troubleshooting.md)
