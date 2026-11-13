# 16 — Cheat sheet

One page. Keep it open.

## Workflow skeleton

```yaml
name: ci

on:
  pull_request:
    paths: ["src/**", ".github/workflows/ci.yml"]
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ci-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

env:
  TOOL_VERSION: "1.2.3"

jobs:
  build:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@<sha> # v4.2.2
        with:
          persist-credentials: false
      - run: make
```

## Triggers

```yaml
on:
  push:
    branches: [main, "release/**"]
    tags: ["v*.*.*"]
    paths-ignore: ["**/*.md"]
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    paths: ["src/**"]
  schedule:
    - cron: "0 6 * * 1-5"        # UTC
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [dev, prod]
  workflow_call:
  release:
    types: [published]
```

## Job keys

```yaml
jobs:
  my_job:
    name: display name
    runs-on: ubuntu-24.04
    needs: [other_job]
    if: github.ref == 'refs/heads/main'
    timeout-minutes: 15
    permissions:
      contents: read
      id-token: write
    env: { KEY: value }
    outputs:
      result: ${{ steps.s.outputs.value }}
    strategy:
      fail-fast: false
      max-parallel: 2
      matrix:
        env: [dev, prod]
    container:
      image: python:3.12
    services:
      postgres:
        image: postgres:16
    steps: [...]
```

## Step keys

```yaml
steps:
  - name: label
    id: step_id
    uses: owner/action@<sha>
    with: { key: value }
    env: { KEY: value }
    if: always()
    continue-on-error: true
    timeout-minutes: 5
    working-directory: src
    shell: bash
    run: echo hi
```

## Permissions

```yaml
permissions:
  contents: read              # unlisted scopes become none
  id-token: write             # OIDC
  pull-requests: write
  packages: write
  actions: read
  checks: write
  security-events: write
```

```yaml
permissions: {}               # nothing
permissions: read-all
```

## Contexts

```yaml
github.repository      github.ref          github.ref_name
github.sha             github.actor        github.event_name
github.run_id          github.run_number   github.token
github.event.pull_request.number
github.workspace

env.NAME               vars.NAME           secrets.NAME
steps.<id>.outputs.<k> steps.<id>.outcome
needs.<job>.outputs.<k> needs.<job>.result
runner.os              runner.temp         runner.arch
matrix.<key>           inputs.<key>        job.status
```

## Expressions

```yaml
if: github.event_name == 'push'
if: startsWith(github.ref, 'refs/tags/v')
if: contains(github.ref, 'release')
if: vars.AWS_PLAN_ROLE_ARN != ''
if: github.event.pull_request.draft == false
if: always()        # also: success() failure() cancelled()
```

```yaml
format('{0}-{1}', a, b)    join(list, ', ')
toJSON(x)                  fromJSON(s)
hashFiles('**/lock')       contains() startsWith() endsWith()
```

## Default environment variables

```bash
GITHUB_WORKSPACE     GITHUB_REPOSITORY    GITHUB_REF
GITHUB_SHA           GITHUB_ACTOR         GITHUB_RUN_ID
GITHUB_EVENT_NAME    RUNNER_TEMP          RUNNER_OS
CI                   # always "true"

# Write to these FILES:
GITHUB_ENV           # vars for later steps
GITHUB_OUTPUT        # step outputs
GITHUB_PATH          # add to PATH
GITHUB_STEP_SUMMARY  # Markdown summary
```

```yaml
- run: echo "version=1.2.3" >> "$GITHUB_OUTPUT"
- run: echo "$HOME/bin" >> "$GITHUB_PATH"
- run: echo "### Done" >> "$GITHUB_STEP_SUMMARY"
```

## Artifacts and cache

```yaml
- uses: actions/upload-artifact@<sha>
  if: always()
  with:
    name: results-${{ matrix.env }}
    path: |
      reports/
      *.xml
    retention-days: 7
    if-no-files-found: error

- uses: actions/download-artifact@<sha>
  with: { name: results-dev }

- uses: actions/cache@<sha>
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

## OIDC to AWS

```yaml
permissions:
  contents: read
  id-token: write

steps:
  - uses: aws-actions/configure-aws-credentials@<sha>
    with:
      role-to-assume: ${{ vars.AWS_ROLE_ARN }}
      role-session-name: ci-${{ github.run_id }}
      aws-region: eu-central-1
```

Trust policy must pin the repo with `StringEquals`:

```
repo:OWNER/REPO:pull_request
repo:OWNER/REPO:ref:refs/heads/main
repo:OWNER/REPO:environment:production
```

## Reuse

```yaml
# Reusable workflow — called as a JOB
jobs:
  plan:
    uses: org/shared/.github/workflows/plan.yml@v2.3.1
    with: { environment: dev }
    secrets: { token: ${{ secrets.TOKEN }} }
    permissions:
      contents: read
      id-token: write
```

```yaml
# Composite action — called as a STEP
- uses: ./.github/actions/setup-tools
  with: { version: "1.2.3" }
```

Composite actions need `shell:` on every `run`.

## Workflow commands

```yaml
- run: |
    echo "::notice file=a.tf,line=10::message"
    echo "::warning::message"
    echo "::error::message"
    echo "::group::Section"
    echo "::endgroup::"
    echo "::add-mask::$COMPUTED"
```

## Safe shell

```yaml
- run: |
    set -euo pipefail        # default bash has -e but NOT pipefail
    cmd_one | tee out.txt
```

## Script injection — the rule

```yaml
# NEVER
- run: echo "${{ github.event.pull_request.title }}"

# ALWAYS
- run: echo "$TITLE"
  env:
    TITLE: ${{ github.event.pull_request.title }}
```

Untrusted: PR/issue titles and bodies, comments, branch names, commit messages,
`github.actor`.

## Safety rules

1. Pin every action to a **commit SHA**, with the tag in a comment.
2. Never interpolate user data into `run:` — use `env:`.
3. `permissions: contents: read` at workflow level; raise per job.
4. `persist-credentials: false` on every checkout.
5. OIDC, not stored cloud keys.
6. Never check out PR code under `pull_request_target`.
7. Never use self-hosted runners on public repositories.
8. `set -euo pipefail` in multi-command `run:` blocks.
9. `if: always()` on artifact uploads and summaries.
10. `timeout-minutes` on every job (default is 360).
11. Verify checksums on downloaded binaries.
12. Enable Dependabot for `github-actions`.

## Validate

```bash
actionlint .github/workflows/*.yml        # best — catches shell + injection
python -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))"
act pull_request -W .github/workflows/ci.yml   # approximate local run
```

Debug logging: repository secrets `ACTIONS_STEP_DEBUG=true`,
`ACTIONS_RUNNER_DEBUG=true`.

## Quick diagnosis

| Symptom | Likely cause |
|---------|--------------|
| Workflow never ran | Wrong path, bad YAML, `on:` mismatch |
| Job skipped | `if:` false, or a `needs:` job failed |
| "No such file or directory" | Missing `actions/checkout` |
| Works locally, fails in CI | Tool not installed on the runner |
| Pipe failure reported as success | Missing `set -o pipefail` |
| Secret empty | Fork PR, wrong scope, or typo |
| `Resource not accessible` | `permissions:` too narrow |
| OIDC denied | Missing `id-token: write`, or `sub` mismatch |
| Artifact missing after failure | Needs `if: always()` |
| Queued forever | No runner matches `runs-on` labels |
| Cancelled unexpectedly | `cancel-in-progress: true` |
| Required check stuck pending | Skipped by a path filter |

---

Previous: [15 — GitHub Actions vs GitLab CI](15-github-vs-gitlab.md) ·
Next: [17 — Hands-on labs](17-hands-on-labs.md) ·
Back to the [index](README.md)
