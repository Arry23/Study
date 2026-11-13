# 11 — Docker images and packages

## Two roles for containers

**The job container** — where your script runs, chosen with `image:`.

**The built image** — an artefact your pipeline produces and pushes to a
registry.

Chapters 03 and 09 covered the first. This one covers the second.

## Why not `docker build`

The obvious approach needs a Docker daemon inside a container — Docker-in-Docker:

```yaml
build:
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t myimage .
```

That requires a **privileged** runner, and a privileged container can escape to
its host. Many organisations forbid it outright.

## Kaniko

Kaniko builds images from a Dockerfile with no daemon and no privileged mode:

```yaml
image-build:
  stage: image-build
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  variables:
    CONTAINER_REGISTRY: $CI_REGISTRY_IMAGE
    DOCKER_CONTEXT: ${CI_PROJECT_DIR}/installables/
    DOCKER_MIRRORS: >
      --registry-mirror docker-cache.repository-ng.intranet.roche.com
      --registry-mirror mirror.gcr.io
    BUILD_TAG: $CI_COMMIT_TAG
  script:
    - cd installables/
    - find ../code -type d -exec chmod 750 {} \;
    - find ../code -type f -exec chmod 640 {} \;
    - /kaniko/executor --context ${DOCKER_CONTEXT:-$CI_PROJECT_DIR} ${DOCKER_MIRRORS} --destination ${CONTAINER_REGISTRY}:${BUILD_TAG} --build-arg BASE_IMAGE=${BASE_IMAGE} --single-snapshot
  rules:
    - if: '$CI_COMMIT_TAG !~ "/^$/"'
```

Piece by piece:

| Part | Meaning |
|------|---------|
| `entrypoint: [""]` | Kaniko's image runs the executor directly; blank it so GitLab can run a shell |
| `--context` | Directory holding the Dockerfile and build context |
| `--destination` | Where to push — `registry/path:tag` |
| `--build-arg` | Passes `ARG` values into the Dockerfile |
| `--single-snapshot` | One filesystem snapshot at the end: faster, fewer layers |
| `--registry-mirror` | Pull base images through a mirror |

### The registry mirrors

```yaml
DOCKER_MIRRORS: >
  --registry-mirror docker-cache.repository-ng.intranet.roche.com
  --registry-mirror mirror.gcr.io
```

Two reasons: the internal mirror is faster and may be the only route out on a
corporate network, and it avoids Docker Hub rate limits. `>` folds the lines
into one argument string — correct here, since it is an argument list rather
than a script.

### Permissions before build

```yaml
- find ../code -type d -exec chmod 750 {} \;
- find ../code -type f -exec chmod 640 {} \;
```

Directories `rwxr-x---`, files `rw-r-----`. Permissions are baked into image
layers, so fixing them before the build is the only way to get them right —
afterwards would need another layer.

### Authentication

Kaniko needs registry credentials in `/kaniko/.docker/config.json`. Many setups
write it explicitly:

```yaml
before_script:
  - mkdir -p /kaniko/.docker
  - echo "{\"auths\":{\"${CI_REGISTRY}\":{\"username\":\"${CI_REGISTRY_USER}\",\"password\":\"${CI_REGISTRY_PASSWORD}\"}}}" > /kaniko/.docker/config.json
```

This pipeline does not, which means credentials come from the runner
configuration. That is cleaner — no credential handling in the pipeline file at
all.

## Tagging

```yaml
BUILD_TAG: $CI_COMMIT_TAG
rules:
  - if: '$CI_COMMIT_TAG !~ "/^$/"'
```

Images are built **only on tag pipelines**, and the image tag is the git tag.
Every image traces to exactly one commit, and no branch build pollutes the
registry.

Common alternatives:

```yaml
BUILD_TAG: $CI_COMMIT_SHORT_SHA        # every commit, immutable
BUILD_TAG: $CI_COMMIT_REF_SLUG         # per branch, mutable
BUILD_TAG: latest                      # avoid as the only tag
```

Prefer immutable tags. `latest` as the *only* tag makes "which code is running?"
unanswerable.

## The container registry

```yaml
CONTAINER_REGISTRY: $CI_REGISTRY_IMAGE
```

| Variable | Value |
|----------|-------|
| `CI_REGISTRY` | Registry host |
| `CI_REGISTRY_IMAGE` | This project's image path |
| `CI_REGISTRY_USER` / `CI_REGISTRY_PASSWORD` | Job-scoped credentials |

Pulling a private image in a later job:

```yaml
deploy:
  image: $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
```

Note `env-package` pulls from a shared internal registry:

```yaml
image: registry.code.roche.com/ocean/systemsquad/artifact-push-image:0.0.4
```

Pinned to `0.0.4` — exactly right, and a good contrast with the `:latest` images
elsewhere in the same file.

## The package registry

Images are not the only artefact. GitLab has a generic package registry:

```yaml
env-package:
  stage: package
  variables:
    ARTIFACT_PATH: ${CI_PROJECT_DIR}/configurations
  script: |
    set -x
    mkdir ${CI_PROJECT_DIR}/configurations
    cp ${CI_PROJECT_DIR}/code/backend/common_configurations/cli_env_constants.py ${CI_PROJECT_DIR}/configurations/
    cp ${CI_PROJECT_DIR}/code/backend/common_configurations/environment_params.json ${CI_PROJECT_DIR}/configurations/
    file_packager
    curl --header "JOB-TOKEN: $CI_JOB_TOKEN" --upload-file /tmp/"${CI_COMMIT_TAG}".zip "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/${CI_COMMIT_TAG}_env/${CI_COMMIT_TAG}_env/env.zip"
  rules:
    - if: '$CI_COMMIT_TAG !~ "/^$/"'
```

The generic package API:

```
PUT /projects/:id/packages/generic/:package_name/:package_version/:file_name
```

Authenticated with `CI_JOB_TOKEN` — no stored credentials.

Note `set -x` at the top of the script: it traces every command. Convenient, but
it prints expanded variables, so never combine it with secrets.

Artifacts vs packages: artifacts are pipeline-scoped and expire; packages are
versioned, durable, and meant for consumption by other projects.

## Writing a good Dockerfile for CI

```dockerfile
ARG BASE_IMAGE=python:3.11-slim
FROM ${BASE_IMAGE}

# System deps first — changes rarely, caches well
RUN apt-get update && apt-get install -y --no-install-recommends \
      libsasl2-dev libldap2-dev libssl-dev \
    && rm -rf /var/lib/apt/lists/*

# Dependencies before source — the key caching move
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Source last: it changes every commit
COPY code/ /app/code/

USER 1000
WORKDIR /app
CMD ["python", "-m", "myapp"]
```

**Order layers by how often they change.** Source last means a code-only change
reuses every dependency layer.

`ARG BASE_IMAGE` is what `--build-arg BASE_IMAGE=${BASE_IMAGE}` feeds, letting
the same Dockerfile build against different bases.

## Build a purpose-built CI image

Several jobs here do this every run:

```yaml
- apt-get update -y
- apt-get install -y libsasl2-dev python3-dev libldap2-dev libssl-dev
- pip install -r installables/requirements.txt
```

That is minutes per job, repeated across jobs and pipelines. Building one image
with those baked in, pushing it to the registry, and using it as `image:` turns
that into nothing. It also pins the toolchain, so a job cannot break because an
upstream package changed.

## Scan what you build

```yaml
scan-image:
  image:
    name: aquasec/trivy:0.76.1
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL "$CI_REGISTRY_IMAGE:$CI_COMMIT_TAG"
```

Building an image from a base with known CVEs is easy; the only way to know is
to scan. `arifa-infra` uses the same tool for Terraform misconfiguration.

## Try it

In [`main/.gitlab-ci.yml`](../main/.gitlab-ci.yml):

1. Why Kaniko instead of `docker build`?
2. Why `entrypoint: [""]`?
3. What does `--single-snapshot` trade away?
4. Why `chmod` before the build rather than after?
5. Under what condition do `image-build` and `env-package` run?
6. Which image in this file is properly pinned, and which are not?

## Check yourself

1. Why does Docker-in-Docker need a privileged runner?
2. What does `--context` tell Kaniko?
3. Why mirror the registry?
4. Why is `latest` a poor sole tag?
5. In a Dockerfile, why copy `requirements.txt` before the source?
6. Artifact vs package — when each?

---

Previous: [10 — Secrets and security](10-secrets-and-security.md) ·
Next: [12 — Testing and code quality](12-testing-and-quality.md)
