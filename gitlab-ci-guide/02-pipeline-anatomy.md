# 02 — Pipeline anatomy

## The file

`.gitlab-ci.yml` at the repository root. Top-level keys are either **global
configuration** or **job definitions** — anything that is not a reserved word is
a job.

```yaml
stages: [...]        # global: the ordered stage list
include: [...]       # global: pull in other YAML
variables: {...}     # global: variables for every job
workflow: {...}      # global: when a pipeline is created
default: {...}       # global: defaults for every job

my_job:              # anything else is a JOB
  stage: test
  script:
    - echo hello
```

The reserved words are: `stages`, `include`, `variables`, `workflow`,
`default`, `image`, `services`, `before_script`, `after_script`, `cache`,
`pages`. Everything else at the top level is a job name.

## `stages`

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

The **order of this list is the order of execution**. A stage begins only when
every job in the previous stage has finished successfully (or was allowed to
fail).

If you omit `stages`, GitLab uses `build`, `test`, `deploy`.

A job with no `stage:` lands in `test`.

### Stage names are arbitrary

Notice that several stages here contain exactly one job with the same name:
`trufflehog`, `sonarqube`, `sonar-report`, `image-build`. The stage exists
purely to force ordering.

That is legitimate, but it is also the main cost of stages: **a stage is a
barrier**. `sonarqube` cannot start until every job in `pytest-coverage`
finishes, even if it only needs one of them. Chapter 07 shows how `needs`
removes those barriers.

## A job

Minimum viable job — a name and a `script`:

```yaml
hello:
  script:
    - echo "hello"
```

A realistic one:

```yaml
semgrep-sast:
  stage: security
  image: returntocorp/semgrep:latest
  tags:
    - linux-amd64-m
  variables:
    GIT_PROJECT_DIRECTORY: /builds/ocean/.../ocean-cdr-coralreef-main
  script:
    - semgrep --config=auto --json --output=semgrep-results.json .
  artifacts:
    paths:
      - semgrep-results.json
    reports:
      sast: semgrep-results.json
    when: always
```

| Key | Meaning | Chapter |
|-----|---------|---------|
| `stage` | Which stage it belongs to | this one |
| `image` | Container to run in | 03 |
| `tags` | Which runners may pick it up | 09 |
| `variables` | Job-scoped variables | 04 |
| `before_script` | Runs before `script` | 03 |
| `script` | **The work.** Required | 03 |
| `after_script` | Runs even if `script` failed | 03 |
| `artifacts` | Files to keep and pass on | 06 |
| `cache` | Files to reuse for speed | 06 |
| `rules` | Whether this job runs at all | 05 |
| `needs` | Run as soon as these finish | 07 |
| `extends` | Inherit from another job | 08 |
| `allow_failure` | Failure does not block | 01 |

## Execution order

```
submodule ──► format ──► trufflehog ──► security ──► pytest-coverage ──► ...
                                        │
                                        ├── semgrep-sast        ┐
                                        └── pre-commit-hooks    ┘ parallel
```

Two rules:

1. Stages run in the declared order.
2. Jobs in the same stage run in parallel, limited by available runners.

`needs` overrides rule 1 for a specific job.

## Global `variables`

```yaml
variables:
  PYTHONPATH: "/builds/ocean/.../tests:$CI_PROJECT_DIR"
```

Available in every job. Jobs can add their own or override these:

```yaml
pytest-coverage:
  variables:
    GIT_PROJECT_DIRECTORY: /builds/ocean/.../ocean-cdr-coralreef-main
```

Chapter 04 covers precedence.

## `workflow`

Controls whether a **pipeline is created at all** — evaluated once, before any
job:

```yaml
workflow:
  rules:
    - if: ($CI_PIPELINE_SOURCE == 'merge_request_event' || $CI_PIPELINE_SOURCE == 'push') || $CI_COMMIT_TAG
```

No matching rule, no pipeline. This is the cleanest way to avoid duplicate
pipelines, where a push and its merge request each spawn a run. Chapter 05.

## `include`

Pull pipeline definitions from elsewhere:

```yaml
include:
  - project: "ocean/enablers/devops-library/cicd-pipelines"
    ref: main
    file:
      - "sonarscan/sonarscan.yml"
  - project: "gitlab/pipeline-lib"
  - template: Jobs/Dependency-Scanning.gitlab-ci.yml
```

Three kinds here: another GitLab project, a second project, and a GitLab-shipped
template. Included files can define jobs, variables and hidden templates. That
is where `.sonarqube-check` and `.get-coverage` come from — they are not in this
file.

> `ref: main` is a **moving reference**. The shared pipeline can change under
> you with no commit in this repository, which means a pipeline that passed
> yesterday can fail today for reasons invisible in your git history. Pin to a
> tag or SHA. Chapter 08.

## `default`

Defaults every job inherits unless it overrides them:

```yaml
default:
  image: python:3.11
  tags:
    - linux-amd64-m
  before_script:
    - pip install -r requirements.txt
```

`main/.gitlab-ci.yml` does not use this, which is why `image: python:3.11` and
`tags: [linux-amd64-m]` are repeated in job after job. A `default` block would
remove most of that duplication.

## Hidden jobs

A name starting with `.` is **not a job** — it is a template to inherit from:

```yaml
.python-base:
  image: python:3.11
  tags:
    - linux-amd64-m

pytest:
  extends: .python-base
  script:
    - pytest
```

`extends: [.get-coverage]` and `extends: .sonarqube-check` in this pipeline both
refer to hidden jobs defined in the included projects. Chapter 08.

## Validating the file

YAML is unforgiving, and a syntax error means no pipeline at all.

- **CI/CD → Editor** in GitLab validates as you type
- **CI Lint** (CI/CD → Pipelines → CI Lint) validates a pasted file and can
  simulate a run
- Locally, any YAML parser catches indentation errors:

```bash
python -c "import yaml,sys; yaml.safe_load(open('.gitlab-ci.yml'))"
```

That only checks YAML validity, not GitLab semantics — a misspelled `scripts:`
parses fine and does nothing.

## Try it

```bash
cd fdd_1.19
python -c "import yaml; d=yaml.safe_load(open('main/.gitlab-ci.yml')); print(d['stages'])"
python -c "import yaml; d=yaml.safe_load(open('main/.gitlab-ci.yml')); print([k for k in d if not k.startswith('_')])"
```

Then, from the file alone:

1. Which top-level keys are global config rather than jobs?
2. Which jobs would run in parallel in the `security` stage?
3. Which stages exist only to force ordering of a single job?

## Check yourself

1. How does GitLab tell a job from global configuration?
2. What happens to a job with no `stage:`?
3. When does a stage start?
4. What is the difference between `workflow:rules` and a job's `rules`?
5. What does a leading `.` on a job name mean?
6. Why is `ref: main` on an `include` a risk?

---

Previous: [01 — What GitLab CI/CD is](01-what-is-gitlab-cicd.md) ·
Next: [03 — Jobs and scripts](03-jobs-and-scripts.md)
