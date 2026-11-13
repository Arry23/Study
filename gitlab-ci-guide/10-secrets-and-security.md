# 10 — Secrets and security

CI runs with credentials to your registry, your cloud, and other repositories.
A pipeline is a high-value target.

## Never in the YAML

```yaml
# Never
variables:
  API_TOKEN: "glpat-xxxxxxxxxxxx"
```

`.gitlab-ci.yml` is in git. Anyone with read access, now or later, has that
token, and git history keeps it after you delete the line.

Secrets go in **Settings → CI/CD → Variables**, or a secrets manager.

## Masked and protected

Two independent flags:

**Masked** — the value is replaced with `[MASKED]` in job logs.

Masking has format requirements: minimum length, single line, restricted
character set. A value that fails them **cannot be masked**, and GitLab will
tell you — but only when you tick the box, not when the pipeline runs.

**Protected** — the variable is only exposed on protected branches and tags.
Production credentials should always be protected, so a feature branch cannot
reach production.

| | Masked | Protected |
|---|---|---|
| Hides in logs | yes | no |
| Restricts to protected refs | no | yes |
| Use for | every secret | production secrets |

Tick both for anything that matters.

## Masking is not containment

> **A real issue in this workspace.** The `submodule` job in
> `main/.gitlab-ci.yml` starts:
>
> ```yaml
> before_script:
>   - echo "token ${PERSONAL_TOKEN}"
>   - echo "shacl repo token ${PERSONAL_TOKEN_SHACL}"
>   - echo "CI Job Token $CI_JOB_TOKEN"
>   - echo "User Login $GITLAB_USER_LOGIN"
>   - echo "User ID $GITLAB_USER_ID"
> ```
>
> Three of those print credentials to the job log.
>
> If the variables are masked you get `[MASKED]` and no harm done. But masking
> is a UI checkbox that can be off, and values failing the format rules are
> **silently unmaskable**. Job logs are readable by every project member and are
> retained after the job.
>
> These read as leftover debugging. **Delete them.** To confirm a variable is
> set without revealing it:
>
> ```yaml
> - echo "PERSONAL_TOKEN length: ${#PERSONAL_TOKEN}"
> - '[ -n "$PERSONAL_TOKEN" ] || { echo "PERSONAL_TOKEN is empty"; exit 1; }'
> ```
>
> If a token was exposed, **rotate it**. Deleting the line does not retract the
> log, and masking it afterwards does not rewrite history.

Related: never run `env`, `printenv`, or set `CI_DEBUG_TRACE: "true"` in a
shared project. All three dump every variable.

## `CI_JOB_TOKEN` first

A short-lived token scoped to the job, valid only while it runs:

```yaml
- curl --header "JOB-TOKEN: $CI_JOB_TOKEN" --upload-file /tmp/pkg.zip "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/..."
```

```yaml
GIT_REPO_NAME: https://gitlab-ci-token:${CI_JOB_TOKEN}@code.roche.com/ocean/.../repo.git
```

Prefer it over a personal access token whenever it is sufficient — it cannot
outlive the job and is scoped to the project.

Personal tokens are the opposite: long-lived, broadly scoped, and tied to a
human who may leave. This pipeline needs `PERSONAL_TOKEN` to clone a *different*
project's submodules, which `CI_JOB_TOKEN` historically could not do. Modern
GitLab supports cross-project `CI_JOB_TOKEN` access via an allowlist — worth
checking whether those personal tokens can be eliminated entirely.

If you must use one, prefer a **project access token** or **group access
token** over a personal one: it is not tied to an individual and can be scoped
narrowly.

## Pin everything executable

Your pipeline runs other people's code with your credentials.

```yaml
# Unpinned — a different tool can run tomorrow
image: trufflesecurity/trufflehog:latest
image: returntocorp/semgrep:latest

include:
  - project: "ocean/enablers/devops-library/cicd-pipelines"
    ref: main
```

All three are mutable references. A compromised upstream tag, or just a
breaking release, runs inside your job with `CI_JOB_TOKEN` and every variable
that job can see.

```yaml
# Pinned
image: trufflesecurity/trufflehog:3.90.8
image: python@sha256:1e8e0...        # digest — immutable

include:
  - project: "ocean/enablers/devops-library/cicd-pipelines"
    ref: v2.3.1
    file: ["sonarscan/sonarscan.yml"]
```

Compare `arifa-infra`'s GitHub workflow, which pins every action to a commit SHA
and verifies a SHA-256 for every downloaded binary, citing a real tag-hijacking
incident as the reason. Same threat, same answer.

## Protect your branches

Protected branches are what make protected variables meaningful:

- Require merge requests; no direct pushes
- Require approvals (CODEOWNERS)
- Require a green pipeline before merge

Without them, anyone who can push to the default branch can edit
`.gitlab-ci.yml` to print every protected secret.

> This is worth sitting with: **whoever can modify the pipeline can exfiltrate
> every secret the pipeline can see.** Pipeline files deserve the same review as
> production code.

## Forks and merge requests

By default a merge request from a fork runs in the fork's context and does not
receive your protected variables. Do not weaken that. "Run pipelines for fork
merge requests" with protected variables exposed means any stranger's MR can
print your credentials.

## Secret scanning

This pipeline does scan — two independent ways:

```yaml
trufflehog:
  image:
    name: trufflesecurity/trufflehog:latest
    entrypoint: [""]
  script:
    - /usr/bin/trufflehog git $GIT_REPO_NAME --branch=$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME
```

```yaml
pre-commit-hooks:
  script:
    - pre-commit run --all-files --show-diff-on-failure
```

Good. Two observations:

- TruffleHog scans `$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME`, which is empty on
  non-MR pipelines — so on a tag or `develop` push the scan is not doing what
  the name suggests. Add a rule restricting it to MR pipelines, or handle the
  empty case.
- Catching a secret in CI means it is already committed. Pre-commit hooks
  locally catch it before that.

## SAST and dependency scanning

```yaml
semgrep-sast:
  script:
    - semgrep --config=auto --json --output=semgrep-results.json .
  artifacts:
    reports:
      sast: semgrep-results.json
```

```yaml
include:
  - template: Jobs/Dependency-Scanning.gitlab-ci.yml
```

`reports:sast:` is what surfaces findings in the merge request widget rather
than burying them in a log.

> `--config=auto` fetches rules from Semgrep's servers at run time. The ruleset
> can therefore change between runs, and it requires outbound internet. Pin a
> ruleset for reproducibility.

## Gates that cannot fail

```yaml
yapf_format:
  allow_failure: true

pytest-coverage:
  allow_failure: true
```

Both can go red without blocking anything. If that is intentional during a
migration, fine — but a test suite that cannot fail the pipeline is not a gate,
and the red icon will be ignored within a week.

This is the same lesson as the `check.sh` bug in `arifa-infra`: **a quality gate
that cannot fail is worse than no gate, because people trust it.**

## Checklist

**Secrets**
- [ ] No credentials in `.gitlab-ci.yml`
- [ ] All secret variables masked
- [ ] Production secrets also protected
- [ ] No `echo` of any secret — including "debug" lines
- [ ] No `env` / `printenv` / `CI_DEBUG_TRACE`
- [ ] `CI_JOB_TOKEN` used wherever sufficient
- [ ] Personal tokens replaced with project/group tokens
- [ ] Tokens rotated on any suspicion of exposure

**Supply chain**
- [ ] Images pinned to a version or digest, never `:latest`
- [ ] `include:` pinned to a tag or SHA, never `main`
- [ ] Every `include` names an explicit `file:`
- [ ] Third-party tools from a trusted registry

**Access**
- [ ] Default branch protected, MRs required
- [ ] Fork MRs do not receive protected variables
- [ ] `.gitlab-ci.yml` under CODEOWNERS review

**Scanning**
- [ ] Secret scanning runs on the right refs
- [ ] SAST results in `reports:` so they reach the MR
- [ ] Dependency scanning enabled
- [ ] Security jobs are **not** `allow_failure: true`

## Try it

In [`main/.gitlab-ci.yml`](../main/.gitlab-ci.yml):

1. Find the three lines that print credentials. Write the replacement.
2. Find every mutable reference — unpinned images and `ref: main`.
3. Which jobs run with `CI_JOB_TOKEN` in a URL?
4. Which security-relevant jobs are `allow_failure: true`?
5. When `CI_MERGE_REQUEST_SOURCE_BRANCH_NAME` is empty, what does the TruffleHog
   command actually scan?

## Check yourself

1. Masked vs protected — what does each do?
2. Why is `echo "$SECRET"` dangerous even when masked?
3. Why prefer `CI_JOB_TOKEN` over a personal token?
4. Why is `image: tool:latest` a supply-chain risk?
5. Why must `.gitlab-ci.yml` be review-protected?
6. What is wrong with `allow_failure: true` on a test job?
7. A secret reached a job log. What is the first action?

---

Previous: [09 — Runners](09-runners.md) ·
Next: [11 — Docker images and packages](11-docker-and-packages.md)
