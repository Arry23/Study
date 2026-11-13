# 15 — Cheat sheet

One page. Keep it open.

## File skeleton

```yaml
stages: [build, test, deploy]

include:
  - project: "org/ci-templates"
    ref: v2.3.1                       # PIN IT
    file: ["python/test.yml"]
  - template: Jobs/Dependency-Scanning.gitlab-ci.yml

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH

default:
  image: python:3.11
  tags: [linux-amd64-m]

my_job:
  stage: test
  needs: []
  script:
    - pytest
```

## Job keys

```yaml
job:
  stage: test                 # which stage
  image: python:3.11          # container
  tags: [linux-amd64-m]       # runner selection
  needs: []                   # DAG dependencies
  variables: { KEY: value }
  before_script: [setup]
  script: [work]              # REQUIRED
  after_script: [cleanup]     # separate shell, runs on failure
  rules: [...]
  artifacts: {...}
  cache: {...}
  allow_failure: true
  timeout: 30 minutes
  retry: { max: 2, when: runner_system_failure }
  environment: production
  extends: .template
```

## Images

```yaml
image: python:3.11

image:                        # when the image has an ENTRYPOINT
  name: gcr.io/kaniko-project/executor:debug
  entrypoint: [""]

services:
  - name: postgres:16
    alias: db
```

## Rules

```yaml
rules:
  - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  - if: $CI_COMMIT_TAG                               # tag pipeline
  - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'
  - if: $CI_PIPELINE_SOURCE == "schedule"
  - exists: [.gitmodules]
  - changes: ["src/**/*"]
  - when: never
```

`when:` values: `on_success` (default), `on_failure`, `always`, `never`,
`manual`, `delayed`.

```yaml
deploy:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
      allow_failure: false        # blocking approval gate
```

## Artifacts

```yaml
artifacts:
  name: "$CI_JOB_NAME-$CI_COMMIT_REF_SLUG"
  paths: [build/, "*.xml"]
  when: always                    # on_success | on_failure | always
  expire_in: 1 week
  reports:
    junit: report.xml
    sast: semgrep-results.json
    coverage_report:
      coverage_format: cobertura
      path: coverage.xml
```

```yaml
dependencies: []                  # download nothing
```

## Cache

```yaml
cache:
  key:
    files: [requirements.txt]     # invalidate on content change
  paths: [.cache/pip]             # MUST be inside CI_PROJECT_DIR
  policy: pull                    # pull | push | pull-push
```

## needs

```yaml
needs: []                         # start immediately

needs:
  - job: build
    artifacts: true
  - job: lint
    artifacts: false
  - job: optional_job
    optional: true
```

## Reuse

```yaml
.base:                            # hidden job — a template
  image: python:3.11
  tags: [linux-amd64-m]

job:
  extends: .base                  # deep merge, works across includes

.anchor: &anchor                  # YAML only, one file, shallow
  key: value
job2:
  <<: *anchor

job3:
  script:
    - !reference [.base, before_script]
```

## Predefined variables

```bash
CI_PROJECT_DIR          # checkout path
CI_PROJECT_NAME         # repo name
CI_PROJECT_ID           # numeric id
CI_PIPELINE_ID
CI_PIPELINE_SOURCE      # push | merge_request_event | schedule | web | api
CI_COMMIT_BRANCH        # empty on MR pipelines
CI_COMMIT_TAG           # empty unless a tag pipeline
CI_COMMIT_SHA / CI_COMMIT_SHORT_SHA
CI_DEFAULT_BRANCH
CI_MERGE_REQUEST_SOURCE_BRANCH_NAME   # empty outside MR pipelines
CI_REGISTRY / CI_REGISTRY_IMAGE
CI_JOB_TOKEN            # short-lived, job-scoped
CI_API_V4_URL
GITLAB_USER_LOGIN / GITLAB_USER_ID
```

## Built-in stages

```yaml
job:
  stage: .pre        # always first
job2:
  stage: .post       # always last, even on failure
```

## Kaniko build

```yaml
image-build:
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - /kaniko/executor
        --context "$CI_PROJECT_DIR"
        --destination "$CI_REGISTRY_IMAGE:$CI_COMMIT_TAG"
        --single-snapshot
  rules:
    - if: $CI_COMMIT_TAG
```

## Package upload

```yaml
- curl --header "JOB-TOKEN: $CI_JOB_TOKEN" \
       --upload-file file.zip \
       "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/NAME/VERSION/file.zip"
```

## Shell safety

```yaml
script:
  - |
    set -euo pipefail        # or a mid-script failure is ignored
    step_one
    step_two
```

`|` keeps newlines (scripts). `>` folds to one line (argument lists).

## Safe variable debugging

```yaml
- echo "length: ${#MY_VAR}"
- '[ -n "$MY_VAR" ] || { echo "MY_VAR empty"; exit 1; }'
```

Never: `env`, `printenv`, `echo "$SECRET"`, `CI_DEBUG_TRACE: "true"`.

## Safety rules

1. No secrets in `.gitlab-ci.yml`. Ever.
2. Never `echo` a secret — not even "temporarily".
3. Mask every secret; protect production ones.
4. Pin images to a version or digest. Never `:latest`.
5. Pin `include: ref:` to a tag or SHA. Never `main`.
6. `set -euo pipefail` in every `|` block.
7. `allow_failure: true` on a gate means it is not a gate.
8. `artifacts: when: always` on anything you need to debug a failure.
9. Cache paths must live inside `CI_PROJECT_DIR`.
10. Prefer `CI_JOB_TOKEN` to a personal token.

## Validate

```bash
# YAML syntax only
python -c "import yaml; yaml.safe_load(open('.gitlab-ci.yml'))"
```

GitLab semantics: CI/CD → Pipelines → **CI Lint** (with "Simulate pipeline
creation").

## Quick diagnosis

| Symptom | Likely cause |
|---------|--------------|
| No pipeline | `workflow:rules`, or invalid YAML |
| Job missing | `rules` did not match |
| Stuck pending | No runner with those tags |
| `command not found` | Tool not in the image |
| Tool runs instead of script | Missing `entrypoint: [""]` |
| File missing from earlier job | Needs an artifact |
| Artifact missing after failure | Needs `when: always` |
| Variable empty | Not set, protected, or empty by design |
| Cache never hits | Path outside `CI_PROJECT_DIR` |
| Passed yesterday, fails today | Something unpinned moved |
| `dubious ownership` | `git config --global --add safe.directory` |

---

Previous: [14 — GitLab vs GitHub Actions](14-gitlab-vs-github-actions.md) ·
Next: [16 — Hands-on labs](16-hands-on-labs.md) ·
Back to the [index](README.md)
