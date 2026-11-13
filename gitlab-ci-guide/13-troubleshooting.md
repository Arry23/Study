# 13 — Troubleshooting

## Where to look

1. **The job log** — every command and its output, in order
2. **The pipeline graph** — which job failed, what was skipped
3. **CI Lint** (CI/CD → Pipelines → CI Lint) — validate and simulate
4. **Settings → CI/CD → Runners** — availability and tags
5. **Settings → CI/CD → Variables** — what is actually set

---

## Nothing runs

### No pipeline was created

**`workflow:rules` excluded it.** Check `CI_PIPELINE_SOURCE` against:

```yaml
workflow:
  rules:
    - if: ($CI_PIPELINE_SOURCE == 'merge_request_event' || $CI_PIPELINE_SOURCE == 'push') || $CI_COMMIT_TAG
```

A scheduled or web-triggered run creates nothing here.

**The YAML is invalid.** GitLab shows a broken-pipeline banner. Paste into CI
Lint for the exact line.

**CI is disabled** for the project, or `.gitlab-ci.yml` is not at the root.

### A specific job is missing

Its `rules` did not match. Remember **first match wins**:

```yaml
rules:
  - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
    when: never
  - if: $CI_PIPELINE_SOURCE == 'push' && $CI_COMMIT_BRANCH == 'develop'
    when: on_success
```

On an MR this stops at rule 1. On a push to `main` neither matches, so the job
is excluded.

Add a temporary debug job:

```yaml
debug_rules:
  stage: .pre
  script:
    - echo "source=$CI_PIPELINE_SOURCE branch=$CI_COMMIT_BRANCH tag=$CI_COMMIT_TAG"
```

---

## Stuck pending

```
This job is stuck because you don't have any active runners online
with any of these tags: linux-amd64-m
```

Work down chapter 09's list: does a runner with that tag exist, is it online,
is it assigned to this project, are its slots free, is the branch protected,
and — if the job has no tags — does any runner accept untagged jobs.

A tag typo is the single most common cause.

---

## Script failures

### `command not found`

The tool is not in the image.

```yaml
image: python:3.11
script:
  - yapf --version        # fails: not installed
```

Install it, or use an image that has it:

```yaml
before_script:
  - pip install yapf
```

### The job runs the tool instead of your script

Missing entrypoint override:

```yaml
image:
  name: trufflesecurity/trufflehog:latest
  entrypoint: [""]
```

### `permission denied`

```yaml
script:
  - ./scripts/build.sh      # not executable
```

Either `chmod +x` and commit the bit, or invoke the interpreter:

```yaml
- bash scripts/build.sh
```

### `detected dubious ownership in repository`

Git's safe-directory check, when the runner user differs from the checkout
owner:

```yaml
before_script:
  - git config --global --add safe.directory ${CI_PROJECT_DIR}
```

Exactly what `pre-commit-hooks` does here.

### A failing command does not fail the job

Inside a `|` block, only the last command's status counts:

```yaml
script:
  - |
    step_one          # fails silently
    step_two          # this decides the job
```

Fix:

```yaml
script:
  - |
    set -euo pipefail
    step_one
    step_two
```

### `exit 1` is ignored

`allow_failure: true` on the job. The script did fail; the pipeline chose not to
care.

---

## Files and paths

### "No such file or directory" for something an earlier job made

Different container. Pass it as an artifact:

```yaml
producer:
  artifacts:
    paths: [build/]

consumer:
  needs:
    - job: producer
      artifacts: true
```

### Artifacts not downloaded

- Only **earlier stages** are downloaded automatically
- `dependencies: []` disables it
- With `needs`, you must say `artifacts: true`
- The producing job may have been skipped by `rules`

### The artifact is missing after a failure

Default is `when: on_success`:

```yaml
artifacts:
  when: always
```

This is why `yapf_format`'s diff file vanishes exactly when you want it.

### `cd` did not persist

`before_script` and `script` share a shell, so `cd` carries over. `after_script`
is a **separate shell** — use absolute paths there, as `pytest-coverage` does.

---

## Variables

### Empty when you expected a value

- **Not set** — check Settings → CI/CD → Variables
- **Protected**, and this is not a protected branch
- **Wrong scope** — restricted to a different environment
- **Overridden** by a higher-precedence source (chapter 04)
- **Empty by design** — `CI_COMMIT_TAG` is empty on branch pipelines,
  `CI_MERGE_REQUEST_SOURCE_BRANCH_NAME` is empty outside MR pipelines

Debug without leaking:

```yaml
- echo "length: ${#MY_VAR}"
- '[ -n "$MY_VAR" ] || { echo "MY_VAR is empty"; exit 1; }'
```

> Never `env`, `printenv`, or `CI_DEBUG_TRACE: "true"` in a shared project.

### Expansion in `rules` does not work

`rules:if` is evaluated **server-side, before the job exists**. Runner-only
variables are not available. Use predefined or project variables.

---

## Caches and images

### Cache never hits

- Path outside `CI_PROJECT_DIR` — redirect the tool's cache dir first
- Key changes every run, for example keyed on `CI_COMMIT_SHA`
- Different runners with separate local caches and no shared storage

### `ERROR: Preparation failed: ... no basic auth credentials`

The runner cannot pull the image — private registry credentials missing.

### `toomanyrequests: You have reached your pull rate limit`

Docker Hub rate limiting. Use a mirror:

```yaml
DOCKER_MIRRORS: >
  --registry-mirror docker-cache.repository-ng.intranet.roche.com
  --registry-mirror mirror.gcr.io
```

---

## Pipeline behaviour

### Two pipelines for one push

Push and merge_request_event both matched. Chapter 05:

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH
```

### A job runs too late

Stage barriers. Add `needs:` (chapter 07).

### It passed yesterday, fails today, no code change

Something unpinned moved:

- `image: tool:latest`
- `include: ... ref: main`
- `semgrep --config=auto` fetching new rules
- A transitive dependency

This is the argument for pinning, in one symptom.

### `yaml invalid` after an edit

- Tabs instead of spaces
- Inconsistent indentation
- An unquoted value starting with `*`, `{`, `[`, or containing `: `
- A duplicate key — YAML silently keeps the last one

---

## Debugging technique

**Shorten the loop.** Reproduce locally in the same image:

```bash
docker run --rm -it -v "$PWD:/w" -w /w python:3.11 bash
# then paste the before_script and script lines
```

**Add a temporary diagnostic job** in `.pre`:

```yaml
debug:
  stage: .pre
  script:
    - pwd && ls -la
    - echo "source=$CI_PIPELINE_SOURCE branch=$CI_COMMIT_BRANCH"
```

**Keep evidence on failure:**

```yaml
artifacts:
  when: always
  paths: [logs/, "*.xml"]
```

**Trace carefully.** `set -x` prints expanded variables — never with secrets
present.

**Bisect by commenting out.** Reduce the job to `script: [echo ok]`, confirm it
runs, then add lines back.

**Use CI Lint's simulation** to test rule changes without pushing.

---

## Before asking for help

1. A link to the failing job
2. The error, copied verbatim
3. What changed since it last worked
4. Whether it fails on other branches or only this one
5. Whether it reproduces locally in the same image

## Check yourself

1. Job stuck pending — first three things to check?
2. Why can't job B see job A's files?
3. Why is an artifact missing after a failure?
4. How do you inspect a variable without leaking it?
5. Why can't `rules:if` see runner variables?
6. Pipeline passed yesterday, fails today, no commits. What do you suspect?

---

Previous: [12 — Testing and code quality](12-testing-and-quality.md) ·
Next: [14 — GitLab vs GitHub Actions](14-gitlab-vs-github-actions.md)
