# 01 — What GitLab CI/CD is

## The problem it solves

Someone pushes a commit. Before it can be merged you want to know: does it
compile, do the tests pass, is it formatted, does it contain a leaked
credential, does it have known-vulnerable dependencies?

Doing that by hand is slow and inconsistent. CI/CD does it automatically, the
same way, every time, on a machine nobody has customised.

- **CI — continuous integration.** Every change is built and tested
  automatically.
- **CD — continuous delivery/deployment.** Changes that pass are packaged and
  shipped.

## The moving parts

```
  .gitlab-ci.yml          GitLab server            Runner
  (your pipeline)   ->    reads the file,     ->   actually executes
                          creates a pipeline       the jobs
```

**`.gitlab-ci.yml`** — a YAML file at the repository root. It defines the
pipeline. If it is absent or invalid, nothing runs.

**GitLab server** — watches for pushes, merge requests and tags. On a matching
event it parses the file and creates a **pipeline**.

**Runner** — a separate agent that picks up jobs and runs them. This is what
executes your `script`. Runners are usually Docker-based: each job gets a fresh
container.

## Pipeline, stage, job

```
pipeline                     one run, triggered by one event
├── stage: format            a named group; stages run in order
│   └── job: yapf_format     the unit of work — a shell script
├── stage: security
│   ├── job: semgrep-sast        jobs in the same stage
│   └── job: pre-commit-hooks    run in parallel
└── stage: pytest-coverage
    └── job: pytest-coverage
```

**Stages run in sequence. Jobs within a stage run in parallel.** By default a
stage starts only when every job in the previous stage has succeeded.

`main/.gitlab-ci.yml` declares ten stages:

```yaml
stages:
  - submodule
  - format
  - trufflehog
  - security
  - pytest-coverage
  - sonarqube
  - sonar-report
  - dependency-scanning-job
  - image-build
  - package
```

Read top to bottom, that is the whole lifecycle: fetch dependencies, check
formatting, scan for secrets, run security analysis, test with coverage, push
quality metrics, scan dependencies, build a container, publish artefacts.

## A job is a shell script in a container

This is the mental model that matters:

```yaml
yapf_format:
  stage: format
  image: python:3.11
  before_script:
    - pip install yapf
  script:
    - yapf --recursive --in-place --verbose .
```

GitLab starts a `python:3.11` container on a runner, clones your repository into
it, runs `pip install yapf`, then runs `yapf`. When the script finishes the
container is **destroyed**.

Consequences that trip up newcomers:

- **Each job is a fresh container.** Files written in one job are gone in the
  next, unless you pass them as artifacts (chapter 06).
- **Installed packages do not persist.** Every job that needs `yapf` installs it.
- **`cd` does not persist between jobs**, only between lines of one script.
- **The machine is not yours.** No local config, no SSH keys, nothing you did
  by hand.

## Success and failure

A job **passes if its script exits 0** and fails otherwise. That is the entire
rule.

```yaml
script:
  - |
    if git diff --name-only | grep -q .; then
      echo "Differences found"
      exit 1          # fails the job
    else
      echo "Code already formatted"
    fi
```

A failed job normally fails the pipeline and blocks the stages after it —
unless it is marked otherwise:

```yaml
allow_failure: true
```

That makes the job advisory: it can go red without stopping anything.

> `main/.gitlab-ci.yml` sets `allow_failure: true` on both `yapf_format` and
> `pytest-coverage`. So formatting violations and failing tests do **not** block
> the pipeline. That may be deliberate during a migration, but it means two of
> the most valuable gates cannot actually gate anything. Chapter 12.

## What triggers a pipeline

Pushes, merge requests, tags, schedules, manual runs, the API, or another
pipeline. You control which with `workflow:rules`:

```yaml
workflow:
  rules:
    - if: ($CI_PIPELINE_SOURCE == 'merge_request_event' || $CI_PIPELINE_SOURCE == 'push') || $CI_COMMIT_TAG
```

Merge requests, pushes and tags create pipelines here; anything else does not.
Chapter 05.

## Where you see the results

- **CI/CD → Pipelines** — every run and its status
- **A pipeline** — the stage/job graph; click any job for its log
- **Merge request** — a summary widget; a red pipeline usually blocks merge
- **CI/CD → Jobs** — all jobs across pipelines, filterable

The job log is the primary debugging tool. It shows every command and its
output, which is also why printing a secret there is so dangerous (chapter 10).

## GitLab vs GitHub Actions in one line

Same concepts, different words:

| GitLab | GitHub Actions |
|--------|----------------|
| Pipeline | Workflow |
| Stage | (no direct equivalent — use `needs`) |
| Job | Job |
| `script` | `steps` with `run` |
| Runner | Runner |
| `include` | reusable workflows |

Chapter 14 is a full translation table.

## Try it

Open [`main/.gitlab-ci.yml`](../main/.gitlab-ci.yml) and answer:

1. How many stages are there, and in what order do they run?
2. Which jobs are in the `security` stage? Do they run in parallel?
3. What container image does `pytest-coverage` use?
4. Which two jobs are allowed to fail?

Then compare with a simpler one, for example
[`common/.gitlab-ci.yml`](../common/.gitlab-ci.yml).

## Check yourself

1. What are the three moving parts of GitLab CI/CD?
2. Stages run in ___; jobs in a stage run in ___.
3. How does GitLab decide whether a job passed?
4. Why can't job B see a file created by job A?
5. What does `allow_failure: true` change?

---

Next: [02 — Pipeline anatomy](02-pipeline-anatomy.md)
