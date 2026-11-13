# 04 — Variables

Variables are environment variables inside the job container. They come from
several places, and knowing which wins matters.

## Defining them

**Globally** — every job sees them:

```yaml
variables:
  PYTHONPATH: "/builds/ocean/.../tests:$CI_PROJECT_DIR"
```

**Per job** — overrides the global:

```yaml
pytest-coverage:
  variables:
    GIT_PROJECT_DIRECTORY: /builds/ocean/.../ocean-cdr-coralreef-main
```

**In the UI** — Settings → CI/CD → Variables. This is where secrets go. Never
in the YAML.

**At run time** — "Run pipeline" lets you supply values manually.

## Using them

```yaml
script:
  - echo $GIT_PROJECT_DIRECTORY
  - echo ${GIT_PROJECT_DIRECTORY}
  - cd ${CI_PROJECT_DIR}/code
```

Standard shell expansion, because that is literally what it is. Prefer `${...}`
when the name is adjacent to other characters:

```yaml
- curl --upload-file /tmp/"${CI_COMMIT_TAG}".zip "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/..."
```

## Precedence

Highest wins:

1. Manual "Run pipeline" values
2. Project UI variables (Settings → CI/CD)
3. Group, then instance UI variables
4. Job-level `variables:`
5. Global `variables:`
6. Predefined (`CI_*`)

The practical consequence: a UI variable **overrides** the YAML. If a job
behaves differently from what the file says, check the UI before doubting the
file.

## Predefined variables

GitLab injects a large set. The ones in daily use:

| Variable | Meaning |
|----------|---------|
| `CI_PROJECT_DIR` | Absolute path to the cloned repository |
| `CI_PROJECT_NAME` | Repository name |
| `CI_PROJECT_ID` | Numeric project ID |
| `CI_PIPELINE_ID` | This pipeline's ID |
| `CI_PIPELINE_SOURCE` | `push`, `merge_request_event`, `schedule`, `web`, `api`, `trigger` |
| `CI_COMMIT_BRANCH` | Branch name (empty on MR pipelines) |
| `CI_COMMIT_TAG` | Tag name, empty if not a tag pipeline |
| `CI_COMMIT_SHA` / `CI_COMMIT_SHORT_SHA` | Commit |
| `CI_MERGE_REQUEST_SOURCE_BRANCH_NAME` | Source branch of the MR |
| `CI_DEFAULT_BRANCH` | Usually `main` or `develop` |
| `CI_REGISTRY` / `CI_REGISTRY_IMAGE` | Container registry host and image path |
| `CI_JOB_TOKEN` | Short-lived token for this job |
| `CI_API_V4_URL` | API base URL |
| `GITLAB_USER_LOGIN` / `GITLAB_USER_ID` | Who triggered it |

All of these appear in the workspace pipelines. For example:

```yaml
rules:
  - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
    when: never
  - if: $CI_PIPELINE_SOURCE == 'push' && $CI_COMMIT_BRANCH == 'develop'
    when: on_success
```

and

```yaml
- git commit -m "Sonar report: ${CI_PROJECT_NAME} pipeline ${CI_PIPELINE_ID} (${CI_COMMIT_SHORT_SHA})"
```

`CI_COMMIT_TAG` is used as a boolean — empty on a non-tag pipeline:

```yaml
rules:
  - if: '$CI_COMMIT_TAG !~ "/^$/"'     # "tag is not empty"
```

## `CI_JOB_TOKEN`

A token scoped to one job, valid only while it runs. Use it to talk to GitLab
without a personal token:

```yaml
variables:
  GIT_REPO_NAME: https://gitlab-ci-token:${CI_JOB_TOKEN}@code.roche.com/ocean/.../ocean-cdr-coralreef-main.git
script:
  - /usr/bin/trufflehog git $GIT_REPO_NAME --branch=$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME
```

and to upload to the package registry:

```yaml
- curl --header "JOB-TOKEN: $CI_JOB_TOKEN" --upload-file /tmp/"${CI_COMMIT_TAG}".zip "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/..."
```

Prefer it over a personal access token wherever it is sufficient. It expires
with the job and is scoped to the project.

## Secret variables

Defined in the UI, never in YAML. Two flags that matter:

**Masked** — GitLab replaces the value with `[MASKED]` in job logs. Requires the
value to meet format rules (minimum length, no newlines, limited character set).

**Protected** — only exposed to jobs on protected branches and tags. This is how
you keep production credentials away from feature-branch pipelines.

```yaml
# Values come from the UI; the YAML only references them.
script:
  - git config --global url."https://oauth2:${PERSONAL_TOKEN}@code.roche.com/ocean/...".insteadOf "https://code.roche.com/ocean/..."
```

### Masking is not containment

> **Danger — a real issue in this workspace.** The `submodule` job in
> `main/.gitlab-ci.yml` prints its tokens:
>
> ```yaml
> before_script:
>   - echo "token ${PERSONAL_TOKEN}"
>   - echo "shacl repo token ${PERSONAL_TOKEN_SHACL}"
>   - echo "CI Job Token $CI_JOB_TOKEN"
> ```
>
> If those variables are masked, the log shows `[MASKED]` and you were lucky.
> If any is not masked — or fails GitLab's masking format rules, which happens
> silently — the raw token is written to a job log that every project member
> can read, and that persists after the job.
>
> These lines look like leftover debugging. **Delete them.** If you must prove a
> variable is set, print its length or a prefix:
>
> ```yaml
> - echo "PERSONAL_TOKEN length: ${#PERSONAL_TOKEN}"
> ```
>
> If a token has been exposed, rotate it. Masking after the fact does not
> un-log it. Chapter 10.

## File-type variables

A variable can be of type **File**: GitLab writes the value to a temp file and
sets the variable to the *path*. Right for certificates, kubeconfigs and service
account JSON:

```yaml
script:
  - kubectl --kubeconfig "$KUBE_CONFIG" get pods
```

## Variables in `rules`

```yaml
rules:
  - if: $CI_COMMIT_BRANCH == "develop"
  - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  - if: '$CI_COMMIT_TAG !~ "/^$/"'
  - if: $DEPLOY_ENABLED == "true"
```

Note: `$VAR` with no operator is truthy if the variable is **set and non-empty**:

```yaml
- if: ($CI_PIPELINE_SOURCE == 'push') || $CI_COMMIT_TAG
```

Comparisons are string comparisons. `=~` and `!~` take regexes.

## Expansion gotchas

**Variables referencing variables** are expanded by the runner:

```yaml
variables:
  BASE: /builds/project
  FULL: ${BASE}/code          # works
```

**But not in every context.** `rules:if` is evaluated by the GitLab server
before the job exists, so runner-only variables are not available there.

**Quoting.** These differ:

```yaml
- echo $VAR       # word-split and glob-expanded
- echo "$VAR"     # literal — usually what you want
```

Quote anything that may contain spaces or be empty.

## Debugging

```yaml
script:
  - env | sort                # DANGER: prints every variable, secrets included
  - echo "$CI_PIPELINE_SOURCE"
  - echo "branch=$CI_COMMIT_BRANCH tag=$CI_COMMIT_TAG"
```

> Never `env` or `printenv` in a shared project. Print the specific non-secret
> variable you care about.

`CI_DEBUG_TRACE: "true"` enables full shell tracing — and **prints every
variable including secrets**. Only on a private throwaway project.

## Try it

In [`main/.gitlab-ci.yml`](../main/.gitlab-ci.yml):

1. Find the one global variable. Why would `PYTHONPATH` be global?
2. Find three jobs that define the same `GIT_PROJECT_DIRECTORY`. How would
   `default:` or a global variable reduce that?
3. Find every use of `CI_JOB_TOKEN`. What is it authenticating to?
4. Find the three `echo` lines that print tokens.

## Check yourself

1. Which wins: a UI variable or a job-level `variables:` entry?
2. What is `CI_JOB_TOKEN` and why prefer it to a personal token?
3. What do **masked** and **protected** each do?
4. Why is `echo "token ${PERSONAL_TOKEN}"` dangerous even when masked?
5. What does a File-type variable hold?
6. Why can't `rules:if` see runner-only variables?

---

Previous: [03 — Jobs and scripts](03-jobs-and-scripts.md) ·
Next: [05 — Rules and workflow](05-rules-and-workflow.md)
