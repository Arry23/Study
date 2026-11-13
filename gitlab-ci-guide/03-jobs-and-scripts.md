# 03 — Jobs and scripts

## The three script keys

```yaml
job_name:
  before_script:
    - setup commands
  script:
    - the actual work
  after_script:
    - cleanup, runs even on failure
```

| Key | Runs | On failure of `script` |
|-----|------|------------------------|
| `before_script` | First | — |
| `script` | The work. **Required** | — |
| `after_script` | Last | **Still runs** |

`before_script` and `script` are effectively one shell session: a failure in
`before_script` fails the job and `script` never starts.

`after_script` runs in a **separate shell**. Variables and the working
directory do not carry over. That is deliberate — it must work even when the
job crashed.

## A worked example

```yaml
pytest-coverage:
  stage: pytest-coverage
  image: python:3.11
  before_script:
    - cp -r ocean-cdr-coralreef-common/code/backend/common_configurations code/backend/
    - rm -rf $GIT_PROJECT_DIRECTORY/ocean-cdr-coralreef-common
    - apt-get update -y
    - apt-get install -y libsasl2-dev python3-dev libldap2-dev libssl-dev
    - pip install -r installables/requirements.txt
    - cd code/backend/apis/tests/
  script:
    - pytest --cov-report term-missing --cov-report xml:coverage.xml --cov --cov-config=../../../../.coveragerc
  after_script:
    - python3.11 $GIT_PROJECT_DIRECTORY/installables/gitlab_ci_update_sonarqube_coverage.py
    - xmllint --noout $GIT_PROJECT_DIRECTORY/code/backend/apis/tests/coverage.xml
```

Note `cd` at the end of `before_script` — it carries into `script`, because they
share a shell. It does **not** carry into `after_script`, which is why those
commands use absolute paths built from `$GIT_PROJECT_DIRECTORY`.

> Jobs that `apt-get install` on every run are slow. A purpose-built image with
> the system packages baked in turns a two-minute install into nothing. See
> chapter 11.

## `image`

The container the job runs in:

```yaml
image: python:3.11
```

Any image your runner can pull. If omitted, the runner's default is used.

### Overriding the entrypoint

Some images define an `ENTRYPOINT` that runs the tool directly. GitLab needs to
run a shell, so you blank it:

```yaml
image:
  name: trufflesecurity/trufflehog:latest
  entrypoint: [""]

image:
  name: gcr.io/kaniko-project/executor:debug
  entrypoint: [""]
```

Without `entrypoint: [""]` those jobs fail immediately with something like
"command not found" or the tool running with no arguments. **Remember this
pattern** — it accounts for a lot of confusing first-run failures.

### Pin your images

```yaml
image: python:3.11                        # minor pin — acceptable
image: trufflesecurity/trufflehog:latest  # unpinned — risky
image: returntocorp/semgrep:latest        # unpinned — risky
```

`:latest` means the tool can change between two runs of the same commit. A
pipeline that passed this morning can fail this afternoon with no change from
you, and a compromised upstream tag runs in your CI with your tokens.

Pin to a version, ideally a digest:

```yaml
image: trufflesecurity/trufflehog:3.90.8
image: python@sha256:1e8e0...      # immutable
```

This is the same argument as pinning GitHub Actions to a SHA, and the same
argument `arifa-infra` makes for pinning Terraform providers.

## `script` is YAML, then shell

Each list item is a line of shell:

```yaml
script:
  - echo "one"
  - echo "two"
```

### Multi-line blocks

For anything with control flow, use a literal block:

```yaml
script:
  - |
    if git diff --name-only | tee diff.txt | grep -q .; then
      echo "Differences found, see diff.txt"
      exit 1
    else
      echo "Code already formatted"
    fi
```

`|` keeps newlines, so the shell sees a real multi-line script. `>` folds lines
into one, which breaks `if` blocks — use `|` for scripts.

### YAML characters that bite

A value starting with certain characters must be quoted:

```yaml
script:
  - echo "hello: world"        # colon-space inside a plain scalar
  - 'echo *'                   # * starts an alias in YAML
  - "echo {{ templated }}"     # { starts a flow mapping
```

When in doubt, quote, or use a `|` block.

## Exit codes decide everything

A job passes when the script exits 0.

```yaml
script:
  - false            # job fails here
  - echo "never runs"
```

Each list item is checked. The first non-zero exit stops the job.

Inside a `|` block it is **one** shell invocation, so only the last command's
status counts unless you opt in:

```yaml
script:
  - |
    set -euo pipefail      # fail on first error, unset var, or pipe failure
    step_one
    step_two
```

> Without `set -e` in a `|` block, a failing middle command is silently ignored
> and the job goes green. This is exactly the class of bug that made
> `arifa-infra`'s `check.sh` report success while two roots were failing — see
> [the Terraform guide, chapter 12](../terraform-guide/12-quality-gates-and-testing.md).

### Forcing a failure

```yaml
script:
  - |
    if [ -s findings.txt ]; then
      cat findings.txt
      exit 1
    fi
```

### Tolerating a failure

```yaml
script:
  - risky_command || true
```

Use sparingly — it is `allow_failure` at the line level, with none of the
visibility.

## `tags`

Which runners may take the job:

```yaml
tags:
  - linux-amd64-m
```

The runner must have **all** the listed tags. Get this wrong and the job sits
in `pending` forever. Chapter 09.

## `variables`

Job-scoped, overriding globals:

```yaml
image-build:
  variables:
    CONTAINER_REGISTRY: $CI_REGISTRY_IMAGE
    BUILD_TAG: $CI_COMMIT_TAG
    DOCKER_MIRRORS: >
      --registry-mirror docker-cache.repository-ng.intranet.roche.com
      --registry-mirror mirror.gcr.io
```

`>` folds those two lines into one space-separated string — correct here,
because it is a single argument list rather than a script.

## `services`

Extra containers alongside your job, reachable by hostname:

```yaml
test:
  image: python:3.11
  services:
    - name: postgres:16
      alias: db
  variables:
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgresql://postgres:test@db:5432/postgres
  script:
    - pytest
```

How you get a real database or Redis in tests without mocking.

## Timeouts and retries

```yaml
job:
  timeout: 30 minutes
  retry:
    max: 2
    when: runner_system_failure
```

Retry only genuinely transient classes — `runner_system_failure`,
`stuck_or_timeout_failure`. Retrying `script_failure` just runs a failing test
three times.

## Try it

Open [`main/.gitlab-ci.yml`](../main/.gitlab-ci.yml) and find:

1. Two jobs that override the image entrypoint, and why
2. A job whose `before_script` ends with `cd`, and where that matters
3. Every `|` block, and what each would do differently with `>`
4. Each unpinned `:latest` image

## Check yourself

1. Do `before_script` and `script` share a shell? Does `after_script`?
2. Why does `after_script` use absolute paths here?
3. When do you need `entrypoint: [""]`?
4. `|` vs `>` — which for a script, which for an argument list?
5. In a `|` block, which command's exit code decides the job?
6. Why is `image: tool:latest` a problem?

---

Previous: [02 — Pipeline anatomy](02-pipeline-anatomy.md) ·
Next: [04 — Variables](04-variables.md)
