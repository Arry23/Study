# 10 — Matrix and concurrency

Two independent controls: one multiplies jobs, the other limits them.

## Matrix

Run the same job across several configurations, in parallel:

```yaml
jobs:
  plan:
    strategy:
      fail-fast: false
      matrix:
        env: [dev]
    steps:
      - name: terraform plan
        env:
          TF_ENV: ${{ matrix.env }}
        run: terraform -chdir="envs/$TF_ENV" plan
```

One entry today. Adding environments is a one-line change:

```yaml
matrix:
  env: [dev, staging, prod]
```

Three jobs, in parallel, each planning its own root. That is the payoff for
designing the matrix before you need it.

## Multiple dimensions

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    python: ["3.11", "3.12", "3.13"]
```

The **cross product** — nine jobs. Grows fast; watch your runner minutes.

## `fail-fast`

```yaml
strategy:
  fail-fast: false        # default is true
```

With `true`, one failing variant cancels the rest. Fast feedback, but you only
learn about the first failure.

With `false`, every variant runs to completion. You learn that dev and prod both
fail, which is a different problem from only dev failing.

> For an environment matrix, `fail-fast: false` is nearly always right —
> `arifa-infra` sets it. Independent targets should report independently.

## `max-parallel`

```yaml
strategy:
  max-parallel: 2
```

Caps simultaneous jobs. Useful when variants contend for a shared resource — a
test database, an API rate limit, a Terraform state lock.

## `include` and `exclude`

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    python: ["3.11", "3.12"]
    exclude:
      - os: windows-latest
        python: "3.11"
    include:
      - os: ubuntu-latest
        python: "3.13"
        experimental: true
```

`exclude` removes combinations. `include` adds a variant, or adds a property to
matching existing ones.

```yaml
- run: ./test.sh
  continue-on-error: ${{ matrix.experimental == true }}
```

## Dynamic matrices

Build one at run time from JSON:

```yaml
jobs:
  discover:
    runs-on: ubuntu-latest
    outputs:
      envs: ${{ steps.find.outputs.envs }}
    steps:
      - uses: actions/checkout@<sha>
      - id: find
        run: echo "envs=$(ls -d envs/*/ | xargs -n1 basename | jq -R . | jq -sc .)" >> "$GITHUB_OUTPUT"

  plan:
    needs: discover
    strategy:
      fail-fast: false
      matrix:
        env: ${{ fromJSON(needs.discover.outputs.envs) }}
    steps:
      - run: terraform -chdir="envs/$TF_ENV" plan
        env:
          TF_ENV: ${{ matrix.env }}
```

For `arifa-infra` this would discover `envs/` automatically, so adding
`envs/staging/` needs no workflow edit. Worth it once there are several
environments; over-engineering for one.

## Naming matrix jobs

```yaml
jobs:
  plan:
    name: plan (${{ matrix.env }})
```

Without this, the checks list shows several identically named jobs.

## Concurrency

Limits how many runs of a workflow may be active.

```yaml
concurrency:
  group: terraform-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

Runs sharing a **group** do not run simultaneously. Push twice to a PR and the
first run is cancelled.

Benefits: saves minutes, and prevents an older run finishing after a newer one
and reporting a stale result.

### Choosing the group key

```yaml
group: ${{ github.workflow }}-${{ github.ref }}          # per workflow per branch
group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
group: deploy-production                                  # global — one at a time
```

> `arifa-infra` keys on the **PR number**, so it serialises within one PR only.
> Two different PRs still plan at once and contend for the same Terraform state
> lock, which is why the plan command carries `-lock-timeout=5m`.
>
> Serialising per environment instead would remove the contention:
>
> ```yaml
> concurrency:
>   group: terraform-plan-dev
>   cancel-in-progress: false
> ```
>
> at the cost of queuing PRs behind each other.

### `cancel-in-progress`

```yaml
cancel-in-progress: true    # PR validation — newest wins
cancel-in-progress: false   # deployments — let them finish
```

> **Never cancel a deployment mid-flight.** A cancelled `terraform apply` can
> leave resources half-created and a lock held. For deploy workflows use
> `cancel-in-progress: false` and let runs queue.

### Job-level concurrency

```yaml
jobs:
  deploy:
    concurrency:
      group: deploy-${{ matrix.env }}
      cancel-in-progress: false
```

Lets you cancel cheap validation while serialising expensive deployments in the
same workflow.

## Combining them

```yaml
concurrency:
  group: terraform-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  plan:
    strategy:
      fail-fast: false
      max-parallel: 1              # one environment at a time
      matrix:
        env: [dev, staging, prod]
    concurrency:
      group: tf-state-${{ matrix.env }}
      cancel-in-progress: false
```

Workflow concurrency cancels superseded PR runs; job concurrency protects each
environment's state lock; `max-parallel: 1` avoids three plans at once.

## Costs

Matrix multiplies runner minutes. Three OSes by three versions is nine jobs — on
a private repository that is nine times the cost.

Trim with:

- `exclude` for combinations nobody uses
- Full matrix only on `main` or nightly; one representative variant on PRs

```yaml
matrix:
  python: ${{ github.event_name == 'pull_request' && fromJSON('["3.12"]') || fromJSON('["3.11","3.12","3.13"]') }}
```

## Try it

In [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. How many jobs does the matrix produce today?
2. Why `fail-fast: false` with a single entry?
3. What does the concurrency group serialise, and what does it not?
4. Why does the plan command need `-lock-timeout=5m`?
5. Write the job-level concurrency that would remove cross-PR lock contention.

## Check yourself

1. What does a two-dimensional matrix produce?
2. When is `fail-fast: false` the right choice?
3. What does `max-parallel` protect?
4. How do you build a matrix at run time?
5. When must `cancel-in-progress` be `false`?
6. Why name matrix jobs explicitly?

---

Previous: [09 — Security hardening](09-security-hardening.md) ·
Next: [11 — Artifacts, caching, summaries](11-artifacts-caching-summaries.md)
