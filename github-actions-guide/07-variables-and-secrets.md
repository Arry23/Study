# 07 — Variables and secrets

Three storage places, and the difference matters.

| | Where | Masked | Use for |
|---|---|---|---|
| `env:` | The workflow file | No | Non-sensitive config visible in git |
| `vars.*` | Repository/org/environment settings | No | Non-sensitive config that varies |
| `secrets.*` | Repository/org/environment settings | **Yes** | Credentials |

## `env:`

```yaml
env:
  TF_IN_AUTOMATION: "true"
  TF_INPUT: "0"
  TFLINT_VERSION: "0.65.0"
  TFLINT_SHA256: "d4c545ed2e29d6338604b586f34f544f380f70f243aa72f7b484713835eaec61"
```

Three scopes — workflow, job, step. Nearest wins:

```yaml
env:
  LEVEL: workflow

jobs:
  build:
    env:
      LEVEL: job
    steps:
      - run: echo "$LEVEL"        # "job"
      - run: echo "$LEVEL"        # "step"
        env:
          LEVEL: step
```

Read them as shell variables (`$TFLINT_VERSION`) or expressions
(`${{ env.TFLINT_VERSION }}`). In a `run:` block prefer the shell form — it
avoids the injection class entirely (chapter 09).

Putting version pins in `env:` keeps them in one place:

```yaml
# Pinned tool versions and archive checksums, checked 2026-11-02.
TFLINT_VERSION: "0.65.0"
TFLINT_SHA256: "d4c545ed..."
```

## `vars.*`

Repository variables, Settings → Secrets and variables → Actions → Variables.

```yaml
plan:
  if: vars.AWS_PLAN_ROLE_ARN != ''
  steps:
    - uses: aws-actions/configure-aws-credentials@<sha>
      with:
        role-to-assume: ${{ vars.AWS_PLAN_ROLE_ARN }}
```

Why a variable and not a secret:

1. **An IAM role ARN is not sensitive.** It is useless without permission to
   assume it, and the trust policy is the real control.
2. **You can test it in a job-level `if:`** — `secrets.*` is not available there.
3. **It appears in logs**, which helps debugging.

That second point is the practical one. This is how the workflow stays useful
before the AWS side exists: no variable, no plan job, static checks still run.

Scopes: repository, organisation, environment. Environment variables let `dev`
and `prod` supply different values to the same workflow.

## `secrets.*`

```yaml
- run: ./deploy.sh
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
```

- Write-only in the UI — you cannot read one back
- Automatically masked in logs
- Not passed to workflows triggered by a fork's `pull_request`
- Not available in job-level `if:`

### Masking is not containment

GitHub masks exact matches of a secret's value. It does not mask transformed
versions:

```yaml
- run: echo "${{ secrets.TOKEN }}" | base64     # NOT masked
- run: echo "${{ secrets.TOKEN }}" | rev        # NOT masked
```

And a secret passed to a process that prints it in pieces is not masked either.

**Treat masking as a safety net, not a control.** Never deliberately print a
secret.

> Compare GitLab, where secrets are ordinary variables with a "masked" checkbox
> that can be off — which is how `echo "token ${PERSONAL_TOKEN}"` ended up in
> `main/.gitlab-ci.yml`. GitHub's explicit `secrets.` namespace makes such a
> line obvious in review. See
> [the GitLab guide, chapter 10](../gitlab-ci-guide/10-secrets-and-security.md).

### Secrets need no declaration

Unlike GitLab, where variables are injected into the environment automatically,
GitHub requires you to pass a secret explicitly:

```yaml
- run: ./deploy.sh              # API_TOKEN not available
- run: ./deploy.sh              # available
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
```

More verbose, and much better: you can see from the step which secrets it can
reach.

## `GITHUB_TOKEN`

Created automatically for every run, scoped to the repository, expires when the
run ends.

```yaml
- name: Quality gates (scripts/check.sh)
  env:
    # tflint --init downloads the AWS ruleset from GitHub; the token avoids rate limits.
    GITHUB_TOKEN: ${{ github.token }}
  run: scripts/check.sh
```

A nice use: `tflint --init` fetches rulesets from the GitHub API, and
unauthenticated requests are rate-limited per IP — painful on a shared corporate
NAT. The token raises the limit without granting anything extra.

Its permissions are controlled by `permissions:`. Chapter 08.

## Environments

Named deployment targets with their own secrets and protection rules:

```yaml
jobs:
  deploy:
    environment: production
    steps:
      - run: ./deploy.sh
        env:
          TOKEN: ${{ secrets.DEPLOY_TOKEN }}   # the production one
```

Settings → Environments lets you add **required reviewers**, a wait timer, and
branch restrictions. A job targeting that environment pauses until approved.

This is the correct way to gate an `apply`. `arifa-infra` chooses not to apply
from CI at all — a stricter stance — but an environment with required reviewers
is the standard middle ground.

## Precedence

Highest wins:

1. Step `env:`
2. Job `env:`
3. Workflow `env:`
4. Environment variables (`vars`) for the job's environment
5. Repository `vars`
6. Organisation `vars`

Secrets resolve similarly: environment, then repository, then organisation.

## Default environment variables

Always present, no declaration:

```bash
GITHUB_WORKSPACE      # checkout path
GITHUB_REPOSITORY     # owner/repo
GITHUB_REF            # refs/heads/main
GITHUB_SHA            # commit
GITHUB_ACTOR          # who triggered
GITHUB_RUN_ID
GITHUB_EVENT_NAME
GITHUB_ENV            # file: set vars for later steps
GITHUB_OUTPUT         # file: set step outputs
GITHUB_PATH           # file: add to PATH
GITHUB_STEP_SUMMARY   # file: Markdown for the run summary
RUNNER_TEMP           # temp dir
RUNNER_OS
CI                    # always "true"
```

The four "file" ones are written to, not read:

```yaml
- run: echo "version=1.2.3" >> "$GITHUB_OUTPUT"
- run: echo "$bin" >> "$GITHUB_PATH"
- run: echo "### Result" >> "$GITHUB_STEP_SUMMARY"
```

`CI=true` is what lets `arifa-infra`'s Makefile skip setting `AWS_PROFILE`:

```makefile
ifndef CI
export AWS_PROFILE ?= arifa
endif
```

## Choosing

```
Visible in git, same everywhere?        -> env:
Varies by repo/environment, not secret? -> vars.*
A credential?                           -> secrets.*
Needs approval before use?              -> environment secret
```

## Checklist

- [ ] No credentials in workflow files
- [ ] Non-sensitive config in `vars`, not `secrets`
- [ ] Secrets passed explicitly via `env:` on the step that needs them
- [ ] No step prints or transforms a secret
- [ ] Production secrets behind an environment with reviewers
- [ ] `GITHUB_TOKEN` preferred over a PAT where sufficient
- [ ] `permissions:` restricts the token (chapter 08)

## Try it

In [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. List everything in `env:`. Why is each there rather than in `vars`?
2. Why is `AWS_PLAN_ROLE_ARN` a variable rather than a secret?
3. Why does the check step set `GITHUB_TOKEN`?
4. The workflow uses no `secrets.*` at all. How does it authenticate to AWS?
5. Where is `RUNNER_TEMP` used, and why not a path in the workspace?

## Check yourself

1. Three places to store a value, and when to use each?
2. Why can't `secrets.*` be used in a job-level `if:`?
3. What does GitHub mask, and what does it miss?
4. Why must secrets be passed explicitly to a step?
5. What is `GITHUB_TOKEN` and how long does it live?
6. How do you require human approval before a job runs?

---

Previous: [06 — Contexts and expressions](06-contexts-and-expressions.md) ·
Next: [08 — Permissions and OIDC](08-permissions-and-oidc.md)
