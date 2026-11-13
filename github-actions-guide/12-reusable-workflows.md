# 12 — Reusable workflows and composite actions

Two mechanisms for sharing. They solve different problems.

| | Reusable workflow | Composite action |
|---|---|---|
| Unit of reuse | A whole **job** | A sequence of **steps** |
| Called from | `jobs.<id>.uses` | `steps[].uses` |
| Own runner | Yes | No — runs in the caller's job |
| Can set `permissions` | Yes | No |
| Can use a matrix | Yes | No |
| Max nesting | 4 | 10 |

**Rule of thumb: a whole job → reusable workflow. A few steps repeated inside
jobs → composite action.**

## Reusable workflows

The called workflow declares its interface:

```yaml
# .github/workflows/terraform-plan.yml
name: terraform plan

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      terraform_version:
        required: false
        type: string
        default: "1.17.1"
    secrets:
      role_arn:
        required: false
    outputs:
      plan_summary:
        value: ${{ jobs.plan.outputs.summary }}

jobs:
  plan:
    runs-on: rcn-ubuntu-amd64
    permissions:
      contents: read
      id-token: write
    outputs:
      summary: ${{ steps.plan.outputs.summary }}
    steps:
      - uses: actions/checkout@<sha>
        with:
          persist-credentials: false
      - uses: hashicorp/setup-terraform@<sha>
        with:
          terraform_version: ${{ inputs.terraform_version }}
          terraform_wrapper: false
      - id: plan
        env:
          TF_ENV: ${{ inputs.environment }}
        run: terraform -chdir="envs/$TF_ENV" plan
```

The caller uses it **as a job**:

```yaml
jobs:
  plan-dev:
    uses: ./.github/workflows/terraform-plan.yml
    with:
      environment: dev
    permissions:
      contents: read
      id-token: write

  plan-prod:
    needs: plan-dev
    uses: org/shared-workflows/.github/workflows/terraform-plan.yml@v2.3.1
    with:
      environment: prod
```

Note: `uses:` at job level, with `with:` and `secrets:` — **not** `steps:`. A
job that calls a reusable workflow cannot have its own steps.

### Passing secrets

```yaml
jobs:
  deploy:
    uses: ./.github/workflows/deploy.yml
    secrets:
      token: ${{ secrets.DEPLOY_TOKEN }}
```

Or forward everything:

```yaml
    secrets: inherit
```

> `secrets: inherit` is convenient and blunt — the called workflow gets every
> secret the caller has. Prefer naming them.

### Permissions

A reusable workflow cannot grant itself more than the caller has. The caller
must declare what the callee needs:

```yaml
jobs:
  plan:
    uses: ./.github/workflows/terraform-plan.yml
    permissions:
      contents: read
      id-token: write          # required or OIDC fails inside
```

A common and confusing failure: OIDC works when the workflow is standalone, then
breaks when it is made reusable, because the caller did not pass `id-token`.

### Pin remote callees

```yaml
uses: org/shared-workflows/.github/workflows/plan.yml@v2.3.1
uses: org/shared-workflows/.github/workflows/plan.yml@3d3c42e5aac5...
```

`@main` has the same problem as an unpinned action or a GitLab `ref: main` —
someone else's change runs with your credentials, with no commit in your
history.

## Composite actions

Bundle repeated steps:

```yaml
# .github/actions/setup-terraform-tools/action.yml
name: Set up Terraform tooling
description: Installs Terraform, tflint and trivy at pinned versions

inputs:
  tflint-version:
    required: false
    default: "0.65.0"
  tflint-sha256:
    required: true
  trivy-version:
    required: false
    default: "0.76.1"
  trivy-sha256:
    required: true

outputs:
  terraform-version:
    value: ${{ steps.tf.outputs.version }}

runs:
  using: composite
  steps:
    - id: tf
      shell: bash
      run: echo "version=$(cat .terraform-version)" >> "$GITHUB_OUTPUT"

    - uses: hashicorp/setup-terraform@<sha>
      with:
        terraform_version: ${{ steps.tf.outputs.version }}
        terraform_wrapper: false

    - shell: bash
      env:
        TFLINT_VERSION: ${{ inputs.tflint-version }}
        TFLINT_SHA256: ${{ inputs.tflint-sha256 }}
      run: |
        set -euo pipefail
        bin="$RUNNER_TEMP/bin"
        mkdir -p "$bin"
        cd "$RUNNER_TEMP"
        curl -fsSLo tflint.zip "https://github.com/terraform-linters/tflint/releases/download/v${TFLINT_VERSION}/tflint_linux_amd64.zip"
        echo "${TFLINT_SHA256}  tflint.zip" | sha256sum --check --strict
        unzip -q -o tflint.zip tflint -d "$bin"
        echo "$bin" >> "$GITHUB_PATH"
```

Used as a step:

```yaml
- uses: ./.github/actions/setup-terraform-tools
  with:
    tflint-sha256: ${{ env.TFLINT_SHA256 }}
    trivy-sha256: ${{ env.TRIVY_SHA256 }}
```

**`shell:` is mandatory on every `run` step in a composite action.** Omitting it
is the most common authoring error.

### Where this would help

`arifa-infra`'s workflow repeats a block in both jobs:

```yaml
- uses: actions/checkout@<sha>
  with:
    persist-credentials: false

- name: Read the Terraform version
  id: tf
  run: echo "version=$(cat .terraform-version)" >> "$GITHUB_OUTPUT"

- uses: hashicorp/setup-terraform@<sha>
  with:
    terraform_version: ${{ steps.tf.outputs.version }}
    terraform_wrapper: false
```

A composite action would collapse that to one step in each job. With only two
jobs the duplication is tolerable; at four it is not.

## Organising shared workflows

```
shared-workflows/                       (a dedicated repository)
├── .github/workflows/
│   ├── python-test.yml                 reusable workflows
│   ├── terraform-plan.yml
│   └── docker-build.yml
└── .github/actions/
    ├── setup-python-tools/action.yml   composite actions
    └── notify-slack/action.yml
```

Rules that keep it maintainable:

1. **Tag releases and have consumers pin the tag.** Never `@main`.
2. **Document every input and output** in the `description` fields.
3. **Default sensibly** so the common case needs little configuration.
4. **Keep a changelog.** Consumers need to know what a bump means.
5. **Test the shared repository's own workflows** against themselves.

## Choosing

```
A whole job, its own runner, needs permissions or a matrix?
    -> reusable workflow

A few steps repeated inside existing jobs?
    -> composite action

Used by exactly one workflow, once?
    -> leave it inline
```

Do not extract too early. A composite action used once is indirection with no
benefit.

## Limitations

**Reusable workflows**

- Cannot be nested more than 4 deep
- The calling job cannot have its own `steps`
- `env:` from the caller is **not** inherited — pass values as inputs
- A matrix in the caller works; `strategy` inside the callee has limits

**Composite actions**

- Cannot set `permissions`
- Cannot use `if:` at the action level (steps inside can)
- Secrets must be passed as inputs — they are not automatically visible
- `shell:` required on every `run`

## Try it

Looking at [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. Which block is duplicated between `check` and `plan`?
2. Write the composite action that would remove it.
3. Would the plan job be better as a reusable workflow? What would that enable?
4. If it were reusable, which `permissions` must the caller declare?
5. Why would `secrets: inherit` be the wrong choice here?

## Check yourself

1. Reusable workflow vs composite action — two differences.
2. Where does `uses:` go for each?
3. Why must a caller declare `id-token: write` for a reusable workflow?
4. What is required on every `run` in a composite action?
5. Why pin a remote reusable workflow?
6. Why is `secrets: inherit` risky?

---

Previous: [11 — Artifacts, caching, summaries](11-artifacts-caching-summaries.md) ·
Next: [13 — Runners](13-runners.md)
