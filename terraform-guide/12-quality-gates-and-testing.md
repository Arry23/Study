# 12 — Quality gates and testing

Terraform changes real infrastructure, so the feedback loop has to be short.
Four cheap checks catch most mistakes before anything reaches AWS.

| Gate | Tool | Catches | Needs AWS? |
|------|------|---------|------------|
| Format | `terraform fmt` | Style drift | No |
| Validate | `terraform validate` | Syntax, bad references, type errors | No |
| Lint | `tflint` | Dead code, invalid AWS values, bad practice | No |
| Scan | `trivy` | Insecure settings, committed secrets | No |

None of them need credentials. All four run in about a minute.

## `terraform fmt`

```bash
terraform fmt -recursive          # rewrite
terraform fmt -check -recursive   # fail if unformatted
terraform fmt -diff               # show what would change
```

One canonical style, enforced by a tool. Never argue about alignment in review.

## `terraform validate`

```bash
terraform init -backend=false
terraform validate
```

Checks syntax, references, types and required arguments — everything knowable
without talking to a provider API.

`-backend=false` is the important flag: it skips the backend so `validate` runs
with no credentials and no state. That is what makes it a CI gate rather than a
deployment step.

It does **not** catch "that instance type does not exist" — only a plan or
tflint will.

## tflint

Rules Terraform itself does not enforce:

```bash
tflint --init                      # download the rulesets
tflint --recursive
```

Configured in `.tflint.hcl`:

```hcl
config {
  # Inspect local modules through their callers, so module inputs are checked with real values.
  call_module_type = "local"
}

plugin "terraform" {
  enabled = true
  preset  = "recommended"
  version = "0.16.1"
  source  = "github.com/terraform-linters/tflint-ruleset-terraform"
}

plugin "aws" {
  enabled = true
  version = "0.50.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}
```

Two details worth copying:

**Plugin versions are pinned.** The comment says why: *"so a laptop and CI run
the same rules whatever tflint version they have."* Unpinned rulesets mean CI
can fail on a rule that did not exist when you committed.

**`call_module_type = "local"`** makes tflint analyse modules *through their
callers*, so it checks the actual values passed in rather than the variable
declarations. That is how it can tell you `instance_type = "ml.t3.medim"` is
invalid.

Typical findings: unused declarations, deprecated syntax, missing `type` on a
variable, invalid AWS instance types, AMI IDs or ARNs.

## trivy

Misconfiguration and secret scanning:

```bash
trivy fs --config trivy.yaml --quiet .
```

`trivy.yaml`:

```yaml
severity:
  - HIGH
  - CRITICAL
exit-code: 1

scan:
  scanners:
    - misconfig
    - secret
  skip-dirs:
    - "**/.terraform"

rego:
  skip-check-update: true
```

The `secret` scanner is the one that matters most — it catches an AWS key or a
private key before it reaches git history, where removing it is painful.

`skip-check-update: true` uses the checks built into the pinned binary rather
than downloading the latest bundle. Results then change only when you change the
pin, so CI cannot break because a rule was published overnight.

### Suppressing a finding

Sometimes a finding is understood and accepted. Suppress it **inline, with a
reason**:

```hcl
# SSE-S3 until the account's KMS key rules are known: a customer-managed key
# needs a key policy that says which roles may decrypt state (docs/roadmap.md).
#trivy:ignore:AVD-AWS-0132
resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
```

Inline beats a global ignore file: the justification sits next to the code, and
it disappears automatically when the code does. A global suppression outlives
the reason for it.

## Running them together

`arifa-infra` puts all four in one script so a laptop and CI check identically:

```bash
make check              # or: bash scripts/check.sh
bash scripts/check.sh fmt       # one gate
bash scripts/check.sh validate
```

The script is the source of truth, not the Makefile — the CI runner has no
`make`, so the workflow calls `scripts/check.sh` directly.

### A bash trap worth knowing

The original version chained the gates like this:

```bash
all) fmt && validate && lint && scan ;;
```

That looks fine but is subtly broken. **`set -e` does not apply inside a
function called from a `&&` list** (except the last one). So a failing
`terraform init` inside `validate` did not abort — and because a function
returns the status of its *last* command, `validate` could return **0** after
real failures, as long as the final module validated cleanly.

The result: `make check` could report success while two roots were failing.

The fix is for each gate to track its own status and for the dispatcher to loop
rather than chain:

```bash
validate() {
  local rc=0
  for root in "${roots[@]}"; do
    if terraform -chdir="$root" init -backend=false -input=false -lockfile=readonly >/dev/null; then
      terraform -chdir="$root" validate || rc=1
    else
      echo "init failed: $root" >&2
      rc=1
    fi
  done
  return "$rc"
}

run_gates() {
  local rc=0
  require_tools "$@" || return 127
  for gate in "$@"; do
    "$gate" || rc=1
  done
  return "$rc"
}
```

Now every gate runs — so one run reports every problem — and the exit status is
non-zero if any failed.

The general lesson: **a quality gate that can silently pass is worse than no
gate**, because you trust it.

## pre-commit

Run the gates before a commit can be created:

```bash
pre-commit install
```

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: 4c4437d784e93cacb1f05f1d7bc28e4354fafd20 # frozen: v1.109.4
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: 3e8a8703264a2f4a69428a0aa4dcb512790b2c8c # frozen: v6.0.0
    hooks:
      - id: detect-private-key
      - id: check-merge-conflict
      - id: check-yaml
```

Every `rev` is a full commit SHA with the tag in a comment. A git tag can be
moved; a SHA cannot. Same reasoning as pinning GitHub Actions (chapter 13).

Consider adding `terraform_docs` so module READMEs cannot drift from
`variables.tf`.

## `terraform test`

Terraform has a native test framework. Tests live in `tests/*.tftest.hcl`:

```hcl
# tests/defaults.tftest.hcl
variables {
  secrets = {
    "test-secret" = { description = "A test secret" }
  }
}

run "creates_one_secret" {
  command = plan

  assert {
    condition     = length(aws_secretsmanager_secret.this) == 1
    error_message = "Expected exactly one secret"
  }
}

run "rejects_bad_name" {
  command = plan

  variables {
    secrets = { "bad name with spaces" = {} }
  }

  expect_failures = [var.secrets]
}
```

```bash
terraform test
```

`command = plan` runs without creating anything — fast and free, and enough to
test validation rules and computed values. `command = apply` creates real
resources and destroys them afterwards; use sparingly.

This is the natural next step for `arifa-infra`'s two modules: the validation
rules in `modules/sagemaker-notebook/variables.tf` are exactly what plan-level
tests are good at.

## Other tools

- **Checkov** — another misconfiguration scanner. One is usually enough; trivy
  also covers secrets.
- **Infracost** — estimates cost change from a plan, posted on the PR.
- **OPA / Conftest** — policy as code against plan JSON, for organisation rules
  no off-the-shelf scanner knows.
- **Terratest** — Go tests that really provision. Powerful, slow, expensive.
  `terraform test` covers most of what teams used it for.

## Try it

```bash
cd arifa-infra

bash scripts/check.sh fmt          # no tools beyond terraform
bash scripts/check.sh              # all four

# see the preflight when a tool is missing
PATH=/usr/bin bash scripts/check.sh
```

Then find the single `#trivy:ignore` in the repository and explain, from the
comment alone, why it is justified and what would remove the need for it.

## Check yourself

1. Why can all four gates run without AWS credentials?
2. What does `-backend=false` do for `validate`?
3. Why are the tflint plugin versions pinned in `.tflint.hcl`?
4. What does `call_module_type = "local"` change?
5. Why is an inline `#trivy:ignore` better than a global ignore file?
6. Why did `fmt && validate && lint && scan` hide failures?
7. What is the difference between `command = plan` and `command = apply` in a test?

---

Previous: [11 — Importing existing infrastructure](11-importing-existing-infrastructure.md) ·
Next: [13 — CI/CD](13-cicd-and-oidc.md)
