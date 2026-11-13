# 07 — needs and the DAG

## The cost of stages

Stages are barriers. Every job in stage N must finish before any job in stage
N+1 starts — even when there is no real dependency.

`main/.gitlab-ci.yml` has ten sequential stages:

```
submodule → format → trufflehog → security → pytest-coverage
          → sonarqube → sonar-report → dependency-scanning-job
          → image-build → package
```

So `trufflehog` (a secret scan of git history) waits for `yapf_format` (a Python
formatting check), despite having nothing to do with it. If formatting takes two
minutes, the secret scan starts two minutes late, and everything after it
inherits the delay.

Worse, `yapf_format` has `allow_failure: true` — so the pipeline waits for a job
whose result cannot even block it.

## `needs` turns the pipeline into a graph

```yaml
trufflehog:
  stage: trufflehog
  needs: []                  # start immediately
```

`needs: []` means "no dependencies" — the job starts as soon as the pipeline
does, ignoring stage order entirely.

```yaml
sonar-report:
  needs:
    - job: sonarqube-scan
    - job: submodule
      artifacts: true
```

This is already in your pipeline. `sonar-report` starts the moment
`sonarqube-scan` and `submodule` are done, rather than waiting for the whole
`dependency-scanning-job` ordering to resolve. `artifacts: true` says it also
wants the submodule job's files.

That turns a chain into a **DAG** — a directed acyclic graph.

## Before and after

Sequential stages:

```
submodule ──► format ──► trufflehog ──► security ──► pytest ──► sonar
  30s          120s         60s           90s         300s      120s
                                                      total ≈ 720s
```

With `needs`:

```
submodule ──┬─► format ──────────────┐
  30s       │    120s                │
            ├─► trufflehog           ├─► sonarqube ──► sonar-report
            │     60s                │      120s           60s
            ├─► semgrep              │
            │     90s                │
            └─► pytest ──────────────┘
                 300s
                                     total ≈ 30 + 300 + 120 + 60 ≈ 510s
```

The critical path is now the slowest chain, not the sum of all stages.

## Rules

**Only earlier jobs.** A job can only need jobs that would run before it, or in
the same stage.

**Maximum 50 per job.**

**No cycles.** A needs B needs A is rejected at pipeline creation.

**`needs` replaces `dependencies`.** When present, artifacts come from the
`needs` list, controlled with `artifacts: true|false`.

## Artifact control

```yaml
job:
  needs:
    - job: build
      artifacts: true        # wait for it AND download its artifacts
    - job: lint
      artifacts: false       # wait for it, skip the download
```

Useful when a job must wait for a slow job but does not need its 500 MB output.

## Optional needs

```yaml
deploy:
  needs:
    - job: build
    - job: integration_tests
      optional: true         # if it was excluded by rules, do not fail
```

Without `optional: true`, needing a job that `rules` excluded fails pipeline
creation. Essential when the needed job is conditional.

## What you could do here

A reasonable `needs` pass over `main/.gitlab-ci.yml`:

```yaml
trufflehog:
  needs: []                       # scans git history — needs nothing

semgrep-sast:
  needs: []                       # scans source — needs nothing

yapf_format:
  needs:
    - job: submodule
      artifacts: true

pre-commit-hooks:
  needs:
    - job: submodule
      artifacts: true

pytest-coverage:
  needs:
    - job: submodule
      artifacts: true

sonarqube-scan:
  needs:
    - job: pytest-coverage
      artifacts: true           # needs coverage.xml

image-build:
  needs:
    - job: submodule
      artifacts: true
```

`trufflehog` and `semgrep-sast` then start immediately instead of waiting
through two stages, and `image-build` no longer waits for scans it does not
depend on.

> Keep the `stages` list even when using `needs`. It still groups the pipeline
> visually, and jobs without `needs` continue to follow it.

## Parent–child pipelines

For very large pipelines, split into child pipelines:

```yaml
backend:
  trigger:
    include: backend/.gitlab-ci.yml
    strategy: depend        # parent waits for the child

frontend:
  trigger:
    include: frontend/.gitlab-ci.yml
```

Each child is its own pipeline with its own stages. Good for monorepos, and it
pairs well with `rules:changes` so only affected components run.

Without `strategy: depend` the parent fires and forgets.

## Multi-project pipelines

Trigger a pipeline in a different project:

```yaml
deploy:
  trigger:
    project: ocean/deployment-pipelines
    branch: main
    strategy: depend
```

## `.pre` and `.post`

Two built-in stages that always exist:

```yaml
notify_start:
  stage: .pre               # before everything
  script: [./notify.sh started]

notify_end:
  stage: .post              # after everything, even on failure
  script: [./notify.sh finished]
```

You do not list them in `stages:`. Useful for setup, notifications and cleanup.

## Visualising it

A pipeline page has three views:

- **Stages** — the classic column view
- **Needs** — the DAG, showing real dependencies
- **Job dependencies** — the same with artifact flow

The Needs view immediately shows accidental serialisation.

## Try it

On [`main/.gitlab-ci.yml`](../main/.gitlab-ci.yml):

1. Which jobs genuinely need `submodule`'s artifacts? Check which ones `cp` from
   `ocean-cdr-coralreef-common`.
2. Which jobs need nothing at all?
3. `sonar-report` already uses `needs`. What does `artifacts: true` buy it?
4. Sketch the DAG. What is the critical path?
5. Which single `needs: []` would save the most wall-clock time?

## Check yourself

1. What problem does `needs` solve?
2. What does `needs: []` mean?
3. How does `needs` change artifact downloading?
4. When do you need `optional: true`?
5. Why keep `stages` when using `needs`?
6. What does `strategy: depend` do on a trigger job?

---

Previous: [06 — Artifacts and cache](06-artifacts-and-cache.md) ·
Next: [08 — Reuse: include, extends, anchors](08-reuse-include-extends.md)
