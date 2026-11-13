# 13 — Runners

A **runner** is the machine that executes a job. Which one you get is decided by
`runs-on:`.

## GitHub-hosted

```yaml
runs-on: ubuntu-latest
runs-on: ubuntu-24.04
runs-on: windows-latest
runs-on: macos-latest
```

A fresh VM per job, destroyed afterwards. Pre-loaded with common toolchains —
Git, Docker, Python, Node, the AWS CLI, and much else.

| | |
|---|---|
| **Good** | Zero maintenance, clean every time, broad tooling |
| **Less good** | No access to private networks, limited resources, costs minutes on private repos |

> `-latest` moves. When GitHub promotes `ubuntu-26.04` to `latest`, your
> workflow changes under you — the same mutable-reference problem as `@v4` or
> `image: tool:latest`. Pin to `ubuntu-24.04` for reproducibility, and upgrade
> deliberately.

## Self-hosted

```yaml
runs-on: rcn-ubuntu-amd64
runs-on: [self-hosted, linux, x64, gpu]
```

A machine you provide, running the runner agent. Matching requires **all**
listed labels.

Reasons to self-host:

- **Private network access** — internal registries, databases, services behind a
  firewall
- **Compliance** — code must not leave a controlled environment
- **Hardware** — GPUs, lots of memory
- **Cost** — heavy usage on private repositories
- **Speed** — warm caches, local mirrors

`arifa-infra` uses `rcn-ubuntu-amd64`, a Roche runner, because hosted runners
are disabled by policy. The workflow header records the consequence:

> Hosted runners are disabled. rcn-ubuntu-amd64 is the Roche runner label. It
> has no make, so CI runs scripts/check.sh and the terraform commands behind the
> make targets.

That is worth dwelling on. A self-hosted runner has **whatever its
administrator installed** — no more. Here, no `make`. So the workflow calls the
script directly:

```yaml
- name: Quality gates (scripts/check.sh)
  run: scripts/check.sh
```

and writes out the Makefile targets longhand:

```yaml
# The next two steps are `make init` and `make plan`, written out.
- name: terraform init
  env:
    TF_ENV: ${{ matrix.env }}
  run: terraform -chdir="envs/$TF_ENV" init -lockfile=readonly
```

This is also why the build logic lives in `scripts/check.sh` rather than the
Makefile: the script is the contract both a laptop and a make-less runner can
honour.

**Never assume a tool exists on a self-hosted runner.** Install it, pinned and
checksum-verified, or fail early with a clear message.

## Security of self-hosted runners

> **Never attach a self-hosted runner to a public repository.** Anyone can open
> a pull request, and by default that executes their code on your machine,
> inside your network.

Even privately:

- **State persists** between jobs unless the runner is ephemeral. Files,
  processes, poisoned caches and credentials can leak from one job to the next.
- **Prefer ephemeral runners** (`--ephemeral`), which deregister after one job.
- **Container or VM isolation** per job is better still.
- **Least privilege on the host** — the runner should not have cloud admin
  credentials sitting in its profile.

## Containers

Run the job's steps inside a container:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: python:3.12
      options: --cpus 2
    steps:
      - uses: actions/checkout@<sha>
      - run: python --version        # 3.12, from the container
```

> This is the default in GitLab (`image:`) but opt-in here. The more common
> GitHub idiom is a setup action on the host:
>
> ```yaml
> - uses: actions/setup-python@<sha>
>   with:
>     python-version: "3.12"
> ```
>
> Setup actions are faster and cache better. Reach for `container:` when you
> need an exact environment that is hard to reproduce with setup actions.

## Service containers

Sidecars for integration tests:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-retries 5
    steps:
      - run: pytest
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/postgres
```

The health check matters — without it the test can start before the database is
accepting connections.

## Groups and labels

Organisations can group runners and restrict which repositories may use them:

```yaml
runs-on:
  group: production-runners
  labels: [self-hosted, linux]
```

Use this to keep production-network runners away from general repositories.

## Scaling

- **Actions Runner Controller (ARC)** — runs ephemeral runners as Kubernetes
  pods, scaling on queue depth. The usual answer at scale.
- **Autoscaling VMs** — cloud instances created per job.
- **Large hosted runners** — GitHub's paid bigger machines, no maintenance.

ARC plus ephemeral pods gives hosted-like isolation with self-hosted network
access.

## Diagnosing a stuck job

Queued forever usually means no runner matches:

1. **Do all the labels exist?** `runs-on: [self-hosted, linux, x64]` needs a
   runner with all three.
2. **Is a runner online?** Settings → Actions → Runners.
3. **Is it allowed for this repository?** Check runner group permissions.
4. **Are all its slots busy?**
5. **Typo in a label?** Silent — nothing errors, it just never matches.

Same failure mode as a GitLab tag with no matching runner.

## Choosing

```
Public repository?                        -> GitHub-hosted, always
Needs private network access?             -> self-hosted
Compliance restricts where code runs?     -> self-hosted
Special hardware?                         -> self-hosted
Otherwise                                 -> GitHub-hosted
```

## Checklist

- [ ] `runs-on` pinned to a version, not `-latest`, where reproducibility matters
- [ ] Self-hosted runners never on public repositories
- [ ] Self-hosted runners ephemeral or container-isolated
- [ ] No assumption that a tool exists — install it pinned
- [ ] Runner groups restrict sensitive runners
- [ ] `timeout-minutes` set on every job

## Try it

In [`terraform.yml`](../arifa-infra/.github/workflows/terraform.yml):

1. Which runner label do both jobs use?
2. Why does the workflow call `scripts/check.sh` rather than `make check`?
3. Which two Makefile targets are written out longhand, and why?
4. The workflow installs tflint and trivy itself. What does that say about the
   runner?
5. What would break if someone changed `runs-on` to `ubuntu-latest`?

## Check yourself

1. Three reasons to self-host?
2. Why is `ubuntu-latest` a mutable reference?
3. Why must self-hosted runners never serve public repositories?
4. What does an ephemeral runner protect against?
5. `container:` vs a setup action — when each?
6. Why does a service container need a health check?

---

Previous: [12 — Reusable workflows and composite actions](12-reusable-workflows.md) ·
Next: [14 — Troubleshooting](14-troubleshooting.md)
