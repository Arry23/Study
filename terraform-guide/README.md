# Terraform, A to Z

A complete course, from "what is this" to running infrastructure in CI.

Every chapter uses real code from [`arifa-infra/`](../arifa-infra) in this
workspace rather than invented examples. When a chapter says *look at
`envs/dev/main.tf`*, open it — that is a working, reviewed Terraform project
and it illustrates nearly everything in this guide.

## How to use this

Read in order if you are new. Each chapter builds on the previous one.
If you already know the basics, the table below tells you where to jump.

Every chapter ends with **Try it** (something to run) and **Check yourself**
(questions you should be able to answer before moving on).

## Chapters

| # | Chapter | What you learn |
|---|---------|----------------|
| 01 | [What Terraform is](01-what-is-terraform.md) | Infrastructure as code, the plan/apply cycle, how Terraform differs from scripts |
| 02 | [Install and set up](02-install-and-setup.md) | Installing on Windows, macOS and Linux; version pinning; AWS credentials |
| 03 | [The HCL language](03-hcl-language.md) | Blocks, arguments, types, expressions, functions, loops |
| 04 | [Providers](04-providers.md) | What a provider is, version constraints, the lock file, provider aliases |
| 05 | [Resources and data sources](05-resources-and-data-sources.md) | Creating things, reading things, references, the dependency graph |
| 06 | [Variables, outputs and locals](06-variables-outputs-locals.md) | Inputs, validation, outputs, locals, sensitive values |
| 07 | [State and backends](07-state-and-backends.md) | What state is, why it matters, remote state, locking, drift |
| 08 | [Meta-arguments](08-meta-arguments.md) | `count`, `for_each`, `depends_on`, `lifecycle`, `provider` |
| 09 | [Modules](09-modules.md) | Writing, calling and versioning reusable components |
| 10 | [The workflow and the CLI](10-workflow-and-cli.md) | `init`, `plan`, `apply`, `destroy`, and reading a plan properly |
| 11 | [Importing existing infrastructure](11-importing-existing-infrastructure.md) | `import` blocks, config generation, `moved` and `removed` |
| 12 | [Quality gates and testing](12-quality-gates-and-testing.md) | `fmt`, `validate`, tflint, trivy, pre-commit, `terraform test` |
| 13 | [CI/CD](13-cicd-and-oidc.md) | Running Terraform in a pipeline, OIDC instead of access keys |
| 14 | [Security and secrets](14-security-and-secrets.md) | Keeping secrets out of code and state, least privilege |
| 15 | [Best practices](15-best-practices.md) | Repository layout, naming, and the anti-patterns to avoid |
| 16 | [Troubleshooting](16-troubleshooting.md) | The errors you will actually hit, and how to read them |
| 17 | [Cheat sheet](17-cheatsheet.md) | One page of commands and syntax to keep open |
| 18 | [Hands-on labs](18-hands-on-labs.md) | **12 exercises you can run offline, no AWS account needed** |

## Suggested path

**Day 1 — fundamentals.** Chapters 01 to 05. By the end you can read any
Terraform file and say what it does.

**Day 2 — structure.** Chapters 06 to 09. By the end you can write a module
and explain why state is the thing to be careful with.

**Day 3 — practice.** Chapters 10 to 12. By the end you can run the full
workflow and know what the quality gates are protecting you from.

**Day 4 — production.** Chapters 13 to 15. By the end you understand how
`arifa-infra` is wired and why it makes the choices it does.

Keep 16 and 17 open while you work.

## The one thing to understand early

Terraform keeps a **state file** that maps your code to real infrastructure.
Almost every confusing Terraform problem is a state problem. Chapter 07 is the
most important chapter in this guide; everything else is syntax.

## Conventions in this guide

- Commands are shown for `bash`. On Windows use Git Bash, which you already
  have at `C:\Program Files\Git\bin\bash.exe`.
- `# ...` in a code block means "omitted for brevity", not a real comment.
- Chapters mark things as **Danger** when getting them wrong can destroy
  infrastructure or leak secrets. Those are worth reading twice.
