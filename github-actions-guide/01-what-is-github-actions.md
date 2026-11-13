# 01 — What GitHub Actions is

## The problem it solves

Someone opens a pull request. Before it can merge you want to know: does it
build, do the tests pass, is it formatted, does it contain a leaked credential,
would it break production?

GitHub Actions runs those checks automatically, on every change, on a clean
machine.

## The moving parts

```
  .github/workflows/*.yml      GitHub                  Runner
  (your workflows)        ->   matches the event,  ->  executes the
                               queues jobs             steps
```

**Workflow file** — YAML in `.github/workflows/`. A repository can have many;
each is independent.

**Event** — a push, a pull request, a schedule, a manual click. GitHub matches
events against each workflow's `on:` block.

**Runner** — a virtual machine or container that executes the job. GitHub-hosted
by default; `arifa-infra` uses a self-hosted Roche runner.

## Workflow, job, step

```
workflow                       one file, triggered by events
├── job: check                 runs on one runner
│   ├── step: checkout         a reusable action
│   ├── step: setup terraform  another action
│   └── step: run check.sh     a shell command
└── job: plan                  a DIFFERENT runner, possibly parallel
    └── ...
```

**Jobs run in parallel by default.** There are no stages — if you need ordering,
you declare it:

```yaml
plan:
  needs: check
```

That is the single biggest structural difference from GitLab CI, where stages
impose order automatically.

**Steps run in sequence** inside one job, sharing a filesystem and a workspace.

## A real workflow

From `arifa-infra`:

```yaml
name: terraform

on:
  pull_request:
    paths:
      - "**/*.tf"
      - "Makefile"

permissions:
  contents: read

jobs:
  check:
    name: check
    runs-on: rcn-ubuntu-amd64
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false

      - name: Quality gates (scripts/check.sh)
        run: scripts/check.sh

  plan:
    needs: check
    if: vars.AWS_PLAN_ROLE_ARN != ''
    runs-on: rcn-ubuntu-amd64
    permissions:
      contents: read
      id-token: write
    steps:
      # ...
```

Reading it: on a pull request touching Terraform files, run `check`. If it
passes **and** a repository variable is set, run `plan` with permission to
request an OIDC token.

## Two kinds of step

This is the defining feature of Actions.

**`run`** — a shell command:

```yaml
- name: Read the Terraform version
  id: tf
  run: echo "version=$(cat .terraform-version)" >> "$GITHUB_OUTPUT"
```

**`uses`** — someone else's reusable code:

```yaml
- uses: hashicorp/setup-terraform@dfe3c3f87815947d99a8997f908cb6525fc44e9e # v4.0.1
  with:
    terraform_version: ${{ steps.tf.outputs.version }}
    terraform_wrapper: false
```

An **action** is a packaged unit — install a tool, upload an artifact,
authenticate to a cloud. There are thousands on the Marketplace.

> That convenience is also the risk. `uses:` runs third-party code on a machine
> holding your credentials. Chapters 05 and 09.

## Checkout is not automatic

```yaml
- uses: actions/checkout@<sha>
```

Unlike GitLab, GitHub does **not** clone your repository automatically. Forget
this step and your workflow runs against an empty directory. It is the single
most common beginner error.

## Each job gets a fresh machine

Two jobs mean two runners and two filesystems. Note that `arifa-infra`'s `plan`
job repeats the checkout and the Terraform setup that `check` already did —
not duplication by mistake, but necessity.

To move files between jobs, use artifacts (chapter 11).

## Success and failure

A step fails if its command exits non-zero. A failed step fails the job and
skips the remaining steps — unless told otherwise:

```yaml
- run: ./flaky.sh
  continue-on-error: true

- name: Write the plan to the job summary
  if: always()        # run even if an earlier step failed
```

`if: always()` is how you keep diagnostics when things go wrong — exactly what
the plan summary step does.

## Where you see results

- **Actions tab** — every run
- **A run** — the job graph; click a job for logs
- **Pull request** — checks appear in the merge box
- **Job summary** — rendered Markdown written to `$GITHUB_STEP_SUMMARY`

That last one is how the Terraform plan is surfaced without needing permission
to comment on the PR.

## Try it

Open [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. What events trigger it?
2. How many jobs, and what forces their order?
3. Which steps are `uses` and which are `run`?
4. Why does `plan` check out the repository again?
5. Which step runs even when the job is failing?

## Check yourself

1. Do jobs run in parallel or in sequence by default? And steps?
2. What is the difference between `run` and `uses`?
3. Why must you add a checkout step?
4. Why can't job B see files job A created?
5. What does `if: always()` do?

---

Next: [02 — Workflow anatomy](02-workflow-anatomy.md)
