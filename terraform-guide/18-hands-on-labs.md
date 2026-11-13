# 18 — Hands-on labs

Reading Terraform teaches you syntax. Breaking it teaches you Terraform.

Every lab below runs **offline with no AWS account and no cost**, using the
`random`, `local` and `null` providers. You get real `init`, `plan`, `apply`,
state files, drift and refactoring — the whole workflow — against resources
that live on your laptop.

## Before you start

You already have what you need:

```bash
terraform -version   # v1.17.1
tflint --version     # 0.65.0
trivy --version      # 0.76.1
```

Create a scratch directory **outside** any git repository:

```bash
mkdir -p ~/tf-labs && cd ~/tf-labs
```

> Nothing here touches `arifa-infra` unless a lab says so explicitly, and those
> labs are read-only or clearly marked.

---

## Lab 1 — Make `make check` green

**Goal:** fix the real blocker in this workspace.
**Covers:** [ch. 04](04-providers.md), [ch. 12](12-quality-gates-and-testing.md)

`make check` currently fails because the lock files have hashes for macOS and
Linux only, and you are on Windows.

1. Open [`arifa-infra/Makefile`](../arifa-infra/Makefile) and find `PLATFORMS`.
2. Add `-platform=windows_amd64`.
3. Run `make lock`.
4. Run `make check`.

**Verify:** `git diff` shows new `h1:`/`zh:` hash lines in both
`bootstrap/.terraform.lock.hcl` and `envs/dev/.terraform.lock.hcl`, and
`validate` now passes.

**Think about:** why does adding a platform *add* hashes rather than replace
them? What would break for your colleagues if you removed the darwin entry?

> This is a genuine change to a shared repo. Make it on a branch and open a PR
> — "add windows_amd64 provider hashes so Windows contributors can run the
> gates".

---

## Lab 2 — The full workflow, offline

**Goal:** run init → plan → apply → plan → destroy and watch state change.
**Covers:** [ch. 07](07-state-and-backends.md), [ch. 10](10-workflow-and-cli.md)

```hcl
# ~/tf-labs/lab2/main.tf
terraform {
  required_version = "~> 1.17.0"
  required_providers {
    random = { source = "hashicorp/random", version = "~> 3.6" }
    local  = { source = "hashicorp/local",  version = "~> 2.5" }
  }
}

resource "random_pet" "name" {
  length    = 2
  separator = "-"
}

resource "local_file" "greeting" {
  filename = "${path.module}/output/hello.txt"
  content  = "Hello from ${random_pet.name.id}\n"
}

output "pet_name" {
  value = random_pet.name.id
}
```

```bash
terraform init
terraform plan        # 2 to add
terraform apply
cat output/hello.txt
terraform plan        # No changes.
```

Now do the interesting part:

```bash
terraform state list
terraform state show random_pet.name
cat terraform.tfstate | python -m json.tool | head -40
```

**Verify:** you can point at the exact line in state that links
`local_file.greeting` to a real file on disk.

**Then break it:**

```bash
rm output/hello.txt
terraform plan        # what does Terraform say, and why?
terraform apply       # it comes back
```

That is **drift detection**. You removed the real resource; state still
believed it existed.

**Finally:**

```bash
rm terraform.tfstate*
terraform plan        # now what? why does it want to create BOTH again?
```

That is why chapter 07 says state is the most important thing in Terraform.

Clean up: `terraform destroy` (after restoring state) or just delete the
directory.

---

## Lab 3 — `count` vs `for_each`, the destructive difference

**Goal:** see the index-shifting problem with your own eyes.
**Covers:** [ch. 08](08-meta-arguments.md)

```hcl
# ~/tf-labs/lab3/main.tf
variable "names" {
  type    = list(string)
  default = ["alice", "bob", "carol"]
}

resource "local_file" "with_count" {
  count    = length(var.names)
  filename = "${path.module}/count/${var.names[count.index]}.txt"
  content  = "user: ${var.names[count.index]}\n"
}

resource "local_file" "with_foreach" {
  for_each = toset(var.names)
  filename = "${path.module}/foreach/${each.key}.txt"
  content  = "user: ${each.key}\n"
}
```

```bash
terraform init && terraform apply
terraform state list
```

Now **remove `"bob"`** from the default list and run `terraform plan`.

**Verify:** read the plan carefully and answer:

- How many `with_count` resources change, and which?
- How many `with_foreach` resources change, and which?
- Why does removing the *middle* element affect `carol` under `count`?

**Expected:** `count` destroys and recreates `carol` (index 2 → index 1) as
well as destroying `bob`. `for_each` touches only `bob`.

On real infrastructure that difference is an outage. This lab is the single
best five minutes you can spend on chapter 08.

---

## Lab 4 — `moved` blocks: refactor without destroying

**Goal:** migrate `count` → `for_each` with zero destruction.
**Covers:** [ch. 11](11-importing-existing-infrastructure.md)

Continue in lab 3. Delete the `with_count` resource and replace it:

```hcl
resource "local_file" "renamed" {
  for_each = toset(var.names)
  filename = "${path.module}/count/${each.key}.txt"
  content  = "user: ${each.key}\n"
}
```

Run `terraform plan`. It wants to **destroy 2 and create 2** — Terraform sees
old addresses vanish and new ones appear.

Now add `moved` blocks:

```hcl
moved {
  from = local_file.with_count[0]
  to   = local_file.renamed["alice"]
}

moved {
  from = local_file.with_count[1]
  to   = local_file.renamed["carol"]
}
```

**Verify:** `terraform plan` now says **No changes**. Apply, confirm, then
delete the `moved` blocks.

**Think about:** why is a `moved` block better than `terraform state mv`?

---

## Lab 5 — Write a module with validation

**Goal:** build a module interface that rejects bad input clearly.
**Covers:** [ch. 06](06-variables-outputs-locals.md), [ch. 09](09-modules.md)

```
~/tf-labs/lab5/
├── main.tf
└── modules/notes/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── versions.tf
```

Requirements for `modules/notes`:

- Input `notes` — `map(object({ body = string, priority = optional(number) }))`
- Validation: every key matches `^[a-z0-9-]{1,32}$`
- Validation: `priority`, when set, is between 1 and 5
- Creates one `local_file` per note
- Output `paths` — a map of note name → file path
- `versions.tf` states a **minimum** version, not a pin

Call it from the root with three notes, one of which is deliberately invalid.

**Verify:**

1. The invalid key produces **your** error message, not a provider error
2. `terraform plan` shows exactly the right number of files
3. `terraform -chdir=modules/notes init -backend=false && validate` passes
   standalone
4. `tflint --recursive` is clean

**Then:** add a fourth note and confirm only one file is added — proving your
`for_each` keys are stable.

---

## Lab 6 — `terraform test`

**Goal:** write real tests for lab 5's module.
**Covers:** [ch. 12](12-quality-gates-and-testing.md)

```hcl
# ~/tf-labs/lab5/modules/notes/tests/defaults.tftest.hcl
variables {
  notes = {
    "first-note" = { body = "hello" }
  }
}

run "creates_one_file" {
  command = plan

  assert {
    condition     = length(local_file.this) == 1
    error_message = "Expected exactly one note file"
  }
}

run "rejects_bad_key" {
  command = plan

  variables {
    notes = { "Invalid Key!" = { body = "x" } }
  }

  expect_failures = [var.notes]
}

run "rejects_bad_priority" {
  command = plan

  variables {
    notes = { "ok-key" = { body = "x", priority = 9 } }
  }

  expect_failures = [var.notes]
}
```

```bash
cd modules/notes && terraform test
```

**Verify:** three passing tests. Now **break a validation rule** on purpose and
confirm the matching test fails.

**Think about:** why does `command = plan` suffice here? When would you need
`command = apply`?

---

## Lab 7 — Import an existing resource

**Goal:** practise the import workflow without touching AWS.
**Covers:** [ch. 11](11-importing-existing-infrastructure.md)

Create a file **outside** Terraform's knowledge:

```bash
mkdir -p ~/tf-labs/lab7 && cd ~/tf-labs/lab7
mkdir -p data
printf 'pre-existing content\n' > data/legacy.txt
```

Now write config and an import block:

```hcl
resource "local_file" "legacy" {
  filename = "${path.module}/data/legacy.txt"
  content  = "pre-existing content\n"
}

import {
  to = local_file.legacy
  id = "./data/legacy.txt"
}
```

```bash
terraform init
terraform plan
```

**Verify:** the plan says **1 to import, 0 to add, 0 to change, 0 to destroy**.
That is a correct import — it changes nothing.

Now **deliberately get it wrong**: change `content` to something else and plan
again. You should see `1 to import, 1 to change`. That is the signal chapter 11
warns about — *your code does not match reality, fix the code*.

Restore the content, apply, then run `terraform plan` again and confirm
`No changes`. Delete the import block.

**Then try:** `terraform plan -generate-config-out=generated.tf` with the
resource block removed. Read what Terraform writes for you.

---

## Lab 8 — Read the errors

**Goal:** build pattern recognition for failures.
**Covers:** [ch. 16](16-troubleshooting.md)

In a scratch directory, cause each error deliberately and record the exact
message:

| Break this | Expected error |
|---|---|
| Set `required_version = "~> 1.5.0"` | Unsupported Terraform Core version |
| Delete `.terraform/` then `plan` | no package cached / module not installed |
| Reference `var.nope` | Reference to undeclared input variable |
| Two resources with the same name | Duplicate resource configuration |
| `for_each` over a `list(string)` | Invalid for_each argument |
| `count` and `for_each` on one resource | Invalid combination |
| Tab instead of spaces in HCL | Parse error |
| `terraform apply` with no `init` | Backend initialization required |

**Verify:** for each, write one sentence on *how you would recognise it in CI*,
where you only see the log.

This is the lab most people skip and the one that saves the most time later.

---

## Lab 9 — The quality gates, on purpose

**Goal:** make each gate fail, then fix it.
**Covers:** [ch. 12](12-quality-gates-and-testing.md)

Working in `arifa-infra` **on a scratch branch**:

```bash
git checkout -b lab-gates
```

1. **fmt** — add stray indentation to a `.tf` file. Run `make fmt-check`.
   Read the diff. Fix with `make fmt`.
2. **validate** — reference an undeclared variable. Run
   `bash scripts/check.sh validate`.
3. **tflint** — declare a variable with no `type`. Run `make lint`.
4. **trivy** — add an S3 bucket resource with no encryption block. Run
   `make scan`. Then suppress it correctly with an inline
   `#trivy:ignore:<ID>` **and a reason**.

**Verify:** each gate fails for the reason you expect, and `make check` reports
**every** failing gate in one run rather than stopping at the first.

**Then:**

```bash
git checkout -- . && git checkout main && git branch -D lab-gates
```

> That last `git checkout -- .` discards your experiments. Make sure you have
> nothing else uncommitted first.

---

## Lab 10 — Audit a real repository

**Goal:** apply chapter 15 to code you did not write.
**Covers:** [ch. 15](15-best-practices.md)

Read `arifa-infra` end to end and produce a written review answering:

1. Which resources would be destroyed if someone deleted `envs/dev/main.tf`?
2. Where is `prevent_destroy` used, and what else deserves it?
3. Find three comments explaining *why*. Find one place that needs one.
4. The repo has no `push: main` CI trigger and no scheduled drift detection.
   What is the risk of each, concretely?
5. `modules/*/README.md` is hand-written. What would make it drift-proof?
6. Which single change would most improve the repository?

**Verify:** your answers cite file and line, not impressions.

---

## Lab 11 — Backend migration

**Goal:** understand `-migrate-state` without an S3 bucket.
**Covers:** [ch. 07](07-state-and-backends.md)

In lab 2's directory, with state already applied:

```bash
mkdir -p ../lab11-backend
```

```hcl
# add to main.tf
terraform {
  backend "local" {
    path = "../lab11-backend/terraform.tfstate"
  }
}
```

```bash
terraform init -migrate-state     # answer yes
ls ../lab11-backend/
terraform plan                    # No changes — state moved, reality unchanged
```

**Verify:** the old `terraform.tfstate` is gone (or empty) and the new path
holds your resources.

**Think about:** this is exactly what
[`bootstrap/backend.tf`](../arifa-infra/bootstrap/backend.tf) describes for
moving bootstrap state into the S3 bucket it just created. Re-read that comment
now — it should make complete sense.

---

## Lab 12 — Expressions drill

**Goal:** fluency in HCL expressions.
**Covers:** [ch. 03](03-hcl-language.md)

```bash
terraform console
```

Produce each result without looking at the chapter:

```hcl
# 1. Merge two maps, second winning on conflict
# 2. From ["a","bb","ccc"], get only items longer than 1, uppercased
# 3. Turn { x = { id = 1 }, y = { id = 2 } } into { x = 1, y = 2 }
# 4. Return true if "dev" is in ["dev","prod"]
# 5. Build "arn:aws:s3:::my-bucket/envs/dev" from parts
# 6. Safely read a key that may not exist, defaulting to "none"
# 7. Validate "my-name-1" against ^[a-z0-9-]+$ returning a bool
# 8. Count the keys in a map
```

**Verify:** all eight in under five minutes, from memory.

---

## Suggested order

| If you have | Do |
|---|---|
| 20 minutes | Lab 3 — it teaches the most per minute |
| 1 hour | Labs 2, 3, 4 |
| An afternoon | Labs 1, 2, 3, 4, 5, 6 |
| A day | All of them, plus lab 10 written up properly |

## Clean up

```bash
rm -rf ~/tf-labs ~/tf-labs-backend
```

Confirm `arifa-infra` is clean:

```bash
cd arifa-infra && git status --short
```

---

Previous: [17 — Cheat sheet](17-cheatsheet.md) ·
Back to the [index](README.md)
