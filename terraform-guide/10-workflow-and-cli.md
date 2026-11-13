# 10 — The workflow and the CLI

## The loop

```bash
terraform init      # once per clone, and after changing providers/backend/modules
terraform plan      # as often as you like. Read-only.
terraform apply      # applies. Shows the plan and asks for confirmation.
```

Via `arifa-infra`'s Makefile, which hides the `-chdir` boilerplate:

```bash
make init  ENV=dev
make plan  ENV=dev
make apply ENV=dev
```

## `init`

Prepares a working directory:

1. Downloads providers into `.terraform/`
2. Downloads or links modules
3. Connects to the backend
4. Creates or verifies `.terraform.lock.hcl`

```bash
terraform init                      # normal
terraform init -lockfile=readonly   # fail if the lock file would change
terraform init -backend=false       # skip the backend — for validate only
terraform init -upgrade             # move to newer allowed provider versions
terraform init -migrate-state       # move state to a new backend
terraform init -reconfigure         # change backend, do not migrate
```

`arifa-infra` uses `-lockfile=readonly` everywhere so a provider upgrade is
always a deliberate commit. `-backend=false` is how `scripts/check.sh` validates
without AWS credentials.

Run `init` again whenever you add a provider, add a module, or change the
backend. It is safe to re-run.

## `plan`

```bash
terraform plan
terraform plan -out=tfplan          # save it
terraform plan -refresh-only        # only show drift
terraform plan -target=module.secrets   # limit scope — see the warning below
terraform plan -no-color            # for logs and CI
terraform plan -lock-timeout=5m     # wait for the state lock
```

A saved plan can be applied exactly:

```bash
terraform plan -out=tfplan
terraform apply tfplan     # no second confirmation — it was approved already
```

That is the right pattern for CI: plan on the pull request, apply the reviewed
artefact. `arifa-infra` does not apply from CI at all, by policy.

### Reading a plan

Only four symbols matter:

| Symbol | Meaning |
|--------|---------|
| `+` | create |
| `-` | destroy |
| `~` | change in place |
| `-/+` | **destroy and recreate** |

The summary is the headline:

```
Plan: 7 to add, 0 to change, 0 to destroy.
```

**Check `to destroy` on every plan.** If it is not zero, know exactly which
resources and why.

Then search the body for one string:

```
# forces replacement
```

That is the difference between editing a notebook and deleting it with
everything on its volume.

### Expected plans

`arifa-infra`'s README states the expected output for each step:

```
make bootstrap-plan  ->  Plan: 7 to add, 0 to change, 0 to destroy.
make plan ENV=dev    ->  Plan: 4 to import, 0 to add, 3 to change, 0 to destroy.
```

and then says:

> Any other change, or any "must be replaced", means the code does not match
> AWS: stop and fix the code.

Writing the expected plan into the runbook is a good habit — it turns review
from a judgement call into a comparison.

### `-target`

```bash
terraform plan -target=aws_s3_bucket.state
```

> Use it for debugging only. It plans a subset, so the result is not a
> consistent view of your configuration. Never make it part of a normal
> workflow — Terraform itself prints a warning saying so.

## `apply`

```bash
terraform apply                 # plan, then ask
terraform apply tfplan          # apply a saved plan
terraform apply -auto-approve   # no prompt — CI only, and only with a saved plan
```

The prompt is a feature. `arifa-infra`'s Makefile keeps it:

```makefile
apply: ## Apply ENV. Interactive: review the plan, then type yes
	terraform -chdir=envs/$(ENV) apply
```

> **Never `-auto-approve` interactively.** The confirmation is the last place a
> `-/+` gets caught.

## `destroy`

```bash
terraform destroy                       # everything in this root
terraform destroy -target=aws_instance.web   # one thing
```

Destroys everything the root manages. `prevent_destroy` blocks it for protected
resources.

A gentler way to remove one thing: delete its block and `apply`. Same result,
but the plan shows it alongside everything else.

## Day-to-day commands

```bash
terraform fmt -recursive          # rewrite to canonical style
terraform fmt -check -recursive   # fail if unformatted (CI)
terraform validate                # syntax and references; needs init
terraform console                 # evaluate expressions interactively
terraform output                  # show outputs
terraform show                    # show state or a saved plan
terraform graph                   # DOT dependency graph
terraform version                 # version, and whether an update exists
terraform providers               # providers each module requires
```

## State commands

```bash
terraform state list
terraform state show <address>
terraform state mv <from> <to>
terraform state rm <address>
terraform state pull > backup.json
terraform force-unlock <LOCK_ID>
```

> Back up before `mv`, `rm` or `push`: `terraform state pull > backup.json`.

## Environment variables

| Variable | Effect |
|----------|--------|
| `TF_LOG` | `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR` |
| `TF_LOG_PATH` | Write the log to a file |
| `TF_IN_AUTOMATION` | Suppress "next step" hints — set in CI |
| `TF_INPUT=0` | Never prompt; fail instead |
| `TF_VAR_<name>` | Set a variable |
| `AWS_PROFILE` | Which credentials to use |

`arifa-infra`'s CI sets the automation pair:

```yaml
env:
  TF_IN_AUTOMATION: "true"
  TF_INPUT: "0"
```

Debugging a provider problem:

```bash
TF_LOG=DEBUG TF_LOG_PATH=./tf.log terraform plan
```

## `-chdir`

```bash
terraform -chdir=envs/dev plan
```

Runs as if from that directory. Note it comes **before** the subcommand. This
is what every Makefile target wraps:

```makefile
plan: ## Plan ENV
	terraform -chdir=envs/$(ENV) plan
```

## A realistic session

```bash
# once
make init ENV=dev

# edit some .tf files
terraform fmt -recursive
make check                    # fmt, validate, tflint, trivy — no AWS needed

make plan ENV=dev             # read it: destroys? forces replacement?
git commit -am "..." && git push     # CI plans again on the PR

# after review and merge, from main
make apply ENV=dev            # read the plan once more, type yes
make plan ENV=dev             # confirm: No changes.
```

That final re-plan is worth the ten seconds. `No changes.` proves the apply
converged and nothing is drifting.

## Try it

```bash
cd arifa-infra

terraform version
terraform fmt -check -recursive        # should pass
bash scripts/check.sh fmt

terraform -chdir=modules/secrets init -backend=false
terraform -chdir=modules/secrets validate

terraform console
> upper("dev")
> merge({ a = 1 }, { b = 2 })
```

## Check yourself

1. When must you re-run `init`?
2. What does `-lockfile=readonly` protect against?
3. What do `+`, `-`, `~` and `-/+` mean?
4. Which string do you search a plan for before approving it?
5. Why is `-target` discouraged?
6. Why does the Makefile keep `apply` interactive?
7. What does `-chdir` do, and where does it go in the command?
8. Why re-plan immediately after an apply?

---

Previous: [09 — Modules](09-modules.md) ·
Next: [11 — Importing existing infrastructure](11-importing-existing-infrastructure.md)
