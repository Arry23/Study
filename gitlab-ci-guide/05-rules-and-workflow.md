# 05 — Rules and workflow

Deciding *when* things run is most of pipeline engineering.

Two levels:

- **`workflow:rules`** — should a pipeline exist at all?
- **job `rules`** — should this job be in it?

## `workflow:rules`

Evaluated once, before anything else:

```yaml
workflow:
  rules:
    - if: ($CI_PIPELINE_SOURCE == 'merge_request_event' || $CI_PIPELINE_SOURCE == 'push') || $CI_COMMIT_TAG
```

Merge requests, pushes and tags create a pipeline. Schedules, API calls and
manual web runs do not.

Rules are evaluated **top to bottom, first match wins**. No match means no
pipeline.

### Avoiding duplicate pipelines

The classic annoyance: you push to a branch that has an open MR, and get two
pipelines — one `push`, one `merge_request_event`. The standard fix:

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH
```

Read it as: MR pipelines always run; a branch push that already has an open MR
does not; any other branch push does.

## Job `rules`

```yaml
sonar-report:
  rules:
    - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
      when: never
    - if: $CI_PIPELINE_SOURCE == 'push' && $CI_COMMIT_BRANCH == 'develop'
      when: on_success
```

First match wins, so order is the logic:

1. On an MR — **never** run.
2. On a push to `develop` — run if earlier stages passed.
3. Neither matched — job is excluded.

That is exactly right for publishing a report: you do not want every MR
rewriting the shared report repository.

### `when`

| Value | Effect |
|-------|--------|
| `on_success` | Run if all prior stages passed (default) |
| `on_failure` | Run only if something earlier failed |
| `always` | Run regardless |
| `never` | Do not add the job |
| `manual` | Add it, but require a click |
| `delayed` | Run after `start_in:` |

```yaml
deploy_prod:
  when: manual
  allow_failure: false     # pipeline waits; without this a manual job is optional
```

`allow_failure: false` on a manual job is the difference between a blocking
approval gate and an optional button.

### `exists`

Run only when a file is present:

```yaml
submodule:
  rules:
    - exists:
        - .gitmodules
```

Elegant: the submodule job disappears entirely in a repository with no
submodules, rather than failing or needing a flag.

### `changes`

Run only when certain paths changed:

```yaml
docs:
  rules:
    - changes:
        - docs/**/*
        - "*.md"
```

> Reliable on merge request pipelines, where GitLab knows the full diff. On
> branch pipelines it compares against the previous commit, which is misleading
> after a force-push or a squash. Prefer `changes` on MR pipelines.

### Regex matching

```yaml
rules:
  - if: '$CI_COMMIT_TAG !~ "/^$/"'
```

"The tag does not match empty" — that is, this is a tag pipeline. Used for both
`image-build` and `env-package`, which should only run on releases.

A clearer equivalent:

```yaml
rules:
  - if: $CI_COMMIT_TAG
```

and for semantic version tags only:

```yaml
rules:
  - if: '$CI_COMMIT_TAG =~ /^v?\d+\.\d+\.\d+$/'
```

### Variables inside rules

A rule can set variables for the job it matches:

```yaml
deploy:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      variables:
        ENVIRONMENT: production
    - if: $CI_COMMIT_BRANCH == "develop"
      variables:
        ENVIRONMENT: staging
  script:
    - ./deploy.sh "$ENVIRONMENT"
```

Cleaner than branching inside the script.

## `only` / `except` — legacy

You will meet these in older pipelines:

```yaml
job:
  only:
    - main
    - tags
  except:
    - schedules
```

They still work but cannot be combined with `rules` in the same job, and
`rules` is strictly more expressive. **Use `rules` in new code.**

Rough translation:

| Old | New |
|-----|-----|
| `only: [main]` | `rules: [{ if: '$CI_COMMIT_BRANCH == "main"' }]` |
| `only: [tags]` | `rules: [{ if: $CI_COMMIT_TAG }]` |
| `only: [merge_requests]` | `rules: [{ if: '$CI_PIPELINE_SOURCE == "merge_request_event"' }]` |
| `except: [schedules]` | `rules: [{ if: '$CI_PIPELINE_SOURCE == "schedule"', when: never }, ...]` |

## Pipeline types

What `CI_PIPELINE_SOURCE` tells you:

| Source | Trigger |
|--------|---------|
| `push` | Commit pushed to a branch |
| `merge_request_event` | MR opened or updated |
| `schedule` | Scheduled pipeline |
| `web` | "Run pipeline" button |
| `api` | API call |
| `trigger` | Trigger token |
| `pipeline` | Another pipeline |
| `parent_pipeline` | Parent/child pipeline |

## Common recipes

**Only on the default branch**

```yaml
rules:
  - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

**Only on MRs**

```yaml
rules:
  - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

**Only on release tags**

```yaml
rules:
  - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'
```

**Manual deploy on main, never elsewhere**

```yaml
rules:
  - if: $CI_COMMIT_BRANCH == "main"
    when: manual
    allow_failure: false
  - when: never
```

**Nightly only**

```yaml
rules:
  - if: $CI_PIPELINE_SOURCE == "schedule"
```

**Skip drafts**

```yaml
rules:
  - if: '$CI_MERGE_REQUEST_TITLE =~ /^Draft:/'
    when: never
  - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

## Debugging rules

Rules are evaluated server-side, so you cannot `echo` your way to the answer.

- **CI Lint** (CI/CD → Pipelines → CI Lint) has "Simulate pipeline creation"
- A pipeline's job list shows what was included; a missing job means no rule
  matched
- Temporarily add a job that prints the deciding variables:

```yaml
debug_rules:
  stage: .pre
  script:
    - echo "source=$CI_PIPELINE_SOURCE branch=$CI_COMMIT_BRANCH tag=$CI_COMMIT_TAG"
```

`.pre` and `.post` are built-in stages that always run first and last.

## Try it

From [`main/.gitlab-ci.yml`](../main/.gitlab-ci.yml):

1. Which pipeline sources create a pipeline?
2. Under exactly what conditions does `sonar-report` run?
3. Why does `submodule` use `exists:` instead of a variable flag?
4. Rewrite `'$CI_COMMIT_TAG !~ "/^$/"'` more readably.
5. `image-build` and `env-package` run only on tags. What would you add so they
   only run on `v1.2.3`-style tags?

## Check yourself

1. Difference between `workflow:rules` and job `rules`?
2. How are rules evaluated — all of them, or first match?
3. What does `when: never` do?
4. Why `allow_failure: false` on a manual job?
5. When is `changes:` unreliable?
6. Why prefer `rules` over `only`/`except`?

---

Previous: [04 — Variables](04-variables.md) ·
Next: [06 — Artifacts and cache](06-artifacts-and-cache.md)
