# 15 — GitHub Actions vs GitLab CI

You have both here: `arifa-infra/.github/workflows/terraform.yml`, and around
twenty `.gitlab-ci.yml` pipelines. The concepts map closely; the vocabulary and
the defaults do not.

## Vocabulary

| GitHub Actions | GitLab CI |
|----------------|-----------|
| Workflow | Pipeline |
| `.github/workflows/*.yml` (many) | `.gitlab-ci.yml` (one) |
| — | Stage |
| Job | Job |
| `steps:` with `run:` | `script:` |
| `uses:` (action) | — (write shell) |
| `runs-on:` | `tags:` |
| `container:` | `image:` |
| `env:` | `variables:` |
| `if:` and `on:` filters | `rules:` |
| `needs:` | `needs:` |
| Reusable workflow | `include:` |
| Composite action | `extends:` + hidden job |
| `upload-artifact` / `download-artifact` | `artifacts:` |
| `actions/cache` | `cache:` |
| `GITHUB_TOKEN` | `CI_JOB_TOKEN` |
| Secrets / Variables | CI/CD Variables |
| Job summary | `reports:` |

## The structural difference

**GitLab: stages first.** A pipeline is an ordered list of stages; jobs fill
them. Order is implicit, and `needs:` is an optimisation you add later.

**GitHub: a graph from the start.** No stages. Jobs run in parallel unless
`needs:` says otherwise.

```yaml
# GitLab
stages: [build, test]
build_job: { stage: build, script: [make] }
test_job:  { stage: test,  script: [make test] }
```

```yaml
# GitHub
jobs:
  build: { runs-on: ubuntu-latest, steps: [{ run: make }] }
  test:  { needs: build, runs-on: ubuntu-latest, steps: [{ run: make test }] }
```

GitLab is friendlier for a simple pipeline. Ten sequential stages — as in
`main/.gitlab-ci.yml` — become a performance problem, because every stage is a
barrier. GitHub forces the dependency question immediately.

## Steps vs script

GitLab's unit is a list of shell lines. GitHub's is a list of steps, each either
shell or a reusable action.

```yaml
# GitLab
before_script: [pip install -r requirements.txt]
script: [pytest --cov]
```

```yaml
# GitHub
steps:
  - uses: actions/checkout@<sha>
  - uses: actions/setup-python@<sha>
    with: { python-version: "3.12" }
  - run: pip install -r requirements.txt
  - run: pytest --cov
```

The marketplace is GitHub's biggest advantage and its biggest risk: `uses:` runs
third-party code with your credentials. GitLab makes you write shell, which is
more verbose and has a smaller attack surface.

## Checkout

GitLab clones automatically. GitHub does not:

```yaml
- uses: actions/checkout@<sha>
```

Forgetting it is the classic first error.

## Containers

GitLab runs the whole job in a container by default. GitHub runs on the runner
OS, with `container:` opt-in — and the idiomatic approach is a setup action
instead.

| | GitLab | GitHub |
|---|---|---|
| Default | Container (`image:`) | Runner OS |
| Toolchain | Pick an image | `setup-*` action |
| Override entrypoint | `entrypoint: [""]` needed | n/a |

That `entrypoint: [""]` quirk — needed for Kaniko, TruffleHog and Semgrep in
your pipelines — has no GitHub equivalent.

## Conditionals

```yaml
# GitLab — ordered list, first match wins
rules:
  - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
    when: never
  - if: $CI_COMMIT_BRANCH == 'develop'
    when: on_success
```

```yaml
# GitHub — a single boolean
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

GitLab's ordered rules express precedence naturally. GitHub's single expression
is simpler until precedence matters, then it gets long.

GitHub has **step-level `if:`**, which GitLab lacks entirely:

```yaml
- name: Write the plan to the job summary
  if: always()
```

GitLab's nearest equivalent is `after_script`, which is far less flexible.

## Path filtering

```yaml
# GitHub — on the trigger, uses the full PR diff
on:
  pull_request:
    paths: ["**/*.tf", "Makefile"]
```

```yaml
# GitLab — per job, unreliable on branch pipelines
rules:
  - changes: ["**/*.tf"]
```

GitHub wins here. GitLab's `changes:` compares against the previous commit on
branch pipelines, which misleads after a force-push or squash.

## Secrets

```yaml
# GitLab — looks like any other variable
script: [curl -H "Authorization: Bearer $API_TOKEN" ...]
```

```yaml
# GitHub — explicit namespace, explicit passing
- run: curl -H "Authorization: Bearer $TOKEN" ...
  env:
    TOKEN: ${{ secrets.API_TOKEN }}
```

GitHub's design is safer in practice:

- `secrets.` makes sensitivity obvious in review
- Secrets must be passed explicitly, so a step's reach is visible
- Masking is automatic, not a checkbox

GitLab's model is how `echo "token ${PERSONAL_TOKEN}"` ended up in
`main/.gitlab-ci.yml` — it looks like any other debug line. See
[the GitLab guide, chapter 10](../gitlab-ci-guide/10-secrets-and-security.md).

## Permissions

GitHub's per-job token scoping has no GitLab equivalent:

```yaml
permissions:
  contents: read

jobs:
  plan:
    permissions:
      contents: read
      id-token: write
```

A real advantage. GitLab's `CI_JOB_TOKEN` scope is project-wide configuration,
not per job.

## Reuse

```yaml
# GitLab
include:
  - project: "org/ci-templates"
    ref: v2.3.1
    file: ["python/test.yml"]

job:
  extends: .python-base
```

```yaml
# GitHub
jobs:
  test:
    uses: org/ci-templates/.github/workflows/python-test.yml@v2.3.1

# or, for steps:
- uses: ./.github/actions/setup-tools
```

GitLab's `extends` deep-merges, which is genuinely convenient. GitHub's
composite actions are more explicit but wordier.

**Both must be pinned.** `ref: main` and `@main` are the same mistake.

## Reporting

GitLab's `reports:` is better out of the box:

```yaml
artifacts:
  reports:
    junit: report.xml
    sast: semgrep-results.json
    coverage_report: { coverage_format: cobertura, path: coverage.xml }
```

Test failures, coverage and security findings appear in the merge request
widget automatically.

GitHub needs more work — job summaries are flexible but manual:

```yaml
- run: |
    echo "### terraform plan" >> "$GITHUB_STEP_SUMMARY"
```

or a third-party action to comment on the PR, which needs
`pull-requests: write`.

## Honest comparison

**GitLab is better at:** merge-request integration and reports, one-file
simplicity, built-in security templates, `extends` deep merging, automatic
checkout and container defaults.

**GitHub is better at:** the actions ecosystem, per-job `permissions`,
step-level `if:`, trigger path filters, matrix builds, multiple independent
workflow files, explicit secret handling.

**Both default badly on:** mutable references. GitLab `ref: main` and
`image: tool:latest`; GitHub `uses: action@v4` and `runs-on: ubuntu-latest`.
Pin everything, everywhere.

## Translating a job

```yaml
# GitLab
semgrep-sast:
  stage: security
  image: returntocorp/semgrep:1.86.0
  tags: [linux-amd64-m]
  script:
    - semgrep --config=auto --json --output=semgrep-results.json .
  artifacts:
    when: always
    paths: [semgrep-results.json]
    reports:
      sast: semgrep-results.json
```

```yaml
# GitHub
jobs:
  semgrep:
    runs-on: rcn-ubuntu-amd64
    container:
      image: returntocorp/semgrep:1.86.0
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@<sha>
        with:
          persist-credentials: false
      - run: semgrep --config=auto --json --output=semgrep-results.json .
      - uses: actions/upload-artifact@<sha>
        if: always()
        with:
          name: semgrep-results
          path: semgrep-results.json
```

Differences: explicit checkout, explicit artifact upload, `permissions`
available, and no direct equivalent of `reports:sast:`.

## Migration tips

Going **GitLab → GitHub**:

1. One workflow per stage group, not one giant file
2. `stages` become `needs`
3. `image:` becomes a setup action, or `container:`
4. `rules:` become `on:` filters plus `if:`
5. Add the checkout step everywhere
6. Add `permissions:` — do not rely on defaults
7. Pin every action to a SHA

Going **GitHub → GitLab**:

1. Merge workflows into one file with stages, or keep `needs`
2. Replace actions with shell, or find a template
3. Remove checkout steps
4. `permissions:` has no equivalent — scope `CI_JOB_TOKEN` at project level
5. Use `reports:` for anything you were summarising manually

## Check yourself

1. What is GitHub's equivalent of a stage?
2. Why does GitHub need an explicit checkout?
3. Why is GitHub's path filtering more reliable?
4. What can `permissions:` do that GitLab cannot?
5. Which platform surfaces test reports in the MR by default?
6. Name a mutable-reference default on each platform.

---

Previous: [14 — Troubleshooting](14-troubleshooting.md) ·
Next: [16 — Cheat sheet](16-cheatsheet.md) ·
Also: [GitLab CI/CD, A to Z](../gitlab-ci-guide/README.md)
