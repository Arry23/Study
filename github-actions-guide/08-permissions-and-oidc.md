# 08 — Permissions and OIDC

Two mechanisms, one principle: a job should hold the least authority that lets
it do its work.

## `GITHUB_TOKEN` permissions

Every run gets an automatic token. The `permissions:` block decides what it can
do.

```yaml
permissions:
  contents: read
```

Declared at workflow level, this is the default for every job. **Anything not
listed is set to `none`** — so one line removes write access to issues, pull
requests, packages, deployments and everything else.

Raise it only where needed:

```yaml
jobs:
  check:
    # inherits contents: read

  plan:
    permissions:
      contents: read
      id-token: write       # only this job can request an OIDC token
```

That is `arifa-infra`'s structure, and it is the right shape: restrictive
default, narrow exception.

### Available scopes

```yaml
permissions:
  actions: read|write|none
  checks: read|write|none
  contents: read|write|none
  deployments: read|write|none
  id-token: write|none          # OIDC
  issues: read|write|none
  packages: read|write|none
  pull-requests: read|write|none
  security-events: read|write|none
  statuses: read|write|none
```

Shortcuts:

```yaml
permissions: read-all
permissions: write-all
permissions: {}          # no permissions at all
```

### Why it matters

Without a `permissions:` block the token may get broad write access depending on
repository settings. A compromised dependency in any step can then push commits,
open pull requests, or publish packages — using a token GitHub handed it
automatically.

Setting `contents: read` at workflow level costs nothing and removes that class
of risk.

> Note also `persist-credentials: false` on checkout (chapter 05). Together they
> mean no token is left in `.git/config` and the token in the environment can
> only read.

## OIDC

The problem: a workflow needs AWS credentials. The old answer was an IAM user
whose access key is stored as a secret — long-lived, broadly scoped, and
whoever reads it once has it forever.

OIDC replaces it. GitHub mints a short-lived, signed token describing the run.
AWS trusts GitHub as an identity provider and exchanges it for temporary
credentials. **Nothing long-lived is stored anywhere.**

### The workflow side

```yaml
permissions:
  contents: read
  id-token: write           # REQUIRED to request the token

steps:
  - uses: aws-actions/configure-aws-credentials@e1253824e5c10ff9df46874f81ed3ec929e19cfd # v6.3.0
    with:
      role-to-assume: ${{ vars.AWS_PLAN_ROLE_ARN }}
      role-session-name: arifa-plan-${{ github.run_id }}
      aws-region: eu-central-1
```

Without `id-token: write` the token request fails. It is the commonest OIDC
error.

`role-session-name` with `github.run_id` makes every AWS API call in CloudTrail
traceable to a single workflow run.

### The AWS side

Covered in depth in
[the Terraform guide, chapter 13](../terraform-guide/13-cicd-and-oidc.md),
because `arifa-infra` defines it in Terraform. The essential part is the trust
policy:

```hcl
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
```

> **The `sub` condition is the security boundary.** Without it, *any* GitHub
> repository anywhere could assume your role.
>
> Use `StringEquals`, never `StringLike` with a wildcard. `repo:myorg/*` trusts
> every repository in the organisation, including ones a contractor can push to.

### `sub` claim forms

```
repo:OWNER/REPO:ref:refs/heads/main       # a specific branch
repo:OWNER/REPO:pull_request              # any PR run
repo:OWNER/REPO:environment:production    # a specific environment
```

Pinning to `environment:production` combined with required reviewers means the
role can only be assumed after a human approves.

`arifa-infra` records a real-world wrinkle:

```hcl
# If AssumeRoleWithWebIdentity is denied although this looks right, read the real
# `sub` claim in CloudTrail: some organisations use the immutable-ID form
# repo:<owner>@<ownerId>/<repo>@<repoId>:pull_request.
```

Read the actual claim rather than guessing.

### Other clouds

Same pattern:

```yaml
# Azure
- uses: azure/login@<sha>
  with:
    client-id: ${{ vars.AZURE_CLIENT_ID }}
    tenant-id: ${{ vars.AZURE_TENANT_ID }}
    subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

# GCP
- uses: google-github-actions/auth@<sha>
  with:
    workload_identity_provider: projects/.../providers/github
    service_account: ci@project.iam.gserviceaccount.com
```

All need `id-token: write`, and all have an equivalent subject condition that
must pin the repository.

## Least privilege for the cloud role

A role that can plan should not be able to apply. `arifa-infra` starts from
`ReadOnlyAccess` and subtracts:

```hcl
statement {
  sid       = "DenySecretValues"
  effect    = "Deny"
  actions   = ["secretsmanager:GetSecretValue", "secretsmanager:BatchGetSecretValue"]
  resources = ["*"]
}
```

An explicit `Deny` always beats an `Allow`, so this holds even though
`ReadOnlyAccess` grants it.

It also notes that a plan is not purely read-only — it takes a state lock:

```hcl
statement {
  sid       = "TakeStateLock"
  actions   = ["s3:PutObject", "s3:DeleteObject"]
  resources = ["${aws_s3_bucket.state.arn}/envs/*.tflock"]
}
```

Narrowly scoped to `*.tflock`.

## Separating plan from apply

```yaml
permissions:
  contents: read

jobs:
  plan:
    permissions:
      contents: read
      id-token: write
    # assumes a READ-ONLY role

  apply:
    needs: plan
    environment: production      # required reviewers
    permissions:
      contents: read
      id-token: write
    # assumes a WRITE role, only reachable after approval
```

Two roles, two trust policies. The plan job's credentials cannot change
anything, so compromising the PR path does not compromise infrastructure.

`arifa-infra` goes further and has no apply job at all:

> Nothing in CI can apply.

For a shared platform account with a small team, that is a defensible choice.

## Forks

Fork pull requests get:

- A **read-only** `GITHUB_TOKEN`
- **No** secrets
- **No** OIDC token

That is why the trust policy comment says *"Fork pull requests get no OIDC
token."* Do not weaken these defaults. `pull_request_target` bypasses them and
is dangerous — chapter 09.

## Checklist

- [ ] `permissions:` at workflow level, defaulting to `contents: read`
- [ ] Elevated permissions only on the jobs that need them
- [ ] `id-token: write` only on jobs doing cloud auth
- [ ] `persist-credentials: false` on checkout
- [ ] OIDC instead of stored cloud keys
- [ ] Trust policy pins the repository with `StringEquals`
- [ ] Plan and apply use different roles
- [ ] Apply gated by an environment with reviewers
- [ ] Fork PRs receive no secrets

## Try it

In [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. Find both `permissions:` blocks. What does each grant?
2. Why does only `plan` need `id-token: write`?
3. Why is `role-session-name` built from `github.run_id`?
4. The workflow uses no `secrets.*`. How does it reach AWS?
5. In [`bootstrap/github_oidc.tf`](../arifa-infra/bootstrap/github_oidc.tf),
   find the `sub` condition. What would `StringLike` with `repo:myorg/*` allow?

## Check yourself

1. What does a workflow-level `permissions:` block do to unlisted scopes?
2. Why is OIDC better than a stored access key?
3. Which permission does OIDC require?
4. What is the `sub` condition for?
5. Why does a read-only plan role still need `s3:PutObject`?
6. What does a fork PR receive?

---

Previous: [07 — Variables and secrets](07-variables-and-secrets.md) ·
Next: [09 — Security hardening](09-security-hardening.md)
