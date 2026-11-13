# 13 — CI/CD

## What a Terraform pipeline is for

Two things:

1. **Run the cheap checks on every change** — format, validate, lint, scan.
2. **Show the plan on the pull request**, so the diff a reviewer approves is
   the actual change to infrastructure.

Applying automatically is optional, and plenty of serious teams choose not to.

## Plan on PR, apply deliberately

`arifa-infra` is explicit:

> Nothing in CI can apply.

CI plans; a human applies from `main`. The trade-off:

- **Cost:** applying is manual.
- **Benefit:** a compromised pipeline cannot change infrastructure, and the
  credentials CI holds are read-only.

For a shared platform account with a small team, that is a sound default. Teams
that do automate apply should gate it behind a GitHub Environment with required
reviewers, and apply a **saved plan** rather than re-planning:

```bash
terraform plan -out=tfplan      # on the PR
terraform apply tfplan          # after approval — applies exactly what was reviewed
```

Re-planning at apply time means approving one thing and applying another.

## OIDC instead of access keys

The old way: create an IAM user, store `AWS_ACCESS_KEY_ID` and
`AWS_SECRET_ACCESS_KEY` as repository secrets. Those credentials are long-lived,
and anyone who can read them has them forever.

The modern way: **OpenID Connect**. GitHub mints a short-lived token describing
the workflow run; AWS trusts GitHub as an identity provider and exchanges it for
temporary credentials. Nothing long-lived is stored anywhere.

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url            = "https://token.actions.githubusercontent.com"
  client_id_list = ["sts.amazonaws.com"]
}
```

The trust policy is where the security lives:

```hcl
data "aws_iam_policy_document" "github_plan_trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github[0].arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    # Only pull_request runs in this repository. Fork pull requests get no OIDC token.
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:${local.github_repository}:pull_request"]
    }
  }
}
```

> **The `sub` condition is the one that matters.** Without it, *any* GitHub
> repository in the world could assume your role. It must pin the repository,
> and should pin the event type or branch too.
>
> Use `StringEquals`, not `StringLike` with a wildcard. `repo:myorg/*` trusts
> every repository in the organisation.

The workflow needs permission to request the token:

```yaml
permissions:
  contents: read
  id-token: write        # required for OIDC
```

```yaml
- uses: aws-actions/configure-aws-credentials@e1253824e5c10ff9df46874f81ed3ec929e19cfd # v6.3.0
  with:
    role-to-assume: ${{ vars.AWS_PLAN_ROLE_ARN }}
    role-session-name: arifa-plan-${{ github.run_id }}
    aws-region: eu-central-1
```

## Least privilege for a plan role

A plan reads everything it manages, so `ReadOnlyAccess` is the obvious baseline.
But `ReadOnlyAccess` also reads *data* — DynamoDB rows, log events, S3 objects —
which a plan never needs.

`arifa-infra` layers explicit denies on top:

```hcl
statement {
  sid       = "DenySecretValues"
  effect    = "Deny"
  actions   = ["secretsmanager:GetSecretValue", "secretsmanager:BatchGetSecretValue"]
  resources = ["*"]
}

statement {
  sid           = "DenyDataReads"
  effect        = "Deny"
  actions       = ["s3:GetObject", "s3:GetObjectVersion"]
  not_resources = ["${aws_s3_bucket.state.arn}/envs/*"]
}

statement {
  sid     = "DenyOtherDataReads"
  effect  = "Deny"
  actions = [
    "dynamodb:GetItem", "dynamodb:Query", "dynamodb:Scan",
    "logs:FilterLogEvents", "logs:GetLogEvents",
    "kinesis:GetRecords", "athena:GetQueryResults",
  ]
  resources = ["*"]
}
```

An explicit `Deny` always beats an `Allow` in IAM, so these win over
`ReadOnlyAccess`.

The comment is honest about the limits:

> That list is not complete: check it against the current ReadOnlyAccess before
> enabling.

Worth imitating. A policy that claims completeness it does not have is worse
than one that flags its own gap.

### A plan is not purely read-only

It takes the state lock, which is a write:

```hcl
statement {
  sid       = "TakeStateLock"
  actions   = ["s3:PutObject", "s3:DeleteObject"]
  resources = ["${aws_s3_bucket.state.arn}/envs/*.tflock"]
}
```

A strictly read-only role fails at the lock step. Scope the write narrowly to
`*.tflock`.

### Where the CI role lives

In `bootstrap/`, not in an environment root:

> The CI role lives here, not in an environment root, so a bad environment apply
> cannot delete the pipeline's own credentials.

Keep the thing that grants access separate from the thing it grants access to.

## Pinning actions

```yaml
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
```

A full commit SHA, with the human-readable tag in a comment. **Tags are
mutable** — `v4` can be repointed at malicious code by anyone who compromises
the action's repository. A SHA cannot.

This is not theoretical. `arifa-infra`'s workflow header records why it avoids
`trivy-action` entirely:

> trivy-action is not used on purpose: its tags were hijacked in March 2026
> (GHSA-69fq-xp46-6x23).

Where no trusted action exists, install the tool from a pinned release archive
and verify the checksum:

```yaml
env:
  TFLINT_VERSION: "0.65.0"
  TFLINT_SHA256: "d4c545ed2e29d6338604b586f34f544f380f70f243aa72f7b484713835eaec61"

# ...
curl -fsSLo tflint.zip "https://github.com/terraform-linters/tflint/releases/download/v${TFLINT_VERSION}/tflint_linux_amd64.zip"
echo "${TFLINT_SHA256}  tflint.zip" | sha256sum --check --strict
unzip -q -o tflint.zip tflint -d "$bin"
```

## Other hardening worth copying

**Minimal permissions, raised per job.**

```yaml
permissions:
  contents: read        # workflow default

jobs:
  plan:
    permissions:
      contents: read
      id-token: write   # only this job gets it
```

**Do not persist credentials after checkout.**

```yaml
- uses: actions/checkout@<sha>
  with:
    persist-credentials: false
```

**Never interpolate untrusted input into a `run:` block.**

```yaml
# Vulnerable: a PR title can contain shell metacharacters
run: echo "${{ github.event.pull_request.title }}"

# Safe: pass through the environment
env:
  TITLE: ${{ github.event.pull_request.title }}
run: echo "$TITLE"
```

`arifa-infra` states this as a rule and follows it — even the matrix value goes
through `env`:

```yaml
- name: terraform plan
  env:
    TF_ENV: ${{ matrix.env }}
  run: terraform -chdir="envs/$TF_ENV" plan -no-color -lock-timeout=5m
```

**Cancel superseded runs.**

```yaml
concurrency:
  group: terraform-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

Note this only serialises runs *within* one PR. Two different PRs can still
plan at once and contend for the same state lock — hence `-lock-timeout=5m`.

**Set timeouts.** `timeout-minutes: 15` stops a hung job burning runner minutes.

## Matrix over environments

```yaml
strategy:
  fail-fast: false
  matrix:
    env: [dev]
```

`fail-fast: false` so a failure in one environment still reports the others. Add
`staging` and `prod` to the list as they appear.

## Gating a job

```yaml
plan:
  needs: check
  if: vars.AWS_PLAN_ROLE_ARN != ''
```

`needs: check` means no plan runs until the static gates pass — no point
spending AWS calls on unformatted code.

The `if` makes the whole job conditional on a repository variable. CI runs
static checks only until someone sets `AWS_PLAN_ROLE_ARN`, so the pipeline is
useful before the AWS side is ready.

## Surfacing the plan

```yaml
- name: Write the plan to the job summary
  if: always()
  run: |
    {
      echo "### terraform plan: envs/${TF_ENV}"
      grep -E '^(Plan:|No changes\.|Error:)' "$plan" || echo "No plan summary line."
      echo "<details><summary>Full plan</summary>"
      echo '```'
      head -c 60000 "$plan"
      echo '```'
      echo "</details>"
    } >> "$GITHUB_STEP_SUMMARY"
```

Summary line first, full plan collapsed. `if: always()` so a failure still
reports something.

Job summary rather than a PR comment means no write permission on the pull
request is needed. The trade-off is that the plan is one click further away.

> Plan output can contain resource names and configuration. Terraform masks
> values marked sensitive, but anyone with repository read access can see the
> summary. Worth a conscious decision rather than a default.

## What is missing here

Two gaps in `arifa-infra`'s pipeline, both easy to close:

**No trigger on `main`.** The workflow runs only on `pull_request`. A direct
push, an admin merge, or a merge commit that differs from what was planned gets
no validation. Add:

```yaml
on:
  pull_request:
    paths: [...]
  push:
    branches: [main]
```

**No drift detection.** Nothing ever plans on a schedule, so a console change
surfaces only when someone opens an unrelated PR. That matters here because the
design deliberately expects out-of-band changes — secret *values* are managed
outside Terraform. A nightly plan posting to the summary would close it:

```yaml
on:
  schedule:
    - cron: "0 6 * * 1-5"
```

## Try it

```bash
cd arifa-infra
cat .github/workflows/terraform.yml
```

Find: every pinned SHA, the two `permissions` blocks, the `if` that gates the
plan job, and every place a variable is passed through `env` instead of being
interpolated.

## Check yourself

1. Why is OIDC better than storing access keys?
2. What does the `sub` condition do, and what happens without it?
3. Why does a read-only plan role still need `s3:PutObject`?
4. Why does an explicit `Deny` beat `ReadOnlyAccess`?
5. Why pin an action to a SHA rather than a tag?
6. Why does the CI role live in `bootstrap/`?
7. Why pass `matrix.env` through `env:` instead of inlining it in `run:`?
8. What does `needs: check` buy you?

---

Previous: [12 — Quality gates and testing](12-quality-gates-and-testing.md) ·
Next: [14 — Security and secrets](14-security-and-secrets.md)
