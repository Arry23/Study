# 02 — Workflow anatomy

## Where files live

```
.github/
└── workflows/
    ├── terraform.yml
    ├── tests.yml
    └── release.yml
```

Only `.github/workflows/` is scanned, and only at the repository root. Each file
is an independent workflow with its own triggers.

Many small workflows beat one large one: they run in parallel, fail
independently, and you can restrict each to the paths it cares about.

## Top-level keys

```yaml
name: terraform               # shown in the UI

on: ...                       # what triggers it

permissions: ...              # GITHUB_TOKEN scope

concurrency: ...              # cancellation / serialisation

env: ...                      # variables for every job

defaults: ...                 # default shell, working directory

jobs: ...                     # the work
```

Only `on:` and `jobs:` are required.

## `name`

```yaml
name: terraform
```

Without it GitHub uses the file path. Give it a name — it appears on every PR
check.

Individual runs can be named too:

```yaml
run-name: deploy to ${{ inputs.environment }} by @${{ github.actor }}
```

## `on`

```yaml
on:
  pull_request:
    paths:
      - "**/*.tf"
      - "**/.terraform.lock.hcl"
      - ".terraform-version"
      - ".tflint.hcl"
      - "trivy.yaml"
      - "Makefile"
      - "scripts/**"
      - ".github/workflows/terraform.yml"
```

Pull requests only, and only when a file that could change the outcome was
touched. A README edit does not spend runner minutes.

Note the last entry: the workflow watches **itself**, so a change to the
pipeline is validated by the pipeline.

> This workflow has no `push:` trigger, so nothing validates `main` after merge.
> A direct push or an admin merge is never checked. Chapter 03.

## `permissions`

```yaml
permissions:
  contents: read
```

The scope of the automatic `GITHUB_TOKEN`. Declaring it at the top sets a
restrictive default; individual jobs raise it only where needed:

```yaml
jobs:
  plan:
    permissions:
      contents: read
      id-token: write       # only this job can request an OIDC token
```

This is least privilege, and GitLab has no equivalent. Chapter 08.

## `concurrency`

```yaml
concurrency:
  group: terraform-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

Runs sharing a group do not run simultaneously. Keyed on the PR number, so
pushing twice cancels the first run — saving minutes and avoiding a stale
result overwriting a fresh one.

> Note this serialises only **within** one PR. Two different PRs still run at
> once, which matters when they share a resource such as a Terraform state lock.
> Chapter 10.

## `env`

```yaml
env:
  TF_IN_AUTOMATION: "true"
  TF_INPUT: "0"
  TFLINT_VERSION: "0.65.0"
  TFLINT_SHA256: "d4c545ed2e29d6338604b586f34f544f380f70f243aa72f7b484713835eaec61"
  TRIVY_VERSION: "0.76.1"
  TRIVY_SHA256: "c7cbe3137fb8a48a17574bede3dd6beb16742d1e9772929940fc90082c39d9c0"
```

Workflow-level variables, visible to every step.

Two patterns worth copying. `TF_IN_AUTOMATION` and `TF_INPUT` put Terraform in
non-interactive mode. The version/checksum pairs keep pinned tool versions in
one obvious place — and the comment above them records when they were last
checked:

```yaml
# Pinned tool versions and archive checksums, checked 2026-11-02.
```

`env` can also be set per job and per step. Nearest wins.

## `defaults`

```yaml
defaults:
  run:
    shell: bash
    working-directory: ./src
```

Applies to every `run` step. Useful in a monorepo where one workflow only
touches a subdirectory.

## `jobs`

```yaml
jobs:
  check:                      # job ID — referenced by needs:
    name: check               # display name
    runs-on: rcn-ubuntu-amd64
    timeout-minutes: 15
    steps:
      - ...

  plan:
    name: plan
    needs: check              # waits for check
    if: vars.AWS_PLAN_ROLE_ARN != ''
    runs-on: rcn-ubuntu-amd64
    timeout-minutes: 20
    permissions:
      contents: read
      id-token: write
    strategy:
      fail-fast: false
      matrix:
        env: [dev]
    steps:
      - ...
```

| Key | Meaning | Chapter |
|-----|---------|---------|
| `runs-on` | Which runner | 13 |
| `needs` | Wait for other jobs | this one |
| `if` | Whether to run at all | 06 |
| `permissions` | Token scope | 08 |
| `strategy.matrix` | Run variants in parallel | 10 |
| `timeout-minutes` | Cap runtime | this one |
| `env` | Job-scoped variables | 07 |
| `outputs` | Values for dependent jobs | 06 |
| `container` / `services` | Run in a container; add sidecars | 13 |
| `steps` | The work. **Required** | 04 |

## Ordering with `needs`

```yaml
jobs:
  check:   { ... }
  plan:    { needs: check }
  apply:   { needs: [check, plan] }
```

No `needs` means the job starts immediately. This is a graph from the start —
there is no "stage" concept to fall back on.

By default a job is skipped if anything it needs fails. To run anyway:

```yaml
notify:
  needs: [check, plan]
  if: always()
```

## Always set a timeout

```yaml
timeout-minutes: 15
```

The default is **360 minutes**. A hung job burns six hours of runner time.
`arifa-infra` sets 15 and 20.

## Validating a workflow

- **Push it** — GitHub reports syntax errors in the Actions tab
- **VS Code** — the GitHub Actions extension gives schema validation
- **`actionlint`** — a dedicated linter that catches far more than YAML errors:

```bash
actionlint .github/workflows/terraform.yml
```

It flags shell bugs inside `run:`, invalid contexts, and — importantly —
script-injection risks (chapter 09). Worth adding to pre-commit.

- Plain YAML validity:

```bash
python -c "import yaml; yaml.safe_load(open('.github/workflows/terraform.yml'))"
```

## Try it

In [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. Which top-level keys are used?
2. Why does the `paths:` filter include the workflow file itself?
3. Why is `permissions` declared twice?
4. What does the `concurrency` group key mean in practice?
5. Why are tool checksums in `env` rather than inline?

## Check yourself

1. Where must workflow files live?
2. Which two top-level keys are required?
3. What does a top-level `permissions` block establish?
4. What is the default job timeout, and why override it?
5. How do you make a job wait for another?
6. What does `actionlint` catch that a YAML parser does not?

---

Previous: [01 — What GitHub Actions is](01-what-is-github-actions.md) ·
Next: [03 — Triggers](03-triggers.md)
