# 16 — Hands-on labs

You cannot run a GitLab pipeline on a laptop without a runner, so these labs
split into three kinds:

- **Audit labs** — work on the real pipelines in this workspace. No execution
  needed; these are the highest-value ones.
- **Authoring labs** — write YAML and validate it offline.
- **Execution labs** — optional, using `gitlab-ci-local` with Podman.

Everything here uses tooling you already have: `python`, `node`, `git`,
`podman`.

---

## Lab 1 — Find the credential leak

**Goal:** spot a real security bug in production CI.
**Covers:** [ch. 10](10-secrets-and-security.md)

Open [`main/.gitlab-ci.yml`](../main/.gitlab-ci.yml) and find the `submodule`
job's `before_script`.

1. List every line that writes a credential to the job log.
2. For each, say what an attacker with repository read access gains.
3. Write the replacement lines that prove a variable is set **without**
   revealing it.
4. Explain why GitLab's "masked" checkbox is not a sufficient defence.

**Verify:** you found **three** credential-printing lines, and your
replacements use `${#VAR}` or a `-n` test.

**Then answer:** a token *was* printed to a log six months ago. What is the
first action, and why is deleting the line insufficient?

---

## Lab 2 — Flatten the pipeline

**Goal:** turn a ten-stage chain into a DAG.
**Covers:** [ch. 07](07-needs-and-dag.md)

The pipeline runs ten stages sequentially. Several jobs wait for work they do
not depend on.

1. For each job, determine what it **genuinely** needs. Hint: grep for which
   jobs `cp -r ocean-cdr-coralreef-common/...`.
2. Draw the dependency graph on paper.
3. Write the `needs:` block for every job.
4. Identify the **critical path** — the longest chain.
5. Which single `needs: []` saves the most wall-clock time?

**Verify:** your graph has at least three jobs starting immediately
(`needs: []`), and you can state the critical path in one sentence.

**Then:** estimate the time saved, assuming
submodule 30 s, format 120 s, trufflehog 60 s, semgrep 90 s, pytest 300 s,
sonar 120 s.

---

## Lab 3 — Pin every mutable reference

**Goal:** eliminate supply-chain drift.
**Covers:** [ch. 08](08-reuse-include-extends.md), [ch. 10](10-secrets-and-security.md)

Find every reference in `main/.gitlab-ci.yml` that can change without a commit
in the repository.

1. List them — there are at least four.
2. For each, write the pinned version.
3. One `include:` has **no `file:`**. What does it pull in, and why is that a
   problem?
4. Semgrep uses `--config=auto`. What are the two consequences?

**Verify:** your list includes two `:latest` images, a `ref:`, and the
fileless include.

**Then answer:** the pipeline passed yesterday and fails today with no commits.
Which of these would you suspect first, and how would you confirm it?

---

## Lab 4 — Remove the duplication

**Goal:** apply `extends` and `default` to a real file.
**Covers:** [ch. 08](08-reuse-include-extends.md)

Count the repetition first:

```bash
cd fdd_1.19
python -c "import yaml,collections; d=yaml.safe_load(open('main/.gitlab-ci.yml')); jobs={k:v for k,v in d.items() if isinstance(v,dict) and 'script' in v}; print('jobs:',len(jobs)); print('image python:3.11:',sum(1 for v in jobs.values() if v.get('image')=='python:3.11')); print('tags linux-amd64-m:',sum(1 for v in jobs.values() if v.get('tags')==['linux-amd64-m']))"
```

Now refactor on paper:

1. Write a `default:` block for what nearly every job shares.
2. Write a `.with-common` hidden job for the three repeated `cp -r` lines.
3. Rewrite `pytest-coverage` using `extends`.
4. How many lines does the file lose?

**Verify:** your `.with-common` is used by at least three jobs, and no job
repeats the `cp -r` block.

**Think about:** why `extends` rather than a YAML anchor here?

---

## Lab 5 — Write rules from requirements

**Goal:** fluency with `rules:`.
**Covers:** [ch. 05](05-rules-and-workflow.md)

Write the `rules:` block for each requirement. No peeking.

1. Run only on merge requests.
2. Run only on pushes to `develop`, never on MRs.
3. Run only on semantic version tags (`v1.2.3`), not `v1.2.3-rc1`.
4. Run on MRs, but skip when the title starts with `Draft:`.
5. Manual job on `main` only, and it must **block** the pipeline until clicked.
6. Run only when files under `src/` changed, on MRs only.
7. Run only when a `.gitmodules` file exists.
8. Never run on scheduled pipelines; otherwise always run.

**Verify:** check 7 against the real `submodule` job, and 2 against
`sonar-report`. Yours should match.

**Then:** rewrite `'$CI_COMMIT_TAG !~ "/^$/"'` in a form a new teammate would
understand immediately.

---

## Lab 6 — Fix the artifact that disappears

**Goal:** understand artifact `when`.
**Covers:** [ch. 06](06-artifacts-and-cache.md)

The `yapf_format` job writes a diff file, fails on purpose when formatting is
wrong, and declares the diff as an artifact.

1. Under what condition is that artifact actually kept?
2. Why is that exactly backwards for its purpose?
3. Write the one-line fix.
4. The job is also `allow_failure: true`. What does that do to the `exit 1`?
5. Propose a simpler implementation that avoids the reformat-then-diff dance
   entirely.

**Verify:** your fix is `when: always`, and your simpler version uses
`yapf --diff`.

---

## Lab 7 — Validate YAML offline

**Goal:** catch errors before pushing.
**Covers:** [ch. 02](02-pipeline-anatomy.md), [ch. 13](13-troubleshooting.md)

```bash
cd fdd_1.19
python -c "import yaml; yaml.safe_load(open('main/.gitlab-ci.yml')); print('YAML OK')"
```

Now break it deliberately, one at a time, and record the error:

| Break | Error message |
|---|---|
| Tab instead of two spaces | |
| Duplicate job name | |
| Unquoted value starting with `*` | |
| `scripts:` instead of `script:` | |
| Missing `-` on a list item | |

**Verify:** note which of these a YAML parser catches and which it does
**not**. The duplicate key and the `scripts:` typo are the interesting ones —
both parse fine and both break the pipeline.

**Then:** write a tiny validator that checks every job has a `script` key:

```bash
python -c "
import yaml
d = yaml.safe_load(open('main/.gitlab-ci.yml'))
reserved = {'stages','include','variables','workflow','default','image','services','before_script','after_script','cache'}
for k, v in d.items():
    if k in reserved or k.startswith('.'): continue
    if isinstance(v, dict) and 'script' not in v and 'extends' not in v and 'trigger' not in v:
        print('no script:', k)
print('done')
"
```

---

## Lab 8 — Compare two pipelines

**Goal:** see what should be shared.
**Covers:** [ch. 08](08-reuse-include-extends.md)

```bash
cd fdd_1.19
python -c "
import yaml
a = yaml.safe_load(open('main/.gitlab-ci.yml'))
b = yaml.safe_load(open('common/.gitlab-ci.yml'))
ja = {k for k,v in a.items() if isinstance(v,dict) and ('script' in v or 'extends' in v)}
jb = {k for k,v in b.items() if isinstance(v,dict) and ('script' in v or 'extends' in v)}
print('only in main :', sorted(ja-jb))
print('only in common:', sorted(jb-ja))
print('in both      :', sorted(ja&jb))
"
```

1. Which jobs exist in both, and are they identical?
2. There are ~20 `.gitlab-ci.yml` files here. What belongs in a shared
   template project?
3. Design the template repository layout.
4. How would consumers pin it, and why does that matter?

**Verify:** your design has tagged releases and hidden jobs rather than
injected ones.

---

## Lab 9 — Security review

**Goal:** produce a written audit.
**Covers:** [ch. 10](10-secrets-and-security.md), [ch. 12](12-testing-and-quality.md)

Work through the checklist in chapter 10 against `main/.gitlab-ci.yml` and
write up findings with severity:

| Finding | Severity | Fix |
|---|---|---|
| | | |

Cover at minimum: credential printing, unpinned images, unpinned includes,
`allow_failure` on security-relevant jobs, the TruffleHog branch variable being
empty outside MR pipelines, and the token embedded in a clone URL in
`sonar-report`.

**Verify:** at least six findings, each with a file reference and a concrete
fix.

**Then:** rank them. Which would you fix this week, and which is a backlog
item?

---

## Lab 10 — Build a pipeline from scratch

**Goal:** author a complete, correct pipeline.
**Covers:** everything

In a scratch directory, write `.gitlab-ci.yml` for a Python service with:

- Stages: `lint`, `test`, `build`, `deploy`
- A `default:` block with a pinned image and tags
- `workflow:rules` preventing duplicate pipelines
- A lint job using `ruff`, with a cache keyed on the lockfile
- A test job producing a JUnit report **and** coverage, kept `when: always`
- A build job that runs **only on tags**, using Kaniko
- A deploy job that is **manual**, on `main` only, and blocks the pipeline
- `needs:` so lint and test run in parallel immediately
- No hardcoded secrets, no `:latest`

**Verify:**

```bash
python -c "import yaml; yaml.safe_load(open('.gitlab-ci.yml')); print('OK')"
```

then check it against the chapter 15 safety rules, one by one.

---

## Lab 11 (optional) — Run a pipeline locally

**Goal:** execute jobs without GitLab.
**Covers:** [ch. 03](03-jobs-and-scripts.md), [ch. 09](09-runners.md)

You have Podman and Node, so `gitlab-ci-local` works:

```bash
npm install -g gitlab-ci-local
export DOCKER_HOST="unix://$(podman info --format '{{.Host.RemoteSocket.Path}}')"
cd <your scratch pipeline from lab 10>
gitlab-ci-local --list
gitlab-ci-local lint-job
```

> It will not reproduce everything — no GitLab API, no `CI_JOB_TOKEN`, no
> runner tags, no `include:` from private projects. Treat it as a fast way to
> iterate on **script logic**, not a substitute for the real pipeline.

**Verify:** one job runs to completion in a container and its exit code
determines success.

**Then:** deliberately make a middle command in a `|` block fail and confirm
the job still passes — then add `set -euo pipefail` and confirm it fails.
That is chapter 03's most important point, demonstrated.

---

## Suggested order

| If you have | Do |
|---|---|
| 20 minutes | Lab 1 — it is a real security bug |
| 1 hour | Labs 1, 2, 6 |
| An afternoon | Labs 1–6 |
| A day | All, with lab 9 written up as a document you would send |

## Turning labs into contributions

Labs 1, 2, 3 and 6 identify genuine issues in a shared repository. Each is a
small, reviewable PR:

- "remove credential echo from submodule job"
- "add needs: to parallelise independent jobs"
- "pin scanner images and shared pipeline ref"
- "keep yapf diff artifact on failure"

That is the best possible outcome of a lab.

---

Previous: [15 — Cheat sheet](15-cheatsheet.md) ·
Back to the [index](README.md)
