# 14 — Troubleshooting

## Where to look

1. **Actions tab → the run → the job** — full logs per step
2. **Re-run with debug logging** — "Re-run jobs" → enable debug
3. **`actionlint`** — catches most mistakes before pushing
4. **Settings → Actions → Runners** — availability
5. **Settings → Secrets and variables** — what is actually set

---

## Nothing runs

### No workflow triggered

- **Wrong location** — must be `.github/workflows/*.yml` at the repository root
- **Invalid YAML** — GitHub shows the error in the Actions tab
- **`on:` does not match** — check the event and any `branches`/`paths` filters
- **Actions disabled** for the repository or organisation
- **Scheduled workflows** only run on the default branch, and are disabled after
  60 days of inactivity
- **Fork PR** — some events are restricted

### Path filter excluded it

```yaml
on:
  pull_request:
    paths:
      - "**/*.tf"
```

A PR touching only `README.md` matches nothing. Intended — but note a **required**
status check that is skipped stays pending and blocks merge. Either do not mark
it required, or add a tiny always-runs job that reports success.

### A job was skipped

Its `if:` was false, or something in `needs:` failed or was skipped.

```yaml
plan:
  needs: check
  if: vars.AWS_PLAN_ROLE_ARN != ''
```

Skipped jobs show as "Skipped", not failed. To run regardless:

```yaml
if: always()
```

---

## Step failures

### `No such file or directory` for your own code

Missing checkout:

```yaml
- uses: actions/checkout@<sha>
```

The most common first-workflow error. GitHub does not clone automatically.

### Works locally, fails in CI

The runner has a different toolset. Install explicitly:

```yaml
- uses: actions/setup-python@<sha>
  with:
    python-version: "3.12"
```

On a self-hosted runner, assume nothing. `arifa-infra`'s runner has no `make`,
which is why the workflow calls `scripts/check.sh` directly.

### `cd` did not persist

Each `run` is a separate shell:

```yaml
- run: |              # one shell
    cd src
    make
```

```yaml
- run: make
  working-directory: src
```

### A failing pipeline reports success

Default `bash` sets `-e` but **not** `-o pipefail`:

```yaml
- run: terraform plan | tee plan.txt      # tee's exit code wins
```

```yaml
- run: |
    set -o pipefail
    terraform plan | tee plan.txt
```

Exactly why `arifa-infra` sets it explicitly in the plan step.

### Permission denied

```yaml
- run: ./scripts/build.sh
```

Commit the executable bit (`git update-index --chmod=+x`), or:

```yaml
- run: bash scripts/build.sh
```

---

## Secrets and permissions

### A secret is empty

- Not set, or misspelled — `secrets.API_TOKEN` vs `secrets.API_token`
- **Fork PR** — secrets are deliberately withheld
- Set at environment scope but the job declares no `environment:`
- Secrets are **not** available in job-level `if:`

Check without leaking:

```yaml
- run: '[ -n "$TOKEN" ] || { echo "TOKEN empty"; exit 1; }'
  env:
    TOKEN: ${{ secrets.API_TOKEN }}
```

### `Resource not accessible by integration`

`GITHUB_TOKEN` lacks a permission:

```yaml
permissions:
  contents: read
  pull-requests: write      # add what you need
```

### OIDC: `Not authorized to perform sts:AssumeRoleWithWebIdentity`

In order of likelihood:

1. **Missing `id-token: write`** on the job
2. **Trust policy `sub` does not match** — compare the real claim in CloudTrail
3. **Wrong `aud`** — should be `sts.amazonaws.com`
4. **Role ARN wrong** — check `vars.AWS_PLAN_ROLE_ARN`

`arifa-infra` documents a real variant:

```hcl
# some organisations use the immutable-ID form
# repo:<owner>@<ownerId>/<repo>@<repoId>:pull_request
```

### `Credentials could not be loaded`

The AWS step did not run, or ran in a different job. Credentials do not cross
jobs.

---

## Expressions

### An expression printed literally

```yaml
- run: echo "${{ github.ref }}"       # evaluated
- run: echo '${{ github.ref }}'       # also evaluated — quotes do not stop it
```

If you see `${{ ... }}` in output, it is probably in a context where expressions
are not evaluated — most `if:` positions are fine, but some keys do not support
them.

### `if:` always true

```yaml
if: ${{ github.event_name == 'push' }}     # fine
if: "false"                                 # a non-empty STRING — truthy!
```

Use real booleans and comparisons.

### Unexpected symbol

An expression starting a YAML value needs quoting:

```yaml
name: "${{ matrix.env }} plan"
```

---

## Jobs and data

### Job B cannot see job A's files

Different machines. Use artifacts:

```yaml
- uses: actions/upload-artifact@<sha>
  with: { name: out, path: dist/ }

- uses: actions/download-artifact@<sha>
  with: { name: out }
```

### The artifact is missing after a failure

```yaml
- uses: actions/upload-artifact@<sha>
  if: always()
```

### `No files were found with the provided path`

Wrong path, or the producing step failed. Make it loud:

```yaml
with:
  if-no-files-found: error
```

### Matrix jobs overwrite each other's artifacts

```yaml
with:
  name: results-${{ matrix.env }}
```

---

## Caching

### Cache never hits

- Key changes every run — do not include the SHA
- Different `runner.os` than when it was saved
- First run on a branch with no `restore-keys` fallback
- Evicted: 10 GB repository limit, least recently used

```yaml
restore-keys: |
  ${{ runner.os }}-pip-
```

### Stale cache breaks the build

Change the key prefix to invalidate:

```yaml
key: ${{ runner.os }}-pip-v2-${{ hashFiles('**/requirements.txt') }}
```

---

## Runners

### Queued forever

No runner matches `runs-on`. Check every label, that a runner is online, that it
is permitted for this repository, and that slots are free. A label typo fails
silently.

### Cancelled unexpectedly

```yaml
concurrency:
  cancel-in-progress: true
```

A newer run superseded it. Correct for PR validation, wrong for deployments.

---

## Debugging technique

**Enable debug logs.** Repository secrets:

```
ACTIONS_STEP_DEBUG=true
ACTIONS_RUNNER_DEBUG=true
```

**Dump a context safely:**

```yaml
- run: echo "$CTX"
  env:
    CTX: ${{ toJSON(github) }}
```

Never `toJSON(secrets)`.

**Lint before pushing:**

```bash
actionlint .github/workflows/terraform.yml
```

**Run locally with `act`** — approximate, but a fast loop:

```bash
act pull_request -W .github/workflows/terraform.yml
```

Self-hosted runner behaviour and OIDC will not reproduce.

**Add a temporary diagnostic job:**

```yaml
debug:
  runs-on: ubuntu-latest
  steps:
    - run: |
        echo "event=$GITHUB_EVENT_NAME ref=$GITHUB_REF"
        echo "actor=$GITHUB_ACTOR"
```

**Re-run a single failed job** rather than the whole workflow.

**Bisect.** Reduce a failing step to `echo ok`, confirm, add back.

---

## Before asking for help

1. Link to the failing run
2. The error, verbatim
3. What changed since it last worked
4. Whether it fails on other branches or only this one
5. Whether `actionlint` is clean

## Check yourself

1. Workflow did not trigger — first three checks?
2. Why does "No such file or directory" usually mean a missing checkout?
3. What does default `bash` not set, and what breaks because of it?
4. Three causes of an OIDC `AssumeRoleWithWebIdentity` denial?
5. Why is the artifact missing after a failed job?
6. Why can a path-filtered required check block a merge?

---

Previous: [13 — Runners](13-runners.md) ·
Next: [15 — GitHub Actions vs GitLab CI](15-github-vs-gitlab.md)
