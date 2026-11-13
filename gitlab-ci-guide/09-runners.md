# 09 — Runners

A **runner** is the agent that actually executes jobs. GitLab decides *what*
should run; a runner does it.

If your job sits in `pending` forever, this chapter is the answer.

## How a job finds a runner

1. GitLab creates the pipeline and queues the jobs.
2. Runners poll GitLab asking for work.
3. A runner takes a job only if it **matches**.

Matching means: the runner is available to the project, and it has **every tag
the job asks for**.

```yaml
semgrep-sast:
  tags:
    - linux-amd64-m
```

Only runners tagged `linux-amd64-m` may take this job. The tag here encodes
platform and size — Linux, amd64, medium.

> **The number one cause of a stuck pipeline is a tag with no matching runner.**
> A typo like `linux-amd64-M` queues forever with no error. Check
> Settings → CI/CD → Runners for the tags that actually exist.

## Runner scope

| Scope | Available to |
|-------|--------------|
| Instance | Every project on the GitLab instance |
| Group | Every project in a group |
| Project | One project |

Settings → CI/CD → Runners shows which apply, their tags, and whether they are
online.

## Executors

How a runner isolates a job:

| Executor | Behaviour |
|----------|-----------|
| **Docker** | Each job in a fresh container. The usual choice |
| Shell | Runs directly on the host. Fast, no isolation, state leaks |
| Kubernetes | Each job as a pod. Scales well |
| Docker Machine | Autoscaling VMs |
| SSH / VirtualBox / Parallels | Niche |

With the Docker executor, `image:` selects the container:

```yaml
image: python:3.11
```

With the Shell executor, `image:` is **ignored** — whatever is installed on the
host is what you get. That difference explains a lot of "works on the other
runner" confusion.

## Untagged jobs

A job with no `tags:` can only be picked up by runners configured to "run
untagged jobs". In many organisations that is disabled, so an untagged job hangs
forever.

In this workspace almost every job is tagged `linux-amd64-m`. Two are not:

```yaml
yapf_format:
  stage: format
  image: python:3.11
  # no tags

env-package:
  image: registry.code.roche.com/ocean/systemsquad/artifact-push-image:0.0.4
  stage: package
  # no tags
```

Those rely on an untagged-capable runner existing. That is an implicit
dependency on runner configuration nobody can see from the file — worth making
explicit by adding the tag.

## Choosing a tag

Tags usually encode capability:

```yaml
tags: [linux-amd64-m]        # platform + architecture + size
tags: [docker]               # executor
tags: [gpu]                  # hardware
tags: [production]           # network zone / permissions
```

Ask for the least you need. `tags: [linux-amd64-l]` on a job that lints YAML
occupies a large runner someone else needs.

## Self-hosted runners

Organisations run their own when they need:

- **Network access** — to internal registries and services behind a firewall
- **Compliance** — code must not leave a controlled environment
- **Hardware** — GPUs, lots of RAM
- **Speed** — warm caches, local mirrors

Hence the registry mirrors in the build job:

```yaml
DOCKER_MIRRORS: >
  --registry-mirror docker-cache.repository-ng.intranet.roche.com
  --registry-mirror mirror.gcr.io
```

An internal mirror first, Google's public mirror as a fallback. On a corporate
network the internal one is faster and may be the only one reachable.

## Concurrency

Runner-side settings determine parallelism:

```toml
# config.toml
concurrent = 4                 # jobs across all runners on this host

[[runners]]
  name = "linux-amd64-m"
  limit = 2                    # jobs for this runner
  [runners.docker]
    image = "alpine:latest"    # default when a job sets none
    privileged = false
```

If every job in a stage queues one at a time, `concurrent` is likely the
bottleneck, not GitLab.

## Why `privileged` matters

Building container images traditionally requires Docker-in-Docker, which needs a
privileged runner — and a privileged container can escape to the host. Many
organisations forbid it.

**Kaniko** builds images without a Docker daemon and without privileged mode,
which is why this pipeline uses it:

```yaml
image-build:
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - /kaniko/executor --context ${DOCKER_CONTEXT} ${DOCKER_MIRRORS} --destination ${CONTAINER_REGISTRY}:${BUILD_TAG} --single-snapshot
```

Chapter 11.

## Timeouts

Three layers, smallest wins:

1. Project timeout (Settings → CI/CD, default 60 minutes)
2. Runner timeout
3. Job timeout:

```yaml
job:
  timeout: 30 minutes
```

## Diagnosing a stuck job

Work down this list:

1. **Does a runner with those tags exist?** Settings → CI/CD → Runners.
2. **Is it online?** Offline runners show a grey icon.
3. **Is it assigned to this project?**
4. **Are all its slots busy?** Check other running pipelines.
5. **Protected branch?** A runner marked "protected" only takes protected-branch
   jobs.
6. **Untagged allowed?** If the job has no tags.

The job page states the reason: *"This job is stuck because you don't have any
active runners online with any of these tags."*

## Runner-related failures

**`no basic auth credentials`** — the runner cannot pull the image. Private
registry credentials missing.

**`ERROR: Job failed (system failure)`** — runner or container problem, not your
script. Worth a retry:

```yaml
retry:
  max: 2
  when: runner_system_failure
```

**Works on one runner, fails on another** — different executors or different
host state. The commonest cause of flakiness in a mixed fleet.

**`no space left on device`** — the runner's disk is full of old containers and
caches. An operations problem, not a pipeline one.

## Try it

In this workspace:

1. Which tag do nearly all jobs request?
2. Find the two jobs with no `tags:`. What are they relying on?
3. Why does `image-build` use Kaniko rather than `docker build`?
4. Why does it pass two registry mirrors?
5. Compare the tags used in [`main/.gitlab-ci.yml`](../main/.gitlab-ci.yml)
   and [`pdil/.gitlab-ci.yml`](../pdil/.gitlab-ci.yml) — consistent?

## Check yourself

1. How does a runner decide it can take a job?
2. What happens to a job whose tag matches no runner?
3. Why is `image:` ignored by the Shell executor?
4. Why does Kaniko avoid needing a privileged runner?
5. Three reasons to self-host runners?
6. Which `retry:when` value suits infrastructure flakiness?

---

Previous: [08 — Reuse: include, extends, anchors](08-reuse-include-extends.md) ·
Next: [10 — Secrets and security](10-secrets-and-security.md)
