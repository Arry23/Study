# 04 — Jobs and steps

## A job

```yaml
jobs:
  check:
    name: check
    runs-on: rcn-ubuntu-amd64
    timeout-minutes: 15
    steps:
      - ...
```

`check` is the **job ID** — used by `needs:` and in expressions. `name:` is the
display label.

Jobs run in parallel unless `needs:` says otherwise, and each gets its own
runner with its own filesystem.

## `runs-on`

```yaml
runs-on: ubuntu-latest          # GitHub-hosted
runs-on: rcn-ubuntu-amd64       # self-hosted, by label
runs-on: [self-hosted, linux, x64]   # must match ALL labels
```

`arifa-infra` uses `rcn-ubuntu-amd64`, a Roche runner, because GitHub-hosted
runners are disabled by policy. The workflow header says so, and adds a
consequence worth noting:

> It has no make, so CI runs scripts/check.sh and the terraform commands behind
> the make targets.

That is why the workflow calls `scripts/check.sh` directly and writes out the
Terraform commands longhand, with a comment:

```yaml
# The next two steps are `make init` and `make plan`, written out.
```

Chapter 13.

## Steps

Each step is either `uses:` or `run:` — never both.

```yaml
steps:
  - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
    with:
      persist-credentials: false

  - name: Read the Terraform version
    id: tf
    run: echo "version=$(cat .terraform-version)" >> "$GITHUB_OUTPUT"

  - uses: hashicorp/setup-terraform@dfe3c3f87815947d99a8997f908cb6525fc44e9e # v4.0.1
    with:
      terraform_version: ${{ steps.tf.outputs.version }}
      terraform_wrapper: false
```

| Key | Purpose |
|-----|---------|
| `name` | Display label |
| `id` | Reference this step's outputs |
| `uses` | Run an action |
| `with` | Inputs to the action |
| `run` | Shell command |
| `env` | Step-scoped variables |
| `if` | Conditional |
| `continue-on-error` | Failure does not fail the job |
| `timeout-minutes` | Cap this step |
| `shell` | Which shell |
| `working-directory` | Where to run |

## Steps share a workspace

Everything in one job runs on one machine, in `$GITHUB_WORKSPACE`. A file
written by step 2 is there for step 5.

But each `run` is a **separate shell invocation**:

```yaml
- run: cd src          # has no effect on the next step
- run: pwd             # still $GITHUB_WORKSPACE
```

Three fixes:

```yaml
- run: |               # one shell
    cd src
    pwd

- run: pwd
  working-directory: src

- run: echo "RESULT=x" >> "$GITHUB_ENV"   # persist a variable
```

## Passing data between steps

**Step outputs** — write to `$GITHUB_OUTPUT`:

```yaml
- name: Read the Terraform version
  id: tf
  run: echo "version=$(cat .terraform-version)" >> "$GITHUB_OUTPUT"

- uses: hashicorp/setup-terraform@<sha>
  with:
    terraform_version: ${{ steps.tf.outputs.version }}
```

A small, elegant pattern: the version lives in one file that both the Makefile
and CI read, so they cannot drift.

**Environment variables** — write to `$GITHUB_ENV`:

```yaml
- run: echo "BUILD_ID=$(date +%s)" >> "$GITHUB_ENV"
- run: echo "$BUILD_ID"
```

`$GITHUB_OUTPUT` is scoped and explicit; `$GITHUB_ENV` leaks into every later
step. Prefer outputs.

**Adding to PATH:**

```yaml
- run: |
    bin="$RUNNER_TEMP/bin"
    mkdir -p "$bin"
    # ... install tools into $bin ...
    echo "$bin" >> "$GITHUB_PATH"
```

That is exactly how `arifa-infra` makes tflint and trivy available to the next
step.

## Passing data between jobs

```yaml
jobs:
  build:
    outputs:
      version: ${{ steps.v.outputs.version }}
    steps:
      - id: v
        run: echo "version=1.2.3" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    steps:
      - run: echo "${{ needs.build.outputs.version }}"
```

For files, use artifacts (chapter 11).

## Shells

Default is `bash` on Linux and macOS, PowerShell on Windows.

```yaml
- run: echo hello
  shell: bash
```

> **Important.** The default `bash` runs with `-e` but **not** `-o pipefail`.
> So this passes even when the first command fails:
>
> ```yaml
> - run: terraform plan | tee plan.txt
> ```
>
> `arifa-infra` handles it explicitly:
>
> ```yaml
> - name: terraform plan
>   run: |
>     set -o pipefail
>     terraform -chdir="envs/$TF_ENV" plan -no-color -lock-timeout=5m | tee "$RUNNER_TEMP/plan.txt"
> ```
>
> Without `set -o pipefail`, a failing plan would be reported as success because
> `tee` exited 0.

For multi-command scripts, be explicit:

```yaml
- run: |
    set -euo pipefail
    step_one
    step_two
```

Or set it for the whole workflow:

```yaml
defaults:
  run:
    shell: bash
```

## A worked installation step

```yaml
- name: Install tflint and trivy (pinned, checksum verified)
  run: |
    set -euo pipefail
    bin="$RUNNER_TEMP/bin"
    mkdir -p "$bin"
    cd "$RUNNER_TEMP"

    curl -fsSLo tflint.zip "https://github.com/terraform-linters/tflint/releases/download/v${TFLINT_VERSION}/tflint_linux_amd64.zip"
    echo "${TFLINT_SHA256}  tflint.zip" | sha256sum --check --strict
    unzip -q -o tflint.zip tflint -d "$bin"

    curl -fsSLo trivy.tar.gz "https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz"
    echo "${TRIVY_SHA256}  trivy.tar.gz" | sha256sum --check --strict
    tar -xzf trivy.tar.gz -C "$bin" trivy

    echo "$bin" >> "$GITHUB_PATH"
```

Worth studying:

- `set -euo pipefail` first
- `curl -f` fails on HTTP errors rather than saving an error page
- **`sha256sum --check --strict`** before unpacking — a moved tag or a
  compromised mirror is caught here
- `$RUNNER_TEMP` is cleaned automatically
- `$GITHUB_PATH` exposes the tools to later steps

The reason this is hand-rolled rather than using an action is in the header:

> trivy-action is not used on purpose: its tags were hijacked in March 2026
> (GHSA-69fq-xp46-6x23).

## Conditionals

```yaml
- name: Write the plan to the job summary
  if: always()
```

| Expression | Meaning |
|------------|---------|
| `success()` | All previous steps passed (implicit default) |
| `failure()` | A previous step failed |
| `always()` | Always, including cancellation |
| `cancelled()` | The run was cancelled |

```yaml
- run: ./upload-logs.sh
  if: failure()

- run: ./deploy.sh
  if: github.ref == 'refs/heads/main'
```

Note: no `${{ }}` needed in `if:` — it is already an expression context.
Chapter 06.

## Error handling

```yaml
- run: ./flaky.sh
  continue-on-error: true      # step may fail; job continues
```

```yaml
jobs:
  optional:
    continue-on-error: true    # job may fail; workflow continues
```

```yaml
- run: ./maybe.sh || true      # swallow at shell level
```

Prefer `continue-on-error` over `|| true` — it is visible in the UI.

## Timeouts

```yaml
jobs:
  check:
    timeout-minutes: 15        # default is 360
    steps:
      - run: ./slow.sh
        timeout-minutes: 5
```

## Try it

In [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. Find every `uses` step and every `run` step.
2. How does the Terraform version flow from a file into the setup action?
3. Why is `set -o pipefail` explicit in the plan step?
4. Why does the install step verify checksums?
5. Why does the summary step use `if: always()`?
6. Why does `plan` repeat the checkout?

## Check yourself

1. Can a step have both `uses` and `run`?
2. Why doesn't `cd` persist between `run` steps?
3. `$GITHUB_OUTPUT` vs `$GITHUB_ENV` — which and why?
4. What does default `bash` not set, and why does it matter?
5. How do you add a directory to PATH for later steps?
6. How do you pass a value from one job to another?

---

Previous: [03 — Triggers](03-triggers.md) ·
Next: [05 — Actions](05-actions.md)
