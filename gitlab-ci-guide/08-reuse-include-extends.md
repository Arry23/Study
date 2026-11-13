# 08 — Reuse: include, extends, anchors

There are around twenty `.gitlab-ci.yml` files in this workspace, many of them
near-identical. That is the problem this chapter solves.

Three mechanisms, in increasing order of power:

| Mechanism | Scope | Use for |
|-----------|-------|---------|
| YAML anchors | One file | Small repetition |
| `extends` | One file, or included files | Job templates |
| `include` | Across projects | Organisation-wide pipelines |

## `include`

```yaml
include:
  - project: "ocean/enablers/devops-library/cicd-pipelines"
    ref: main
    file:
      - "sonarscan/sonarscan.yml"
  - project: "gitlab/pipeline-lib"
  - template: Jobs/Dependency-Scanning.gitlab-ci.yml
```

Four forms:

```yaml
include:
  - local: "/ci/jobs.yml"                   # this repository
  - project: "group/project"                # another GitLab project
    ref: v1.4.0
    file: "/templates/python.yml"
  - remote: "https://example.com/ci.yml"    # any URL
  - template: Security/SAST.gitlab-ci.yml   # GitLab-provided
```

Included files are merged before the pipeline is created. They can define jobs,
variables, hidden templates, even their own `include`.

That is where `.sonarqube-check` and `.get-coverage` come from — neither is
defined in `main/.gitlab-ci.yml`, yet both are extended by jobs in it.

### Pin your includes

> **`ref: main` is a moving reference.** The shared pipeline can change with no
> commit in your repository. A pipeline that passed yesterday fails today, your
> git history explains nothing, and in the worst case someone else's change runs
> with your `CI_JOB_TOKEN`.
>
> Pin to a tag or SHA:
>
> ```yaml
> include:
>   - project: "ocean/enablers/devops-library/cicd-pipelines"
>     ref: v2.3.1                     # or a full commit SHA
>     file: ["sonarscan/sonarscan.yml"]
> ```
>
> This is the same argument as pinning a container image, a GitHub Action to a
> SHA, or a Terraform provider version. Upgrades should be commits.

Note also that `- project: "gitlab/pipeline-lib"` has **no `file:`**, which
means it pulls that project's default `.gitlab-ci.yml`. Whatever it contains is
silently part of your pipeline. Be explicit about which file you want.

### Overriding included jobs

Redefine a job after including it and the keys merge:

```yaml
include:
  - template: Jobs/Dependency-Scanning.gitlab-ci.yml

gemnasium-python-dependency_scanning:
  stage: dependency-scanning-job
  variables:
    SECURE_LOG_LEVEL: "debug"
    PIP_CONSTRAINT: "/tmp/constraints.txt"
  before_script:
    - echo "setuptools==65.5.1" > /tmp/constraints.txt
    - apt-get update && apt-get install -y libpq-dev
```

That job comes from GitLab's template; this adds a stage, variables and a
`before_script` without copying the whole definition. The merge is shallow —
specifying `before_script` replaces the template's entirely.

## `extends`

```yaml
.python-base:
  image: python:3.11
  tags:
    - linux-amd64-m
  before_script:
    - pip install -r installables/requirements.txt

pytest:
  extends: .python-base
  stage: test
  script:
    - pytest
```

A name starting with `.` is a **hidden job** — a template, never executed.

`extends` deep-merges: nested keys combine rather than replace, unlike YAML
anchors.

Multiple parents, later wins:

```yaml
pytest-coverage:
  extends: [.get-coverage]

sonarqube-scan:
  extends: .sonarqube-check
```

Both of those come from included projects. You can chain up to eleven levels,
though more than two is usually a sign to simplify.

### What this workspace could use

Nearly every job repeats:

```yaml
image: python:3.11
tags:
  - linux-amd64-m
variables:
  GIT_PROJECT_DIRECTORY: /builds/ocean/.../ocean-cdr-coralreef-main
```

A single hidden job removes all of it:

```yaml
.base:
  image: python:3.11
  tags:
    - linux-amd64-m
  variables:
    GIT_PROJECT_DIRECTORY: $CI_PROJECT_DIR

.with-common:
  extends: .base
  before_script:
    - cp -r ocean-cdr-coralreef-common/code/backend/common_configurations code/backend/
    - cp -r ocean-cdr-coralreef-common/code/backend/common_utilities code/backend/
    - cp -r ocean-cdr-coralreef-common/code/backend/handlers code/backend/

pytest-coverage:
  extends: .with-common
  script:
    - pytest --cov
```

Those three `cp -r` lines appear in **three** jobs today. One edit site instead
of three.

## `default`

For things nearly every job shares:

```yaml
default:
  image: python:3.11
  tags:
    - linux-amd64-m
  retry:
    max: 2
    when: runner_system_failure
```

Any job can override. `default` suits settings; `extends` suits behaviour.

## YAML anchors

Pure YAML, no GitLab involvement:

```yaml
.common: &common
  image: python:3.11
  tags: [linux-amd64-m]

job_a:
  <<: *common
  script: [echo a]
```

`&name` defines an anchor, `*name` references it, `<<:` merges a mapping.

> Anchors only work **within one file** — they do not cross `include`
> boundaries, and the merge is shallow. `extends` works across includes and
> merges deeply. **Prefer `extends`.**

Anchors remain handy for repeated scalars and lists:

```yaml
.tags: &tags [linux-amd64-m]

job:
  tags: *tags
```

## `!reference`

Reuse one key from another job without inheriting the rest:

```yaml
.setup:
  script:
    - apt-get update
    - pip install -r requirements.txt

test:
  script:
    - !reference [.setup, script]
    - pytest
```

Composes script fragments, which neither `extends` nor anchors do cleanly.

## Structuring shared pipelines

A common organisation layout:

```
cicd-pipelines/                 (a dedicated project)
├── python/
│   ├── test.yml
│   └── lint.yml
├── docker/
│   └── kaniko-build.yml
└── sonarscan/
    └── sonarscan.yml
```

consumed as:

```yaml
include:
  - project: ocean/enablers/devops-library/cicd-pipelines
    ref: v2.3.1
    file:
      - python/test.yml
      - docker/kaniko-build.yml
```

Rules that keep this maintainable:

1. **Tag the shared project and pin the tag.** Never `ref: main`.
2. **Templates are hidden jobs.** Let consumers opt in via `extends`, rather
   than injecting jobs they did not ask for.
3. **Parameterise with variables**, with sensible defaults.
4. **Document the inputs** — a `README.md` beside each template.
5. **Changelog the shared project.** Consumers need to know what a bump means.

## Which to use

```
Across projects?                 -> include
A job shape used several times?  -> extends (hidden job)
Settings almost every job wants? -> default
One repeated scalar or list?     -> YAML anchor
One key from another job?        -> !reference
```

## Try it

In [`main/.gitlab-ci.yml`](../main/.gitlab-ci.yml):

1. Three `include` entries — which kind is each?
2. Find `extends` usages. Where are those templates defined?
3. Count how many jobs repeat `image: python:3.11` and `tags: [linux-amd64-m]`.
4. Find the three `cp -r ocean-cdr-coralreef-common/...` lines. Write the
   hidden job that would replace them.
5. Compare with [`common/.gitlab-ci.yml`](../common/.gitlab-ci.yml) — what is
   duplicated across the two repositories, and what would you extract?

## Check yourself

1. Four forms of `include`?
2. Why pin `ref:` on an included project?
3. `extends` vs YAML anchors — two differences?
4. What does a leading `.` on a job name mean?
5. When would you use `!reference`?
6. Why should shared templates be hidden jobs?

---

Previous: [07 — needs and the DAG](07-needs-and-dag.md) ·
Next: [09 — Runners](09-runners.md)
