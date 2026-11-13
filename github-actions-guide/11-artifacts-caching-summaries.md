# 11 — Artifacts, caching, summaries

Three ways to get data out of a job.

| | Purpose | Guaranteed |
|---|---|---|
| Artifacts | Pass files between jobs; download later | Yes |
| Cache | Reuse downloads for speed | No |
| Job summary | Render Markdown in the run UI | Yes |

## Artifacts

Each job runs on its own machine, so files do not carry over. Artifacts bridge
that.

```yaml
- uses: actions/upload-artifact@<sha>
  with:
    name: terraform-plan
    path: plan.txt
    retention-days: 5
```

```yaml
- uses: actions/download-artifact@<sha>
  with:
    name: terraform-plan
```

> Unlike GitLab, nothing is passed automatically. You upload and download
> explicitly — more verbose, but there is no mystery about where a file came
> from.

### Options

```yaml
- uses: actions/upload-artifact@<sha>
  if: always()                 # keep evidence from failures
  with:
    name: results-${{ matrix.env }}
    path: |
      reports/
      *.xml
    retention-days: 7
    if-no-files-found: error   # warn (default) | error | ignore
    overwrite: true
```

Two habits worth forming:

**`if: always()`** — otherwise the artifact is skipped exactly when the job
failed and you need it most.

**`if-no-files-found: error`** — the default silently uploads nothing when a
path is wrong, and you discover it when the consumer fails.

**Unique names in a matrix.** Two jobs uploading `results` collide; use
`results-${{ matrix.env }}`.

### Saving a plan for apply

```yaml
jobs:
  plan:
    steps:
      - run: terraform plan -out=tfplan
      - uses: actions/upload-artifact@<sha>
        with:
          name: tfplan
          path: tfplan

  apply:
    needs: plan
    environment: production
    steps:
      - uses: actions/download-artifact@<sha>
        with:
          name: tfplan
      - run: terraform apply tfplan
```

Applying a **saved plan** means approving exactly what gets applied. Re-planning
at apply time means approving one thing and applying another.

`arifa-infra` does not apply from CI, so it writes the plan to the job summary
instead.

## Caching

```yaml
- uses: actions/cache@<sha>
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

**`key`** — exact match restores and skips saving at the end.

**`restore-keys`** — prefix fallbacks. A partial hit is usually better than
nothing: pip updates a few packages instead of downloading all.

`hashFiles` is the important part. The key changes only when dependencies
change, so the cache is correct by construction.

### Setup actions cache for you

```yaml
- uses: actions/setup-python@<sha>
  with:
    python-version: "3.12"
    cache: pip
```

```yaml
- uses: actions/setup-node@<sha>
  with:
    node-version: "20"
    cache: npm
```

Prefer this over hand-rolled caching.

### What to cache

| Cache | Do not cache |
|-------|--------------|
| `~/.cache/pip`, `~/.npm`, `~/.m2` | Build output other jobs need (artifact) |
| `node_modules` (keyed on the lockfile) | Anything secret |
| Compiler caches | Anything that must be fresh |

### Limits and scope

- 10 GB per repository; least-recently-used entries evicted
- Caches from a branch are visible to that branch and its children
- A PR branch can read `main`'s cache but writes its own

> A cache is best effort. A job must work with an empty one.

### Terraform plugin cache

```yaml
env:
  TF_PLUGIN_CACHE_DIR: ${{ runner.temp }}/terraform-plugins

- uses: actions/cache@<sha>
  with:
    path: ${{ runner.temp }}/terraform-plugins
    key: tf-plugins-${{ hashFiles('**/.terraform.lock.hcl') }}
```

Keyed on the lock file, so it invalidates exactly when providers change.
`arifa-infra` does not do this — provider downloads are re-run every time.

## Job summaries

Markdown rendered on the run page:

```yaml
- name: Write the plan to the job summary
  if: always()
  env:
    TF_ENV: ${{ matrix.env }}
  run: |
    plan="$RUNNER_TEMP/plan.txt"
    {
      echo "### terraform plan: envs/${TF_ENV}"
      echo
      if [ -f "$plan" ]; then
        grep -E '^(Plan:|No changes\.|Error:)' "$plan" || echo "No plan summary line; see the full output."
        echo
        echo "<details><summary>Full plan</summary>"
        echo
        echo '```'
        head -c 60000 "$plan"
        echo
        echo '```'
        echo "</details>"
      else
        echo "No plan output: init or authentication failed. See the step logs."
      fi
    } >> "$GITHUB_STEP_SUMMARY"
```

Worth studying — several good decisions:

**`if: always()`** so a failed plan still reports.

**Summary line first.** `grep -E '^(Plan:|No changes\.|Error:)'` surfaces the
one line a reviewer needs; the rest is collapsed in `<details>`.

**`head -c 60000`** caps the output. The limit is 1 MiB per step.

**A fallback message** when the file is missing, distinguishing "no changes"
from "it never ran".

**`|| echo`** so a missing summary line does not fail the step under `set -e`.

### Why a summary and not a PR comment

Commenting needs `pull-requests: write`. The summary needs nothing. Given the
workflow's restrictive `permissions: contents: read`, this keeps the token
read-only.

The trade-off: a reviewer must open the run rather than seeing it inline.

> Plan output can contain resource names and configuration. Terraform masks
> values marked sensitive, but anyone with repository read access can read the
> summary. Worth a deliberate decision.

### Other summary uses

```yaml
- if: always()
  run: |
    {
      echo "## Test results"
      echo "| Suite | Result |"
      echo "|-------|--------|"
      echo "| unit  | ${{ steps.unit.outcome }} |"
    } >> "$GITHUB_STEP_SUMMARY"
```

## Workflow commands

Annotations that appear inline on the PR diff:

```yaml
- run: |
    echo "::notice file=main.tf,line=10::Consider pinning this version"
    echo "::warning file=main.tf,line=20::Deprecated syntax"
    echo "::error file=main.tf,line=30::Invalid configuration"
```

Grouping and masking:

```yaml
- run: |
    echo "::group::Dependency install"
    pip install -r requirements.txt
    echo "::endgroup::"

    echo "::add-mask::$COMPUTED_SECRET"
```

`::add-mask::` is useful for a value derived at run time that GitHub cannot know
is sensitive.

## Choosing

```
Needed by another job?            -> artifact
Downloadable after the run?       -> artifact
Just makes the build faster?      -> cache
Something a human should read?    -> job summary
Tied to a file or line?           -> annotation
```

## Try it

In [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. It uses no artifacts and no cache. What would benefit from each?
2. Why does the summary step use `if: always()`?
3. Why is the summary line grepped out before the full plan?
4. Why `head -c 60000`?
5. Why a job summary rather than a PR comment?
6. Write the `actions/cache` step for the Terraform plugin cache.

## Check yourself

1. Artifact vs cache — one sentence.
2. Why does GitHub not pass artifacts automatically?
3. Why `if: always()` on an upload?
4. What does `restore-keys` do?
5. Why key a cache with `hashFiles`?
6. Which permission does a PR comment need that a summary does not?

---

Previous: [10 — Matrix and concurrency](10-matrix-and-concurrency.md) ·
Next: [12 — Reusable workflows and composite actions](12-reusable-workflows.md)
