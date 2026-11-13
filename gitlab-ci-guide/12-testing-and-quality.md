# 12 — Testing and code quality

The pipeline in this workspace runs six distinct quality jobs. This chapter
walks through what each is for, and where the current setup undermines itself.

| Job | Checks |
|-----|--------|
| `yapf_format` | Python formatting |
| `pre-commit-hooks` | All pre-commit hooks |
| `trufflehog` | Secrets in git history |
| `semgrep-sast` | Static security analysis |
| `pytest-coverage` | Tests and coverage |
| `sonarqube-scan` | Quality metrics and gate |
| `gemnasium-python-dependency_scanning` | Vulnerable dependencies |

## Formatting

```yaml
yapf_format:
  stage: format
  image: python:3.11
  before_script:
    - pip install yapf
  script:
    - yapf --recursive --in-place --verbose .
    - |
      if git diff --name-only . ':(exclude)ocean-cdr-coralreef-common' | tee ${CI_PIPELINE_ID}_yapf_diff.txt | grep -q .; then
        echo "Differences found, see yapf_diff.txt"
        exit 1
      else
        echo "Code already formatted"
      fi
  artifacts:
    paths:
      - $GIT_PROJECT_DIRECTORY/code/backend/apis/tests/${CI_PIPELINE_ID}_yapf_diff.txt
  allow_failure: true
```

The pattern: reformat in place, then ask git whether anything moved. Non-empty
diff means the committed code was not formatted.

`:(exclude)` is git pathspec magic, excluding the vendored submodules from the
comparison.

Two problems:

**`allow_failure: true`** means the `exit 1` achieves nothing. The job goes
orange and the pipeline continues.

**The artifact is not kept on failure.** Default `when: on_success`, so the diff
file — the whole point — is discarded exactly when you need it. It needs:

```yaml
artifacts:
  when: always
```

A simpler alternative avoids the diff dance entirely:

```yaml
script:
  - yapf --recursive --diff .      # non-zero exit if reformatting needed
```

## pre-commit

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
  before_script:
    - pip install pre-commit
    - git config --global --add safe.directory ${CI_PROJECT_DIR}
  script:
    - cp ${CI_PROJECT_DIR}/ocean-cdr-coralreef-common/.pre-commit-config.yaml .
    - cp ${CI_PROJECT_DIR}/ocean-cdr-coralreef-common/pyproject.toml .
    - pre-commit run --all-files --show-diff-on-failure
```

Well put together. The cache is keyed on the hook config, so hook environments
are rebuilt only when the hooks change (chapter 06). `--show-diff-on-failure`
prints what would change, so the log is actionable.

`git config --global --add safe.directory` works around git's ownership check,
which fires when the runner user differs from the checkout owner — a very
common CI papercut.

Running pre-commit in CI **and** locally is the right combination: local hooks
give fast feedback, CI guarantees nobody skipped them with `--no-verify`.

## Tests and coverage

```yaml
pytest-coverage:
  script:
    - pytest --cov-report term-missing --cov-report xml:coverage.xml --cov --cov-config=../../../../.coveragerc
  after_script:
    - python3.11 $GIT_PROJECT_DIRECTORY/installables/gitlab_ci_update_sonarqube_coverage.py
    - xmllint --noout $GIT_PROJECT_DIRECTORY/code/backend/apis/tests/coverage.xml
  artifacts:
    name: coverage.xml
    paths:
      - $GIT_PROJECT_DIRECTORY/code/backend/apis/tests/coverage.xml
  extends: [.get-coverage]
  allow_failure: true
```

Two coverage outputs: `term-missing` for the human reading the log,
`xml:coverage.xml` for SonarQube.

`after_script` post-processes and then validates the XML with `xmllint --noout`
— a nice touch, since a malformed report otherwise fails much later inside
SonarQube with a worse error.

`extends: [.get-coverage]` pulls in a shared template that typically supplies
the `coverage:` regex GitLab uses to display a percentage on the MR.

> **`allow_failure: true` on the test job.** Failing tests do not block the
> pipeline or the merge request. Whatever the reason — a flaky suite, a
> migration in progress — this needs a deadline. A test suite that cannot fail
> the build stops being read, and within a month nobody notices it is red.
>
> Same lesson as the `check.sh` bug in `arifa-infra`: **a gate that cannot fail
> is worse than no gate, because people trust it.**

### Surfacing results properly

Adding a JUnit report puts individual failures in the MR widget:

```yaml
script:
  - pytest --junitxml=report.xml --cov --cov-report=xml:coverage.xml
artifacts:
  when: always
  reports:
    junit: report.xml
    coverage_report:
      coverage_format: cobertura
      path: coverage.xml
```

Reviewers then see *which* test failed without opening a log.

## SonarQube

```yaml
sonarqube-scan:
  stage: sonarqube
  extends: .sonarqube-check

sonar-report:
  stage: sonar-report
  needs:
    - job: sonarqube-scan
    - job: submodule
      artifacts: true
  script:
    - python generate_sonar_report.py --output index.html --json-output data.json
    - git clone --depth 1 --branch "$REPORT_DEST_BRANCH" "https://oauth2:${SONAR_REPORT_REPO_TOKEN}@${REPORT_DEST_REPO}" report_dest
    - cp index.html "${TARGET_DIR}/index.html"
    - git add -A
    - |
      if git diff --cached --quiet; then
        echo "No report changes to commit."
      else
        git commit -m "Sonar report: ${CI_PROJECT_NAME} pipeline ${CI_PIPELINE_ID} (${CI_COMMIT_SHORT_SHA})"
        git push origin "$REPORT_DEST_BRANCH"
      fi
  rules:
    - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
      when: never
    - if: $CI_PIPELINE_SOURCE == 'push' && $CI_COMMIT_BRANCH == 'develop'
      when: on_success
```

Several good decisions here:

**Scoped by rules.** Publishing happens only on `develop` pushes, never on MRs
— otherwise every MR would rewrite a shared repository.

**Commits only when something changed.** `git diff --cached --quiet` prevents a
stream of empty commits.

**`needs` with `artifacts: true`** so it starts as soon as its two real
dependencies finish.

**`--depth 1`** on the clone keeps it fast.

One thing to watch: `${SONAR_REPORT_REPO_TOKEN}` is embedded in a URL. If any
command in this job ever runs under `set -x`, that URL — token included — is
printed. Prefer a credential helper or `.netrc` over inline URL credentials.

## SAST

```yaml
semgrep-sast:
  image: returntocorp/semgrep:latest
  script:
    - semgrep --config=auto --json --output=semgrep-results.json .
  artifacts:
    paths:
      - semgrep-results.json
    reports:
      sast: semgrep-results.json
    when: always
```

`reports:sast:` is the important key — it puts findings in the MR security
widget rather than a log nobody opens. `when: always` keeps results even when
the job fails.

Two caveats: the image is unpinned (chapter 10), and `--config=auto` downloads
rules at run time, so results vary between runs and the job needs internet
access. Pin both for reproducibility.

## Dependency scanning

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
    - apt-get update && apt-get install -y libpq-dev libsasl2-dev libldap2-dev libssl-dev
  after_script:
    - rm /tmp/constraints.txt
```

GitLab's template does the work; the job only adds what the project needs. The
`PIP_CONSTRAINT` pin to `setuptools==65.5.1` is a workaround for a resolution
conflict — exactly the kind of thing that deserves a comment explaining when it
can be removed.

## Secret scanning

```yaml
trufflehog:
  script:
    - /usr/bin/trufflehog git $GIT_REPO_NAME --branch=$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME
```

Scans git **history**, not just the working tree — the right approach, since a
secret deleted in a later commit is still in history.

But `CI_MERGE_REQUEST_SOURCE_BRANCH_NAME` is empty outside MR pipelines, so on
`develop` pushes and tag builds the `--branch` flag is empty. Either restrict
the job to MR pipelines:

```yaml
rules:
  - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

or fall back:

```yaml
- /usr/bin/trufflehog git "$GIT_REPO_NAME" --branch="${CI_MERGE_REQUEST_SOURCE_BRANCH_NAME:-$CI_COMMIT_BRANCH}"
```

## Ordering gates by cost

Run the cheap things first:

```
format (seconds) → lint (seconds) → unit tests (minutes) → SAST (minutes) → build (minutes)
```

This pipeline broadly does that. With `needs` (chapter 07) you get both: fast
feedback on cheap checks *and* expensive jobs started immediately.

## What I would change

1. **Remove `allow_failure: true`** from `pytest-coverage`, with a date.
2. **Add `when: always`** to the `yapf_format` artifact.
3. **Pin** `semgrep` and `trufflehog` images, and the Semgrep ruleset.
4. **Fix the TruffleHog branch** for non-MR pipelines.
5. **Add `reports:junit:`** so test failures reach the MR widget.
6. **Delete the token `echo` lines** in `submodule` (chapter 10).
7. **Add `needs:`** to break the ten-stage chain (chapter 07).

## Try it

1. Which quality jobs can currently fail the pipeline, and which cannot?
2. Why does `yapf_format` reformat and then diff, rather than using `--diff`?
3. Why validate `coverage.xml` with `xmllint`?
4. Why does `sonar-report` never run on merge requests?
5. What does `reports:sast:` do that `paths:` does not?

## Check yourself

1. Why run pre-commit in CI as well as locally?
2. What does `--show-diff-on-failure` give you?
3. Why key the pre-commit cache on `.pre-commit-config.yaml`?
4. What is wrong with `allow_failure: true` on tests?
5. Why is `--config=auto` a reproducibility problem?
6. Why scan git history rather than the working tree?

---

Previous: [11 — Docker images and packages](11-docker-and-packages.md) ·
Next: [13 — Troubleshooting](13-troubleshooting.md)
