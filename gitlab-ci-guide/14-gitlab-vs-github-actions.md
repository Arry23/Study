# 14 — GitLab CI/CD vs GitHub Actions

You have both in this workspace: twenty-odd `.gitlab-ci.yml` pipelines, and
`arifa-infra/.github/workflows/terraform.yml`. The concepts map closely; the
vocabulary does not.

## Vocabulary

| GitLab | GitHub Actions |
|--------|----------------|
| Pipeline | Workflow |
| `.gitlab-ci.yml` (one file) | `.github/workflows/*.yml` (many files) |
| Stage | — (use `needs`) |
| Job | Job |
| `script:` lines | `steps:` with `run:` |
| Runner | Runner |
| `tags:` | `runs-on:` |
| `image:` | `container:` or the runner's OS |
| `variables:` | `env:` |
| `rules:` | `if:` plus `on:` filters |
| `needs:` | `needs:` |
| `include:` | reusable workflows / composite actions |
| `extends:` | composite actions |
| Artifacts | `upload-artifact` / `download-artifact` |
| `cache:` | `actions/cache` |
| `CI_JOB_TOKEN` | `GITHUB_TOKEN` |
| CI/CD Variables | Secrets and Variables |
| — | Actions (reusable units) |

## Structural difference

**GitLab: stages first.** The pipeline is a sequence of stages; jobs fill them.
Parallelism is implicit within a stage, and `needs` is an optimisation you add.

**GitHub: a graph from the start.** There are no stages. Jobs run in parallel
unless `needs` says otherwise.

```yaml
# GitLab
stages: [build, test]
build_job: { stage: build, script: [make] }
test_job:  { stage: test,  script: [make test] }
```

```yaml
# GitHub
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [{ run: make }]
  test:
    needs: build
    runs-on: ubuntu-latest
    steps: [{ run: make test }]
```

GitLab's stages are friendlier for simple pipelines and become a drag on large
ones. GitHub forces you to think about dependencies immediately.

## Steps vs script

GitLab's unit of work is a **list of shell lines**:

```yaml
pytest-coverage:
  before_script:
    - pip install -r requirements.txt
  script:
    - pytest --cov
```

GitHub's is a **list of steps**, each either a shell command or a reusable
action:

```yaml
- uses: actions/setup-python@<sha>
  with:
    python-version: "3.11"
- run: pip install -r requirements.txt
- run: pytest --cov
```

This is the biggest practical difference. GitHub has a marketplace of
pre-built actions; GitLab has you write shell. Actions reduce boilerplate but
add supply-chain risk — which is why `arifa-infra` pins every one to a commit
SHA.

## Triggers

```yaml
# GitLab
workflow:
  rules:
    - if: ($CI_PIPELINE_SOURCE == 'merge_request_event' || $CI_PIPELINE_SOURCE == 'push') || $CI_COMMIT_TAG
```

```yaml
# GitHub
on:
  pull_request:
    paths:
      - "**/*.tf"
  push:
    branches: [main]
```

GitHub filters at the trigger; GitLab evaluates expressions. GitHub's `paths:`
filter is more convenient than GitLab's `rules:changes`, which is unreliable on
branch pipelines.

## Conditionals

```yaml
# GitLab — job level, first match wins
rules:
  - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
    when: never
  - if: $CI_COMMIT_BRANCH == 'develop'
    when: on_success
```

```yaml
# GitHub — a single boolean
if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

GitLab's ordered list expresses precedence naturally. GitHub's single
expression is simpler until you need precedence, then it gets long.

GitHub also allows step-level `if:`, which GitLab has no equivalent for:

```yaml
- name: Write the plan to the job summary
  if: always()
```

## Runner selection

```yaml
# GitLab
tags:
  - linux-amd64-m
```

```yaml
# GitHub
runs-on: rcn-ubuntu-amd64        # self-hosted label
runs-on: ubuntu-latest           # GitHub-hosted
```

Same idea. GitLab requires **all** listed tags; GitHub matches labels similarly.

## Containers

GitLab runs the whole job in a container by default:

```yaml
image: python:3.11
```

GitHub runs on the runner OS, with containers opt-in:

```yaml
container:
  image: python:3.11
```

More commonly GitHub uses setup actions instead:

```yaml
- uses: actions/setup-python@<sha>
  with: { python-version: "3.11" }
```

## Secrets

```yaml
# GitLab — plain variables, masked in the UI
script:
  - curl -H "Authorization: Bearer $API_TOKEN" ...
```

```yaml
# GitHub — explicit namespace
run: curl -H "Authorization: Bearer $TOKEN" ...
env:
  TOKEN: ${{ secrets.API_TOKEN }}
```

GitHub's `secrets.` namespace makes it obvious in review that a value is
sensitive. GitLab's secrets look like any other variable, which is partly why
`echo "token ${PERSONAL_TOKEN}"` slipped into your `submodule` job.

GitHub masks secret values in logs automatically; GitLab requires the masked
checkbox and silently cannot mask values that fail its format rules.

## Tokens

| | GitLab | GitHub |
|---|---|---|
| Automatic token | `CI_JOB_TOKEN` | `GITHUB_TOKEN` |
| Lifetime | The job | The job |
| Permissions | Project-scoped, configurable | Declared per workflow/job |

GitHub's `permissions:` block is a real advantage:

```yaml
permissions:
  contents: read          # default for the whole workflow

jobs:
  plan:
    permissions:
      contents: read
      id-token: write     # raised only where needed
```

GitLab has no per-job equivalent.

## Cloud authentication

Both support OIDC, and both should use it instead of stored keys:

```yaml
# GitHub
permissions:
  id-token: write
steps:
  - uses: aws-actions/configure-aws-credentials@<sha>
    with:
      role-to-assume: ${{ vars.AWS_PLAN_ROLE_ARN }}
      aws-region: eu-central-1
```

```yaml
# GitLab
job:
  id_tokens:
    AWS_TOKEN:
      aud: https://gitlab.com
  script:
    - aws sts assume-role-with-web-identity --web-identity-token "$AWS_TOKEN" ...
```

See [the Terraform guide, chapter 13](../terraform-guide/13-cicd-and-oidc.md)
for why the trust policy's subject condition is the part that matters.

## Reuse

| | GitLab | GitHub |
|---|---|---|
| Whole pipelines | `include: project:` | reusable workflows (`uses:` on a job) |
| Job templates | `extends` + hidden jobs | composite actions |
| Fragments | YAML anchors, `!reference` | composite actions |

```yaml
# GitLab
include:
  - project: "org/ci-templates"
    ref: v2.3.1
    file: ["python/test.yml"]
```

```yaml
# GitHub
jobs:
  test:
    uses: org/ci-templates/.github/workflows/python-test.yml@v2.3.1
```

Both must be **pinned**. `ref: main` and `@main` have the same problem.

## Output and reporting

GitLab renders `reports:` in the merge request widget — test results, coverage,
SAST findings. That integration is genuinely better than GitHub's default.

GitHub has job summaries, which are flexible but manual:

```yaml
- run: |
    {
      echo "### terraform plan"
      echo '```'
      head -c 60000 plan.txt
      echo '```'
    } >> "$GITHUB_STEP_SUMMARY"
```

## Honest comparison

**GitLab is better at:** integrated reports in the MR, one-file simplicity for
small pipelines, built-in security templates, `extends` for deep merging.

**GitHub is better at:** the actions ecosystem, per-job `permissions`,
step-level `if:`, path filters on triggers, matrix builds, multiple workflow
files per repository.

**Both get wrong by default:** unpinned references. GitLab `ref: main` and
`image: tool:latest`; GitHub `uses: action@v4`. Pin everything.

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
```

```yaml
# GitHub
jobs:
  semgrep:
    runs-on: rcn-ubuntu-amd64
    container:
      image: returntocorp/semgrep:1.86.0
    steps:
      - uses: actions/checkout@<sha>
      - run: semgrep --config=auto --json --output=semgrep-results.json .
      - uses: actions/upload-artifact@<sha>
        if: always()
        with:
          name: semgrep-results
          path: semgrep-results.json
```

Note GitHub needs an explicit checkout step — GitLab clones automatically.

## Check yourself

1. What is GitHub's equivalent of a stage?
2. Why does GitHub need an explicit checkout step?
3. How does GitHub's `secrets.` namespace help review?
4. What can GitHub's `permissions:` do that GitLab cannot?
5. Which platform renders test reports in the merge request by default?
6. What mistake do both platforms make easy?

---

Previous: [13 — Troubleshooting](13-troubleshooting.md) ·
Next: [15 — Cheat sheet](15-cheatsheet.md) ·
Also: [GitHub Actions, A to Z](../github-actions-guide/README.md)
