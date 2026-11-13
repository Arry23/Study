# 05 — Actions

An **action** is a packaged, reusable step. This is what distinguishes GitHub
Actions from GitLab CI, where you write shell for everything.

It is also where the supply-chain risk lives.

## Using one

```yaml
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
  with:
    persist-credentials: false
```

- `actions/checkout` — owner and repository
- `@3d3c42e...` — the version reference
- `with:` — inputs
- `# v7.0.1` — a human-readable comment; **not** functional

## Three kinds

**JavaScript** — runs directly on the runner. Fast, cross-platform.

**Docker** — runs in a container. Any language, Linux only, slower.

**Composite** — a bundle of other steps in YAML. No code. Chapter 12.

You rarely care which, except that Docker actions do not work on Windows or
macOS runners.

## Version references

```yaml
- uses: actions/checkout@v4                 # major tag — MUTABLE
- uses: actions/checkout@v4.2.2             # exact tag — still mutable
- uses: actions/checkout@3d3c42e5aac5...    # commit SHA — IMMUTABLE
- uses: actions/checkout@main               # branch — worst
```

> **Tags are mutable.** `v4` is a pointer the action's maintainer can move at
> any time. Anyone who compromises that repository can repoint it at malicious
> code, and every workflow using `@v4` runs it on the next trigger — with your
> `GITHUB_TOKEN`, your secrets and your cloud credentials.
>
> This is not theoretical. The `tj-actions/changed-files` compromise in March
> 2025 repointed tags to code that dumped runner memory — including secrets —
> into public build logs across thousands of repositories.
>
> **Pin to a full commit SHA.** A SHA cannot be moved.

`arifa-infra` pins all three of its actions:

```yaml
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
- uses: hashicorp/setup-terraform@dfe3c3f87815947d99a8997f908cb6525fc44e9e # v4.0.1
- uses: aws-actions/configure-aws-credentials@e1253824e5c10ff9df46874f81ed3ec929e19cfd # v6.3.0
```

with the policy stated in the header:

> Every action is pinned to a full commit SHA. The org requires it, and tags can
> be moved.

Keep the tag in a comment — a bare SHA is unreadable, and Dependabot uses the
comment to propose updates.

## When no action is trustworthy

Sometimes the right answer is not to use one:

```yaml
env:
  TFLINT_VERSION: "0.65.0"
  TFLINT_SHA256: "d4c545ed2e29d6338604b586f34f544f380f70f243aa72f7b484713835eaec61"

# ...
- run: |
    set -euo pipefail
    curl -fsSLo tflint.zip "https://github.com/terraform-linters/tflint/releases/download/v${TFLINT_VERSION}/tflint_linux_amd64.zip"
    echo "${TFLINT_SHA256}  tflint.zip" | sha256sum --check --strict
    unzip -q -o tflint.zip tflint -d "$bin"
```

Ten lines instead of one, and in exchange: an exact version, a verified
checksum, and no third-party code running with your credentials.

The header explains both halves of the decision:

> Only GitHub-owned and verified-creator actions run. tflint and trivy have no
> allowed action, so they are installed from pinned release archives with a
> committed sha256. trivy-action is not used on purpose: its tags were hijacked
> in March 2026 (GHSA-69fq-xp46-6x23).

**Allowlisting by publisher, then falling back to verified downloads** is a
sound policy. Organisations can enforce it in Settings → Actions.

## Inputs and outputs

```yaml
- uses: hashicorp/setup-terraform@<sha>
  with:
    terraform_version: ${{ steps.tf.outputs.version }}
    terraform_wrapper: false
```

`terraform_wrapper: false` is worth knowing: by default this action wraps the
`terraform` binary to capture stdout, which interferes with piping and exit
codes. Turning it off is usually right when you handle output yourself.

Reading an action's outputs:

```yaml
- uses: actions/some-action@<sha>
  id: thing
- run: echo "${{ steps.thing.outputs.result }}"
```

## Actions worth knowing

| Action | Purpose |
|--------|---------|
| `actions/checkout` | Clone the repository. **Required** |
| `actions/setup-node` / `setup-python` / `setup-java` | Install a toolchain |
| `actions/cache` | Cache dependencies |
| `actions/upload-artifact` / `download-artifact` | Move files between jobs |
| `actions/github-script` | Run JS against the API |
| `aws-actions/configure-aws-credentials` | AWS auth, including OIDC |
| `docker/build-push-action` | Build and push images |

## `actions/checkout` options

```yaml
- uses: actions/checkout@<sha>
  with:
    persist-credentials: false   # do not leave a token in .git/config
    fetch-depth: 0               # full history (default 1)
    ref: ${{ github.event.pull_request.head.sha }}
    submodules: recursive
    path: subdir
```

`persist-credentials: false` is a meaningful hardening step. By default
checkout writes an authentication token into `.git/config`, where any later
step — including a compromised dependency — can read it and push to your
repository.

`arifa-infra` sets it on **both** of its checkouts. Make it a habit; set
`fetch-depth: 0` only when you genuinely need history.

## Evaluating a third-party action

Before `uses:` on something unfamiliar:

1. **Who publishes it?** `actions/*` is GitHub. "Verified creator" is a badge.
2. **Is it maintained?** Recent commits, open issues addressed.
3. **How much does it do?** A hundred-line action is auditable; a framework is not.
4. **What permissions does it need?** Anything wanting `write` deserves scrutiny.
5. **Can you read the source?** Actually open the repository at that SHA.
6. **Could ten lines of shell replace it?** Often yes.

## Keeping pins fresh

Pinning creates a maintenance duty. Dependabot handles it:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

It opens PRs updating the SHA **and** the version comment. You review and merge
— an upgrade becomes a commit, which is the whole point.

> `arifa-infra` has no `dependabot.yml`. Its pins are dated
> "checked 2026-11-02" by hand. Adding Dependabot would automate it.

## Organisation controls

Settings → Actions → General:

- Disable Actions entirely
- Allow only actions created by GitHub
- Allow GitHub plus verified creators
- Allow a specific list

Combined with required SHA pinning, this is the strongest available control.

## Try it

In [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. How many `uses:` steps? Who publishes each?
2. Verify each pin is a 40-character SHA with a version comment.
3. Why is `persist-credentials: false` on both checkouts?
4. Why is `terraform_wrapper: false` set?
5. Why are tflint and trivy installed by hand?
6. Write the `dependabot.yml` this repository is missing.

## Check yourself

1. What are the three kinds of action?
2. Why is `@v4` unsafe when `@3d3c42e...` is not?
3. What does `persist-credentials: false` prevent?
4. When should you write shell instead of using an action?
5. What does Dependabot do for pinned actions?
6. Three things to check before using an unfamiliar action?

---

Previous: [04 — Jobs and steps](04-jobs-and-steps.md) ·
Next: [06 — Contexts and expressions](06-contexts-and-expressions.md)
