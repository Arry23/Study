# GitHub Actions, A to Z

A complete course, from "what is a workflow" to a hardened, OIDC-authenticated
pipeline.

Every chapter uses real code from
[`arifa-infra/.github/workflows/terraform.yml`](../arifa-infra/.github/workflows/terraform.yml)
in this workspace. That workflow is unusually well hardened — every action
pinned to a commit SHA, checksum-verified tool downloads, OIDC instead of stored
keys, least-privilege permissions, and a documented refusal to use a
compromised third-party action. It makes a good teacher because the reasoning
is written down in the file itself.

## How to use this

Read in order if you are new. Each chapter ends with **Try it** and
**Check yourself**.

| # | Chapter | What you learn |
|---|---------|----------------|
| 01 | [What GitHub Actions is](01-what-is-github-actions.md) | Workflows, jobs, steps, runners |
| 02 | [Workflow anatomy](02-workflow-anatomy.md) | The shape of a workflow file |
| 03 | [Triggers](03-triggers.md) | `on:`, events, path and branch filters |
| 04 | [Jobs and steps](04-jobs-and-steps.md) | `runs-on`, `run`, `uses`, shells, exit codes |
| 05 | [Actions](05-actions.md) | Using actions safely; why pinning matters |
| 06 | [Contexts and expressions](06-contexts-and-expressions.md) | `${{ }}`, `github.*`, functions, `if:` |
| 07 | [Variables and secrets](07-variables-and-secrets.md) | `env`, `vars`, `secrets`, masking |
| 08 | [Permissions and OIDC](08-permissions-and-oidc.md) | `GITHUB_TOKEN`, cloud auth without keys |
| 09 | [Security hardening](09-security-hardening.md) | Script injection, supply chain, forks |
| 10 | [Matrix and concurrency](10-matrix-and-concurrency.md) | Parallelism and cancellation |
| 11 | [Artifacts, caching, summaries](11-artifacts-caching-summaries.md) | Passing data and reporting |
| 12 | [Reusable workflows and composite actions](12-reusable-workflows.md) | Sharing pipeline code |
| 13 | [Runners](13-runners.md) | GitHub-hosted vs self-hosted |
| 14 | [Troubleshooting](14-troubleshooting.md) | The failures you will actually hit |
| 15 | [GitHub Actions vs GitLab CI](15-github-vs-gitlab.md) | Concept-by-concept translation |
| 16 | [Cheat sheet](16-cheatsheet.md) | One page to keep open |
| 17 | [Hands-on labs](17-hands-on-labs.md) | **12 exercises, including exploiting a real script injection** |

## Suggested path

**Day 1 — fundamentals.** 01 to 05. You can read any workflow and say what runs
and when.

**Day 2 — logic.** 06 to 08. You can write conditionals and authenticate to a
cloud without storing credentials.

**Day 3 — hardening and scale.** 09 to 12.

**Day 4 — operations.** 13 to 15.

## The one thing to understand early

A workflow step is either **a shell command** or **somebody else's code running
with your credentials**. The second is what makes Actions powerful and what
makes it a supply-chain target.

Chapter 05 and chapter 09 are the ones that matter most.

## Conventions

- YAML is whitespace-sensitive. Two spaces, never tabs.
- `<sha>` in an example means a real 40-character commit SHA.
- **Danger** marks things that leak secrets or allow code execution.

## Related guides in this workspace

- [GitLab CI/CD, A to Z](../gitlab-ci-guide/README.md)
- [Terraform, A to Z](../terraform-guide/README.md) — chapter 13 covers the AWS
  side of OIDC in depth
