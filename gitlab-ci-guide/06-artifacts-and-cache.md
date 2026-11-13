# 06 — Artifacts and cache

Both put files somewhere. They solve different problems and are not
interchangeable.

| | **Artifacts** | **Cache** |
|---|---|---|
| Purpose | Pass results **forward** | Reuse downloads for **speed** |
| Guaranteed? | Yes | **No** — best effort |
| Scope | Later jobs in this pipeline | Any job, any pipeline |
| Downloadable in UI | Yes | No |
| If missing | Job fails | Job just runs slower |

**Rule: if losing it breaks the build, it is an artifact. If losing it only
makes the build slower, it is a cache.**

## Artifacts

```yaml
semgrep-sast:
  script:
    - semgrep --config=auto --json --output=semgrep-results.json .
  artifacts:
    paths:
      - semgrep-results.json
    reports:
      sast: semgrep-results.json
    when: always
```

By default, **every later job automatically downloads all artifacts from
earlier stages.** That is how the submodule job's output reaches the jobs that
need it:

```yaml
submodule:
  stage: submodule
  script:
    - git submodule update --recursive --remote
  artifacts:
    paths:
      - $GIT_PROJECT_DIRECTORY/ocean-cdr-coralreef-common
      - $GIT_PROJECT_DIRECTORY/ocean-ontology
```

Every subsequent job can then `cp -r ocean-cdr-coralreef-common/...` — those
directories exist because they arrived as artifacts, not because the job
fetched them.

### `when`

```yaml
artifacts:
  when: always        # on_success (default) | on_failure | always
```

`always` is right for anything you need in order to debug a failure — test
reports, scan output, diffs. `main/.gitlab-ci.yml` uses it on `semgrep-sast`.

The same thinking applies to the format job:

```yaml
yapf_format:
  script:
    - |
      if git diff --name-only | tee ${CI_PIPELINE_ID}_yapf_diff.txt | grep -q .; then
        exit 1
      fi
  artifacts:
    paths:
      - $GIT_PROJECT_DIRECTORY/code/backend/apis/tests/${CI_PIPELINE_ID}_yapf_diff.txt
```

The diff is written, the job fails, and you download the diff to see what to
reformat. Note the artifact has no `when: always` — with the default
`on_success` the file is **not** kept when the job fails, which is precisely
when you want it. Worth fixing.

### `expire_in`

```yaml
artifacts:
  paths:
    - index.html
  expire_in: 1 week
```

Artifacts consume storage. Set an expiry on anything that is not a release
artefact. `1 week`, `30 days`, `never`.

### `name`

```yaml
artifacts:
  name: coverage.xml
  paths:
    - $GIT_PROJECT_DIRECTORY/code/backend/apis/tests/coverage.xml
```

Names the downloadable zip. Useful with variables:

```yaml
name: "$CI_JOB_NAME-$CI_COMMIT_REF_SLUG"
```

### `reports`

Special artifact types GitLab understands and renders:

```yaml
artifacts:
  reports:
    sast: semgrep-results.json          # security findings in the MR widget
    junit: report.xml                   # test results
    coverage_report:
      coverage_format: cobertura
      path: coverage.xml
    dependency_scanning: gl-dependency-scanning-report.json
```

`reports` do more than store a file — they feed the merge request widget, so
reviewers see new vulnerabilities and failing tests without opening logs.

Note `reports` artifacts are **not** downloadable the way `paths` artifacts are;
list a file in both if you want each behaviour.

### Controlling what you receive

Jobs download everything from earlier stages by default. In a large pipeline
that is wasted time:

```yaml
job:
  dependencies: []          # download nothing
```

```yaml
job:
  dependencies:
    - submodule             # only this job's artifacts
```

With `needs`, artifacts follow the `needs` list instead:

```yaml
sonar-report:
  needs:
    - job: sonarqube-scan
    - job: submodule
      artifacts: true       # explicitly want this one's files
```

That is from `main/.gitlab-ci.yml`, and it is the modern form. Chapter 07.

## Cache

```yaml
pre-commit-hooks:
  variables:
    PRE_COMMIT_HOME: ${CI_PROJECT_DIR}/.cache/pre-commit
  cache:
    key:
      files:
        - .pre-commit-config.yaml
    paths:
      - ${PRE_COMMIT_HOME}
```

A well-built cache. Three things to notice:

**The cache path is inside `CI_PROJECT_DIR`.** Runners can only cache paths
inside the project directory, which is why `PRE_COMMIT_HOME` is redirected
there.

**`key:files:`** derives the cache key from a file's contents. Change
`.pre-commit-config.yaml` and you get a fresh cache automatically; leave it
alone and every pipeline reuses the hooks. Exactly right — the cache is only
valid for that set of hooks.

**It caches a tool's own store**, not the repository.

### Cache keys

```yaml
cache:
  key: "$CI_COMMIT_REF_SLUG"       # per branch
  key: "$CI_JOB_NAME"              # per job
  key:
    files:
      - requirements.txt           # per lockfile content — best
  paths:
    - .cache/pip
```

Keying on a lockfile is almost always what you want: the cache is reused while
dependencies are unchanged, and invalidated automatically when they change.

### `policy`

```yaml
cache:
  policy: pull          # read only — do not re-upload
  policy: push          # write only
  policy: pull-push     # default
```

Jobs that only consume a cache should use `pull`, saving the upload.

### Typical caches

```yaml
# Python
cache:
  key: { files: [requirements.txt] }
  paths: [.cache/pip]
variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

# Node
cache:
  key: { files: [package-lock.json] }
  paths: [.npm]
```

> A cache is best effort. A job must still work with an empty cache — never
> treat it as a dependency.

## Artifacts vs cache, decided

| Need | Use |
|------|-----|
| Build output used by a later job | Artifact |
| Test or coverage report | Artifact (`reports`) |
| Something to download from the UI | Artifact |
| `pip`/`npm` downloads | Cache |
| Pre-commit hook environments | Cache |
| Compiler object files | Cache |

The submodule checkout in `main/.gitlab-ci.yml` is correctly an **artifact** —
later jobs genuinely cannot work without it.

## Pitfalls

**Absolute paths.** Artifact paths should be relative to `CI_PROJECT_DIR`.
`main/.gitlab-ci.yml` uses `$GIT_PROJECT_DIRECTORY`, which is an absolute path
that happens to equal the project directory. It works, but it is fragile — the
literal path is hardcoded in several places and breaks if the project moves.
`$CI_PROJECT_DIR` is the portable choice.

**Caching the wrong directory.** `pip` caches to `~/.cache/pip` by default,
outside the project directory, so caching it silently does nothing. Redirect it
with `PIP_CACHE_DIR` first — the same move this pipeline makes for
`PRE_COMMIT_HOME`.

**Artifacts that do not expire.** Default retention plus large artifacts fills
storage quotas.

**Expecting artifacts from the same stage.** Only *earlier* stages are
downloaded automatically. Use `needs` to pull from a parallel job.

## Try it

In [`main/.gitlab-ci.yml`](../main/.gitlab-ci.yml):

1. Which job produces the artifact nearly every other job depends on?
2. Why is the pre-commit cache keyed on `.pre-commit-config.yaml`?
3. `semgrep-sast` sets `when: always` — why does `yapf_format` need it more?
4. Which job uses `expire_in`, and should the others?
5. Which jobs could set `dependencies: []` and save download time?

## Check yourself

1. One sentence: artifact vs cache.
2. Which jobs automatically receive a job's artifacts?
3. What does `when: always` do, and when do you need it?
4. What is `key:files:` for?
5. Why must a cache path be inside `CI_PROJECT_DIR`?
6. What makes `reports:` different from `paths:`?

---

Previous: [05 — Rules and workflow](05-rules-and-workflow.md) ·
Next: [07 — needs and the DAG](07-needs-and-dag.md)
