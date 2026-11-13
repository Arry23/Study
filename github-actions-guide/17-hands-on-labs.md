# 17 — Hands-on labs

These labs use the workflow in this workspace plus a scratch repository. Some
run entirely offline; the execution labs need a GitHub repository you control
(a free personal account is fine).

You have `python`, `node`, `git` and `podman`. Lab 1 installs the one tool you
are missing — and teaches the checksum pattern from chapter 05 while doing it.

---

## Lab 1 — Install actionlint the hard way

**Goal:** practise pinned, checksum-verified tool installation.
**Covers:** [ch. 05](05-actions.md), [ch. 09](09-security-hardening.md)

`actionlint` catches script injection, invalid contexts and shell bugs. Install
it the way `arifa-infra` installs tflint — version pinned, checksum verified.

1. Find the latest release at
   `https://github.com/rhysd/actionlint/releases`.
2. Download the Windows archive **and** the `checksums.txt` for that release.
3. Verify the hash before extracting:

```powershell
$expected = (Select-String -Path checksums.txt -Pattern 'windows_amd64').Line.Split()[0]
$actual   = (Get-FileHash .\actionlint_windows_amd64.zip -Algorithm SHA256).Hash.ToLower()
"expected: $expected"; "actual  : $actual"
if ($expected -eq $actual) { "OK" } else { "MISMATCH - STOP" }
```

4. Extract to `C:\tools\bin` (already on your PATH).

```bash
actionlint --version
actionlint .github/workflows/*.yml     # from arifa-infra
```

**Verify:** actionlint runs and reports clean on `arifa-infra`.

**Think about:** why did the workflow's author choose this over
`uses: some/setup-actionlint@v1`? Re-read the header comment about the March
2026 tag hijack.

---

## Lab 2 — Exploit a script injection

**Goal:** understand the vulnerability by exploiting it.
**Covers:** [ch. 09](09-security-hardening.md)

Create a scratch repository with this **deliberately vulnerable** workflow:

```yaml
# .github/workflows/vulnerable.yml
name: vulnerable
on: pull_request
permissions:
  contents: read
jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - run: echo "PR title: ${{ github.event.pull_request.title }}"
```

1. Run `actionlint` on it. What does it say?
2. Open a PR from a branch, with the title:

```
test"; echo PWNED; echo "
```

3. Read the job log.

**Verify:** `PWNED` appears in the output. You just executed arbitrary code
through a PR title.

4. Now fix it:

```yaml
      - run: echo "PR title: $TITLE"
        env:
          TITLE: ${{ github.event.pull_request.title }}
```

5. Re-run with the same malicious title.

**Verify:** the title is printed **literally**, not executed. Confirm
actionlint is now clean.

**Then list:** six other untrusted context values from chapter 09.

> Do this only in a scratch repository you own. Never test injection against a
> repository with real secrets.

---

## Lab 3 — Pin an action to a SHA

**Goal:** convert tag references to immutable ones.
**Covers:** [ch. 05](05-actions.md)

```bash
# Resolve a tag to its commit SHA without cloning
git ls-remote https://github.com/actions/checkout refs/tags/v4.2.2
git ls-remote https://github.com/actions/cache refs/tags/v4.2.0
```

1. Write the pinned `uses:` line for each, with the tag as a comment.
2. Explain why `git ls-remote` on an **annotated** tag may show a tag object
   rather than the commit, and how to get the commit
   (`refs/tags/v4.2.2^{}`).
3. Verify `arifa-infra`'s three pins are 40-character SHAs with version
   comments.

**Verify:** your pinned lines match the form used in
[`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml).

**Then:** write the `.github/dependabot.yml` that keeps those pins fresh. The
repository does not have one — this is a real gap from chapter 05.

---

## Lab 4 — Close the `main` gap

**Goal:** fix a genuine weakness in this workspace.
**Covers:** [ch. 03](03-triggers.md)

`arifa-infra`'s workflow triggers only on `pull_request`. Nothing validates
`main` after a merge.

1. Write the `on:` block adding `push: branches: [main]`.
2. Explain two ways code reaches `main` without a validated PR.
3. Would the existing `paths:` filter apply to the push trigger? Should it?
4. Add a nightly scheduled run for drift detection. What cron, and in which
   timezone?
5. Why do scheduled workflows only run on the default branch?

**Verify:** your `on:` block has three triggers and you can justify each.

**Then:** open it as a PR against `arifa-infra` — it is a real improvement.

---

## Lab 5 — Permissions archaeology

**Goal:** reason about least privilege.
**Covers:** [ch. 08](08-permissions-and-oidc.md)

Using [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. What can the `check` job's `GITHUB_TOKEN` do? List the scopes explicitly.
2. What can the `plan` job's do, and what extra scope does it have?
3. Why can only one job request an OIDC token?
4. What would break if you deleted the top-level `permissions:` block?
5. The workflow uses **no** `secrets.*`. How does it authenticate to AWS?
6. Find the `sub` condition in
   [`bootstrap/github_oidc.tf`](../arifa-infra/bootstrap/github_oidc.tf). What
   would `StringLike` with `repo:myorg/*` allow?

**Verify:** your answer to 4 mentions that unlisted scopes default to `none`
*only when a block is present*.

**Then:** write the `permissions:` block for a workflow that must comment on
PRs and publish a package — and nothing else.

---

## Lab 6 — Remove the duplication with a composite action

**Goal:** author a composite action.
**Covers:** [ch. 12](12-reusable-workflows.md)

Both jobs in `terraform.yml` repeat checkout → read version → setup Terraform.

Write `.github/actions/setup-terraform/action.yml`:

- `using: composite`
- Reads `.terraform-version` and exposes it as an **output**
- Calls `hashicorp/setup-terraform` pinned to a SHA
- Every `run` step has `shell: bash`

Then rewrite both jobs to use it.

**Verify:**

1. `actionlint` is clean
2. The checkout stays in the job (a composite action should not assume it)
3. You can explain why `shell:` is mandatory

**Then answer:** would a **reusable workflow** have been better here? What
would that enable that a composite action cannot?

---

## Lab 7 — Build a workflow from scratch

**Goal:** author a complete, hardened workflow.
**Covers:** everything

In a scratch repository, write `.github/workflows/ci.yml` for a Python project:

- Triggers: PRs touching `src/**`, pushes to `main`, manual dispatch with an
  `environment` choice input
- `permissions: contents: read` at the top
- `concurrency` cancelling superseded PR runs
- A `test` job with a matrix over Python 3.11/3.12/3.13, `fail-fast: false`
- Dependency caching via `setup-python`'s `cache: pip`
- Upload coverage as an artifact `if: always()`, named per matrix entry
- A job summary with the test result
- A `deploy` job needing `test`, gated on `github.ref == 'refs/heads/main'`,
  using an environment with required reviewers
- Every action SHA-pinned
- `timeout-minutes` on every job

**Verify:**

```bash
actionlint .github/workflows/ci.yml
```

then walk the twelve safety rules in [chapter 16](16-cheatsheet.md) and tick
each off.

---

## Lab 8 — Pass data between steps and jobs

**Goal:** master `$GITHUB_OUTPUT`, `$GITHUB_ENV`, `$GITHUB_PATH` and artifacts.
**Covers:** [ch. 04](04-jobs-and-steps.md), [ch. 11](11-artifacts-caching-summaries.md)

Write a workflow with:

1. A step that computes a version string and exposes it as a **step output**
2. A later step that consumes it
3. A step that adds a directory to `$GITHUB_PATH`, and a later step that runs a
   binary from it
4. A **job output** consumed by a second job via `needs`
5. A file written in job A, uploaded, and downloaded in job B
6. A `$GITHUB_STEP_SUMMARY` table showing all of the above

**Verify:** job B prints a value computed in job A **and** reads a file created
there. Then delete the `upload-artifact` step and confirm job B fails — proving
files do not cross jobs.

**Then answer:** when would you choose `$GITHUB_ENV` over `$GITHUB_OUTPUT`,
and why is it usually the worse choice?

---

## Lab 9 — The pipefail trap

**Goal:** see a failing pipeline report success.
**Covers:** [ch. 04](04-jobs-and-steps.md), [ch. 14](14-troubleshooting.md)

```yaml
jobs:
  trap:
    runs-on: ubuntu-latest
    steps:
      - name: Without pipefail
        run: |
          false | tee out.txt
          echo "step finished"

      - name: With pipefail
        run: |
          set -o pipefail
          false | tee out.txt
          echo "never reached"
```

**Verify:** the first step **passes** despite `false` failing; the second
fails.

**Then:** find where `arifa-infra` guards against exactly this:

```yaml
- name: terraform plan
  run: |
    set -o pipefail
    terraform -chdir="envs/$TF_ENV" plan -no-color | tee "$RUNNER_TEMP/plan.txt"
```

Without it, a failing `terraform plan` would be reported as a successful check.

---

## Lab 10 — Error catalogue

**Goal:** recognise failures from the log alone.
**Covers:** [ch. 14](14-troubleshooting.md)

Cause each error deliberately in a scratch repository and record the message:

| Break this | Expected |
|---|---|
| Omit `actions/checkout` then `cat README.md` | No such file or directory |
| `uses:` a nonexistent action | Unable to resolve action |
| Request `pull-requests: write` work with `contents: read` only | Resource not accessible by integration |
| OIDC step without `id-token: write` | Not authorized / credentials not loaded |
| `needs:` a job excluded by `if:` | Job skipped |
| Upload an artifact from a path that does not exist | No files were found |
| `runs-on: nonexistent-label` | Queued indefinitely |
| Expression `if: "false"` | Step runs anyway — truthy string |

**Verify:** for each, one sentence on how you would recognise it in CI.

---

## Lab 11 (optional) — Run workflows locally

**Goal:** fast iteration without pushing.
**Covers:** [ch. 14](14-troubleshooting.md)

`act` runs workflows in containers; you have Podman.

```bash
# install act (see https://github.com/nektos/act/releases — pin and verify)
export DOCKER_HOST="unix://$(podman info --format '{{.Host.RemoteSocket.Path}}')"
act pull_request -W .github/workflows/ci.yml --list
act pull_request -W .github/workflows/ci.yml -j test
```

> `act` approximates. It will not reproduce OIDC, self-hosted runner labels,
> `GITHUB_TOKEN` permissions, job summaries or environments with reviewers.
> Use it for **shell logic**, not for validating security behaviour.

**Verify:** one job runs locally and its exit code matches what GitHub would
report.

---

## Lab 12 — Audit and report

**Goal:** produce a review you would actually send.
**Covers:** [ch. 09](09-security-hardening.md)

Audit `arifa-infra`'s workflow against the chapter 09 checklist and write:

| Item | Status | Evidence | Action |
|---|---|---|---|

Cover all ten audit questions. Be fair — this workflow does most things right.
Your report should note the strengths (SHA pins, checksum verification,
`persist-credentials: false`, least-privilege permissions, env indirection) as
well as the two real gaps (no `push: main` trigger, no `dependabot.yml`).

**Verify:** every row cites a line, and your two gaps match chapters 03 and 05.

---

## Suggested order

| If you have | Do |
|---|---|
| 20 minutes | Lab 9 — the pipefail trap, in one job |
| 1 hour | Labs 1, 2, 9 |
| An afternoon | Labs 1–6 |
| A day | All, with lab 12 written up |

## Turning labs into contributions

Labs 3, 4 and 6 produce real improvements to `arifa-infra`:

- "add dependabot config for github-actions"
- "validate main after merge and add nightly drift plan"
- "extract shared setup into a composite action"

---

Previous: [16 — Cheat sheet](16-cheatsheet.md) ·
Back to the [index](README.md)
