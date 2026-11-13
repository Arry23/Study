# 03 — Triggers

`on:` decides when a workflow runs. Getting it wrong means either burning
runner minutes on irrelevant changes, or — worse — never checking the code that
matters.

## The common events

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 6 * * 1-5"
  workflow_dispatch:
  release:
    types: [published]
```

| Event | Fires on |
|-------|----------|
| `push` | A commit pushed to a branch or tag |
| `pull_request` | PR opened, synchronised, reopened |
| `pull_request_target` | Same, but in the **base** repo context — dangerous |
| `schedule` | Cron |
| `workflow_dispatch` | Manual button, with optional inputs |
| `workflow_call` | Called by another workflow |
| `release` | A release published |
| `issues`, `issue_comment` | Issue activity |
| `workflow_run` | Another workflow finished |

## Path filters

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

The workflow runs only when a file that could change the result was touched.

Each entry is deliberate: `.tf` files are the code, the lock file pins
providers, `.terraform-version` pins Terraform, `.tflint.hcl` and `trivy.yaml`
configure the gates, `scripts/**` is what CI executes, and the workflow watches
itself.

> GitLab's `rules:changes` is unreliable on branch pipelines because it compares
> against the previous commit. GitHub's `paths:` uses the full PR diff and is
> dependable.

Inverse form:

```yaml
paths-ignore:
  - "**/*.md"
  - "docs/**"
```

Use one or the other, never both in the same trigger.

> A required status check that is skipped by a path filter stays **pending**
> forever and blocks merge. Either do not mark it required, or add a tiny
> always-runs job that reports success.

## Branch and tag filters

```yaml
on:
  push:
    branches:
      - main
      - "release/**"
    branches-ignore:
      - "dependabot/**"
    tags:
      - "v*.*.*"
```

Patterns: `*` matches within a path segment, `**` across segments.

## `push` and `pull_request` together

```yaml
on:
  push:
    branches: [main]
  pull_request:
```

The standard setup: validate PRs, and re-validate `main` after merge. A merge
commit can differ from what was tested, and admin merges bypass PRs entirely.

> **`arifa-infra` only has `pull_request`.** Nothing validates `main` after a
> merge. Adding four lines closes it:
>
> ```yaml
> on:
>   pull_request:
>     paths: [...]
>   push:
>     branches: [main]
> ```

Without path filters this combination double-runs on PR branches in the same
repository. With `branches: [main]` on the push trigger, it does not.

## `pull_request` activity types

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
```

Default is `[opened, synchronize, reopened]`. Add `ready_for_review` to skip
drafts, paired with:

```yaml
jobs:
  test:
    if: github.event.pull_request.draft == false
```

## `pull_request` vs `pull_request_target`

> **The most dangerous distinction in GitHub Actions.**

| | `pull_request` | `pull_request_target` |
|---|---|---|
| Code checked out | The PR's code | The **base** branch |
| Secrets available | No, for forks | **Yes** |
| `GITHUB_TOKEN` | Read-only for forks | Write |

`pull_request_target` exists so a workflow can label or comment on a fork PR.
It runs in the trusted base context **with secrets**.

```yaml
# DANGEROUS
on: pull_request_target
jobs:
  build:
    steps:
      - uses: actions/checkout@<sha>
        with:
          ref: ${{ github.event.pull_request.head.sha }}   # the fork's code
      - run: npm install && npm test                        # runs their code
```

Checking out untrusted code and executing it, with secrets present, is full
compromise. Anyone can open a PR.

**Rule: never check out PR code in a `pull_request_target` workflow.** Use it
only for metadata operations on the base checkout.

## Scheduled runs

```yaml
on:
  schedule:
    - cron: "0 6 * * 1-5"       # 06:00 UTC, weekdays
```

Five fields: minute, hour, day-of-month, month, day-of-week. **Always UTC.**

Uses: nightly dependency scans, drift detection, cache warming.

> Terraform repositories benefit enormously from a scheduled plan. `arifa-infra`
> has none, so infrastructure drift — someone changing a resource in the console
> — only surfaces when an unrelated PR happens to run a plan. See
> [the Terraform guide, chapter 07](../terraform-guide/07-state-and-backends.md).

Caveats: schedules only run on the default branch, can be delayed under load,
and are disabled after 60 days of repository inactivity.

## Manual runs

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        required: true
        default: dev
        type: choice
        options: [dev, staging, prod]
      dry_run:
        type: boolean
        default: true
```

Adds a "Run workflow" button. Read inputs with `inputs.environment`.

Types: `string`, `number`, `boolean`, `choice`, `environment`.

This is the usual pattern for `apply` jobs that must stay deliberate.

## Chaining workflows

```yaml
on:
  workflow_run:
    workflows: ["terraform"]
    types: [completed]

jobs:
  notify:
    if: github.event.workflow_run.conclusion == 'failure'
```

Runs in the base repository context with secrets, so the same caution as
`pull_request_target` applies.

## Filtering further with `if`

`on:` is coarse; `if:` refines:

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

```yaml
plan:
  if: vars.AWS_PLAN_ROLE_ARN != ''
```

That second one is from `arifa-infra`: the plan job is skipped entirely until a
repository variable is set, so static checks are useful before the AWS side
exists.

Chapter 06.

## Skipping a run

Including `[skip ci]` or `[no ci]` in a commit message skips `push` and
`pull_request` workflows. Useful for documentation-only commits, though path
filters are a better long-term answer.

## Try it

In [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. What is the only trigger?
2. Explain each of the eight path patterns.
3. Why is `.github/workflows/terraform.yml` itself listed?
4. What is not validated because there is no `push:` trigger?
5. Write the `on:` block that adds validation of `main` and a nightly drift plan.

## Check yourself

1. Why are path filters more reliable here than GitLab's `rules:changes`?
2. `pull_request` vs `pull_request_target` — what changes?
3. What must you never do in a `pull_request_target` workflow?
4. What timezone is `schedule:` in?
5. Why can a path-filtered required check block a merge?
6. When would you use `workflow_dispatch`?

---

Previous: [02 — Workflow anatomy](02-workflow-anatomy.md) ·
Next: [04 — Jobs and steps](04-jobs-and-steps.md)
