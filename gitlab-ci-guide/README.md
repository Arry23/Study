# GitLab CI/CD, A to Z

A complete course, from "what is a pipeline" to a hardened production setup.

Every chapter uses real code from the pipelines in this workspace —
[`main/.gitlab-ci.yml`](../main/.gitlab-ci.yml) in particular, a ten-stage
pipeline that builds, tests, scans and packages a Python service. There are
around twenty more `.gitlab-ci.yml` files here to compare against.

Using real code means you also see real problems. This guide points out several
genuine issues in those pipelines, including a secret that is printed to the job
log. Learning to spot those is the point.

## How to use this

Read in order if you are new. Each chapter ends with **Try it** and
**Check yourself**.

| # | Chapter | What you learn |
|---|---------|----------------|
| 01 | [What GitLab CI/CD is](01-what-is-gitlab-cicd.md) | Pipelines, jobs, runners, and what actually executes your code |
| 02 | [Pipeline anatomy](02-pipeline-anatomy.md) | `stages`, `jobs`, execution order, the shape of the file |
| 03 | [Jobs and scripts](03-jobs-and-scripts.md) | `image`, `script`, `before_script`, `after_script`, exit codes |
| 04 | [Variables](04-variables.md) | Predefined, custom, masked, protected, and precedence |
| 05 | [Rules and workflow](05-rules-and-workflow.md) | Controlling when pipelines and jobs run |
| 06 | [Artifacts and cache](06-artifacts-and-cache.md) | Passing files between jobs; speeding builds up |
| 07 | [needs and the DAG](07-needs-and-dag.md) | Breaking out of stage order for faster pipelines |
| 08 | [Reuse: include, extends, anchors](08-reuse-include-extends.md) | Sharing pipeline code across projects |
| 09 | [Runners](09-runners.md) | Tags, executors, and why your job is stuck pending |
| 10 | [Secrets and security](10-secrets-and-security.md) | Masking, protected variables, and the leak in this repo |
| 11 | [Docker images and packages](11-docker-and-packages.md) | Kaniko, the registry, generic packages |
| 12 | [Testing and code quality](12-testing-and-quality.md) | Coverage, SonarQube, SAST, dependency scanning |
| 13 | [Troubleshooting](13-troubleshooting.md) | The failures you will actually hit |
| 14 | [GitLab vs GitHub Actions](14-gitlab-vs-github-actions.md) | Concept-by-concept translation |
| 15 | [Cheat sheet](15-cheatsheet.md) | One page to keep open |
| 16 | [Hands-on labs](16-hands-on-labs.md) | **11 exercises on the real pipelines in this workspace** |

## Suggested path

**Day 1 — fundamentals.** 01 to 04. You can read any `.gitlab-ci.yml` and say
what runs, where, and in what order.

**Day 2 — control.** 05 to 07. You can decide precisely when a job runs and
make a slow pipeline fast.

**Day 3 — scale.** 08 to 11. You can share pipeline code and handle secrets
and images properly.

**Day 4 — production.** 12 to 14. You understand the pipelines in this
workspace, including what is wrong with them.

## The one thing to understand early

A GitLab job is **a shell script running in a container on a machine you do not
control**. Everything else — stages, rules, artifacts — is scheduling and
plumbing around that one fact.

If you remember that, "why can't my job see that file?" answers itself: it is a
different container.

## Conventions

- YAML is whitespace-sensitive. Two spaces, never tabs.
- `# ...` inside a snippet means "omitted", not a real comment.
- **Danger** marks things that leak secrets or break production.

## Related guides in this workspace

- [Terraform, A to Z](../terraform-guide/README.md)
- [GitHub Actions, A to Z](../github-actions-guide/README.md)
