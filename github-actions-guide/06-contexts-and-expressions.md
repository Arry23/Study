# 06 — Contexts and expressions

## `${{ }}`

Anything inside `${{ }}` is evaluated by GitHub **before** the step runs:

```yaml
- run: echo "Branch is ${{ github.ref }}"
```

The runner never sees the expression — it sees the substituted result. That
distinction is the root of the script-injection problem in chapter 09.

## Contexts

Objects holding information about the run.

| Context | Contains |
|---------|----------|
| `github` | Event, repository, actor, ref, SHA |
| `env` | Variables from `env:` blocks |
| `vars` | Repository/org/environment variables |
| `secrets` | Secrets |
| `job` | Current job status and services |
| `jobs` | Other jobs' results (reusable workflows) |
| `steps` | Outputs of previous steps in this job |
| `needs` | Outputs of jobs this one needs |
| `runner` | OS, arch, temp directory |
| `matrix` | Current matrix values |
| `inputs` | `workflow_dispatch` / `workflow_call` inputs |

## The `github` context

```yaml
github.repository          # "roche-innersource/arifa-infrastructure"
github.repository_owner    # "roche-innersource"
github.ref                 # "refs/heads/main" / "refs/pull/42/merge"
github.ref_name            # "main"
github.sha                 # commit SHA
github.actor               # who triggered it
github.event_name          # "pull_request", "push", ...
github.run_id              # unique run ID
github.run_number          # incrementing counter
github.workspace           # checkout path
github.token               # the GITHUB_TOKEN
github.event.*             # the raw webhook payload
```

`arifa-infra` uses several:

```yaml
concurrency:
  group: terraform-${{ github.event.pull_request.number }}

- uses: aws-actions/configure-aws-credentials@<sha>
  with:
    role-session-name: arifa-plan-${{ github.run_id }}

run: scripts/check.sh
env:
  GITHUB_TOKEN: ${{ github.token }}
```

`github.run_id` in the session name makes every AWS session traceable back to
one workflow run — useful in CloudTrail.

### `github.event`

The entire webhook payload. Shape depends on the event:

```yaml
github.event.pull_request.number
github.event.pull_request.title          # UNTRUSTED
github.event.pull_request.head.ref       # UNTRUSTED
github.event.head_commit.message         # UNTRUSTED
```

> Anything a user can type is untrusted. Chapter 09.

To explore it:

```yaml
- run: echo "$EVENT"
  env:
    EVENT: ${{ toJSON(github.event) }}
```

## `env`, `vars`, `secrets`

```yaml
${{ env.TFLINT_VERSION }}       # from an env: block
${{ vars.AWS_PLAN_ROLE_ARN }}   # repository variable, not secret
${{ secrets.API_TOKEN }}        # secret, masked in logs
```

`vars` is for non-sensitive configuration — ARNs, region names, feature flags.
`secrets` is for credentials. Chapter 07.

```yaml
plan:
  if: vars.AWS_PLAN_ROLE_ARN != ''
```

A clean gate: the whole job is skipped until an administrator sets the variable.

## `steps` and `needs`

```yaml
- name: Read the Terraform version
  id: tf
  run: echo "version=$(cat .terraform-version)" >> "$GITHUB_OUTPUT"

- uses: hashicorp/setup-terraform@<sha>
  with:
    terraform_version: ${{ steps.tf.outputs.version }}
```

```yaml
deploy:
  needs: build
  steps:
    - run: echo "${{ needs.build.outputs.version }}"
```

Also available: `steps.<id>.outcome` and `.conclusion`, `needs.<job>.result`.

## `runner`

```yaml
runner.os            # Linux | Windows | macOS
runner.arch          # X64 | ARM64
runner.temp          # temp dir, cleaned automatically
runner.tool_cache
```

Used as `$RUNNER_TEMP` in shell:

```yaml
run: |
  bin="$RUNNER_TEMP/bin"
  mkdir -p "$bin"
```

```yaml
run: terraform plan | tee "$RUNNER_TEMP/plan.txt"
```

## Operators

```yaml
==  !=  <  <=  >  >=
&&  ||  !
```

Comparison is loose — strings and numbers coerce. Use `==` with explicit
strings:

```yaml
if: github.event_name == 'push'
if: github.ref == 'refs/heads/main'
if: vars.AWS_PLAN_ROLE_ARN != ''
```

## Functions

```yaml
contains(github.ref, 'release')
startsWith(github.ref, 'refs/tags/')
endsWith(github.ref, '/main')
format('{0}-{1}', github.run_id, matrix.env)
join(matrix.*.name, ', ')
toJSON(github.event)
fromJSON('["a","b"]')
hashFiles('**/package-lock.json')
```

`hashFiles` powers cache keys:

```yaml
key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
```

### Status functions

```yaml
success()     # default
failure()
always()
cancelled()
```

```yaml
- name: Write the plan to the job summary
  if: always()
```

Without `always()` this step would be skipped when the plan fails — exactly when
you most want the output.

> `always()` also runs on cancellation. If you want "on failure but not
> cancelled", use `if: failure()`.

## `if:`

```yaml
- run: ./deploy.sh
  if: github.ref == 'refs/heads/main'
```

**No `${{ }}` needed** — `if:` is already an expression context. Both work;
the bare form is conventional.

Job level:

```yaml
jobs:
  plan:
    needs: check
    if: vars.AWS_PLAN_ROLE_ARN != ''
```

A skipped job shows as "Skipped", not failed. Jobs depending on it are skipped
too, unless they use `always()` or `optional`-style handling.

## Multi-line and YAML

```yaml
# Fine
if: github.event_name == 'push' && github.ref == 'refs/heads/main'

# Starts with {{ — YAML flow mapping. Quote it.
if: ${{ github.event_name == 'push' }}       # needs quoting in some positions
```

When an expression starts a value and uses `${{`, quote the whole thing:

```yaml
name: "${{ matrix.env }} plan"
```

## Common patterns

**Only on the default branch**

```yaml
if: github.ref == format('refs/heads/{0}', github.event.repository.default_branch)
```

**Only on tags**

```yaml
if: startsWith(github.ref, 'refs/tags/v')
```

**Skip draft PRs**

```yaml
if: github.event.pull_request.draft == false
```

**Not from a fork**

```yaml
if: github.event.pull_request.head.repo.full_name == github.repository
```

**Only when a secret exists**

```yaml
if: vars.AWS_PLAN_ROLE_ARN != ''
```

> You cannot use `secrets.*` in a job-level `if:`. That is why `arifa-infra`
> gates on a **variable** rather than a secret — the role ARN is not sensitive,
> so it belongs in `vars` anyway.

## Debugging expressions

```yaml
- run: |
    echo "event=${{ github.event_name }}"
    echo "ref=${{ github.ref }}"
    echo "actor=${{ github.actor }}"
```

Dump a whole context safely:

```yaml
- run: echo "$CTX"
  env:
    CTX: ${{ toJSON(github) }}
```

> Note the indirection through `env:` rather than inlining `${{ toJSON(github) }}`
> in the script. That is the script-injection-safe form — chapter 09.
>
> Never dump `secrets` this way.

Enable verbose logs with repository secrets `ACTIONS_STEP_DEBUG: true` and
`ACTIONS_RUNNER_DEBUG: true`.

## Try it

In [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. Find every `${{ }}`. Which context does each use?
2. Why is the concurrency group built from `github.event.pull_request.number`?
3. Why `github.run_id` in the AWS session name?
4. Why gate the plan job on `vars` rather than `secrets`?
5. What would break if the summary step dropped `if: always()`?

## Check yourself

1. When is `${{ }}` evaluated — before or during the step?
2. `vars` vs `secrets` — when each?
3. Why no `${{ }}` in `if:`?
4. What does `always()` include that `failure()` does not?
5. Why can't you use `secrets.*` in a job-level `if:`?
6. How do you safely dump a context for debugging?

---

Previous: [05 — Actions](05-actions.md) ·
Next: [07 — Variables and secrets](07-variables-and-secrets.md)
