# 16 — Troubleshooting

How to read Terraform errors, and the ones you will actually meet. Several of
these are taken verbatim from this workspace.

## Read the error properly

Terraform errors are unusually good. They give you the file, the line, and
usually the fix:

```
╷
│ Error: Unsupported Terraform Core version
│
│   on versions.tf line 3, in terraform:
│    3:   required_version = "~> 1.17.0"
│
│ This configuration does not support Terraform version 1.16.6.
╵
```

File, line, offending expression, explanation. Read all of it before searching
the web.

---

## Version and setup

### Unsupported Terraform Core version

```
Error: Unsupported Terraform Core version
This configuration does not support Terraform version 1.16.6.
```

**Cause.** Your Terraform is outside `required_version`. `~> 1.17.0` means
`>= 1.17.0, < 1.18.0`, so 1.16.6 is refused.

**Fix.** Install the version in `.terraform-version`. With tfenv:

```bash
cd arifa-infra && tfenv install && tfenv use 1.17.1
```

On Windows, install the exact version and put it first on `PATH` (chapter 02).

**Do not** widen the constraint to make the error go away. It exists so everyone
plans identically.

### No package cached in .terraform/providers

```
Error: registry.terraform.io/hashicorp/aws: there is no package for
registry.terraform.io/hashicorp/aws 6.71.0 cached in .terraform\providers
```

**Cause.** `init` has not run here, or it ran and failed.

**Fix.** `terraform init`. If it is a cascade from the version error above, fix
that first — a failed `init` installs nothing.

### Module not installed

```
Error: Module not installed
  on main.tf line 30:
  30: module "notebook" {
This module is not yet installed. Run "terraform init" to install all modules.
```

**Cause.** Same — `init` has not linked the modules.

**Fix.** `terraform init`, or `make init ENV=dev`.

### command not found

```
scripts/check.sh: line 33: tflint: command not found
make: *** [Makefile:24: check] Error 127
```

**Cause.** The tool is not installed or not on `PATH`. Exit 127 is the shell's
code for "command not found"; make reports it and exits 2 itself.

**Fix.** Install the pinned versions (chapter 02). `scripts/check.sh` now
reports this up front:

```
missing tool: tflint trivy
install the pinned versions listed under Setup in README.md
```

### Lock file would change

```
Error: Provider dependencies have changed
...but the lock file is readonly.
```

**Cause.** `-lockfile=readonly` and the lock would need updating — usually a
changed constraint, or a lock without hashes for your platform.

**Fix.** Deliberately update it and commit:

```bash
make lock      # regenerates for darwin_arm64 and linux_amd64
```

---

## State

### State lock

```
Error: Error acquiring the state lock
Lock Info:
  ID:        1a2b3c4d-...
  Operation: OperationTypeApply
  Who:       someone@host
  Created:   2026-11-03 10:15:00
```

**Cause.** Another run holds the lock, or a previous run was killed before
releasing it.

**Fix.** Check `Who` and `Created`. If a colleague is applying, wait. If it is
stale:

```bash
terraform force-unlock 1a2b3c4d-...
```

> Only when you are certain nothing is running. Breaking a live lock corrupts
> state.

### Resource already exists

```
Error: creating S3 Bucket: BucketAlreadyOwnedByYou
```

**Cause.** The resource exists in AWS but not in state — created by hand, or
state was lost.

**Fix.** Import it rather than deleting it (chapter 11):

```hcl
import {
  to = aws_s3_bucket.state
  id = "my-bucket-name"
}
```

### Resource already managed

```
Error: Resource already managed by Terraform
```

**Cause.** You are importing something already in state.

**Fix.** `terraform state list | grep <name>` to confirm, then delete the
`import` block.

### Terraform wants to recreate everything

**Cause.** Usually the wrong state — a backend misconfiguration, or a local
state file shadowing the remote one.

**Fix.** Stop. Do not apply.

```bash
terraform state list       # empty or short? wrong state.
cat backend.tf             # right bucket and key?
ls terraform.tfstate*      # a stray local file?
terraform init -reconfigure
```

---

## Plan and apply

### Forces replacement

```
-/+ resource "aws_sagemaker_notebook_instance" "this" {
      ~ subnet_id = null -> "subnet-0abc" # forces replacement
```

**Not an error** — a warning you must read. The resource is destroyed and
recreated. For a notebook, everything on its volume is lost.

**Fix.** Decide if it is acceptable. Back up first. For `arifa-infra`'s notebook
this is exactly why roadmap items 9 and 10 are planned rather than done.

### Value cannot be determined until apply

```
Error: Invalid for_each argument
The "for_each" value depends on resource attributes that cannot be determined
until apply.
```

**Cause.** `for_each` keys come from something not yet created.

**Fix.** Key by something static — a variable or a literal:

```hcl
# bad
for_each = { for b in aws_s3_bucket.many : b.arn => b }

# good
for_each = var.bucket_names
```

### Provider configuration not present

```
Error: Provider configuration not present
```

**Cause.** A module expects a provider the caller did not pass, usually an
alias.

**Fix.**

```hcl
module "secrets" {
  source    = "../../modules/secrets"
  providers = { aws = aws.imported }
}
```

### Cycle

```
Error: Cycle: aws_iam_role.a, aws_iam_policy.b
```

**Cause.** A depends on B and B depends on A.

**Fix.** Break it — usually by splitting an inline policy into a separate
attachment resource, or by referencing a name instead of an ARN.

```bash
terraform graph | grep -A5 cycle
```

### Account guard

```
Error: Incorrect AWS Account ID
```

**Cause.** `allowed_account_ids` does not match your credentials. Working as
designed.

**Fix.** Use the right profile:

```bash
export AWS_PROFILE=arifa
aws sts get-caller-identity    # confirm the account
```

---

## Credentials

### No valid credential sources

```
Error: No valid credential sources found for AWS Provider
```

**Fix.**

```bash
aws sso login --profile arifa
export AWS_PROFILE=arifa
aws sts get-caller-identity
```

If the CLI works and Terraform does not, you are probably in a shell where
`AWS_PROFILE` is not exported, or `make` set it for its own subprocess only.

### ExpiredToken

```
Error: ExpiredToken: The security token included in the request is expired
```

**Fix.** `aws sso login --profile arifa`. SSO sessions are typically 8–12 hours.

### Not authorized to perform sts:AssumeRoleWithWebIdentity

CI only.

**Cause.** The OIDC trust policy does not match the token's claims.

**Fix.** Check the `sub` condition against the real claim. The repo warns about
one organisation-specific form:

```hcl
# If AssumeRoleWithWebIdentity is denied although this looks right, read the real
# `sub` claim in CloudTrail: some organisations use the immutable-ID form
# repo:<owner>@<ownerId>/<repo>@<repoId>:pull_request.
```

Also check that the job has `id-token: write`.

---

## Quality gates

### fmt fails in CI but passes locally

Different Terraform versions format differently. Pin the version (chapter 02).

### tflint fails in CI but passes locally

Different tflint or ruleset versions. The rulesets are pinned in `.tflint.hcl`;
the tflint **binary** version is pinned only in the CI workflow. Match it
locally — 0.65.0 for this repo.

### make check passes but something is broken

This was a real bug here. `fmt && validate && lint && scan` suppressed `set -e`
inside the first three gates, and a function returns its *last* command's
status — so `validate` could return 0 after real failures.

Fixed by giving each gate its own status and looping instead of chaining
(chapter 12). If you inherit a script shaped like that, distrust its green.

---

## Debugging technique

**Turn on logging.**

```bash
TF_LOG=DEBUG TF_LOG_PATH=./tf.log terraform plan
TF_LOG=TRACE ...        # includes API requests and responses
```

> `TRACE` logs can contain secrets. Do not attach them to a ticket unredacted.

**Inspect state.**

```bash
terraform state list
terraform state show <address>
terraform show -json | jq '.values.root_module'
```

**Test an expression.**

```bash
terraform console
> var.secrets
> keys(var.secrets)
> can(regex("^[a-z-]+$", "my-name"))
```

**Narrow the scope** — for diagnosis only:

```bash
terraform plan -target=module.secrets
```

**Check for drift without proposing changes.**

```bash
terraform plan -refresh-only
```

**Start clean** when `.terraform` looks corrupt:

```bash
rm -rf .terraform .terraform.lock.hcl
terraform init
```

> Removing the lock file means re-resolving providers. Commit the regenerated
> one deliberately, and run `make lock` so other platforms keep working.

---

## Before you ask for help

1. Full error text, not a paraphrase
2. `terraform version`
3. Which directory you ran it in
4. What changed since it last worked
5. `terraform state list` output if it looks state-related

## Check yourself

1. What does exit code 127 mean, and what does make report?
2. Why is widening `required_version` the wrong fix?
3. When is `force-unlock` safe?
4. Terraform wants to recreate everything. What do you check first?
5. Why must `for_each` keys be known at plan time?
6. What is the risk of `TF_LOG=TRACE`?

---

Previous: [15 — Best practices](15-best-practices.md) ·
Next: [17 — Cheat sheet](17-cheatsheet.md)
