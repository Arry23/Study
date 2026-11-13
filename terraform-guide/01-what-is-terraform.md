# 01 — What Terraform is

## The problem it solves

You need a server, a database and a secret store in AWS. You have three options.

**Click them in the console.** Fast the first time. But nobody can review it,
nobody knows who changed what, and rebuilding the same thing in a second
environment means doing it all again from memory.

**Write a script.** `aws ec2 run-instances ...`. Now it is repeatable, but you
have to write all the "does it already exist?" logic yourself, and running it
twice either fails or creates duplicates.

**Declare the end state.** You write *what you want to exist*. Terraform works
out what to create, change or delete to get there. Running it twice is safe:
the second run finds nothing to do.

That third approach is **infrastructure as code**, and it is what Terraform does.

## Declarative, not imperative

A script says *do these steps*. Terraform config says *this is what should exist*:

```hcl
resource "aws_s3_bucket" "state" {
  bucket = "arifa-infrastructure-tfstate-483075907105-eu-central-1"
}
```

This does not mean "create a bucket". It means "a bucket with this name should
exist". Terraform compares that wish against reality and decides the action:

| Reality | Terraform does |
|---------|----------------|
| No such bucket | Creates it |
| Bucket exists, matches | Nothing |
| Bucket exists, differs | Changes it |
| You delete the block | Destroys it |

That last row is the one that bites people. **Deleting code deletes
infrastructure.** Chapter 11 covers how to remove code without destroying
anything.

## The three pieces

```
   your code              state file            real AWS
   (.tf files)      (terraform.tfstate)        (the truth)
       │                     │                      │
       └──────── plan compares all three ───────────┘
                             │
                      apply makes 2 and 3
                        match 1
```

1. **Configuration** — `.tf` files, committed to git. What you want.
2. **State** — a JSON file mapping each code block to a real resource ID.
   What Terraform believes exists.
3. **The real world** — what is actually in AWS.

`terraform plan` reads all three and prints the difference. `terraform apply`
carries it out.

State is covered properly in chapter 07. For now, just hold onto this: Terraform
cannot find your resources without it. Lose the state file and Terraform forgets
it manages anything.

## The core loop

```bash
terraform init      # download providers, connect to the backend. Once per clone.
terraform plan      # show me what would change. Safe, read-only.
terraform apply     # do it. Shows the plan again and asks for confirmation.
```

You will run `plan` hundreds of times and `apply` rarely. `plan` changes
nothing, so run it whenever you are unsure.

## What Terraform is not

- **Not a configuration manager.** It creates the VM; Ansible or a startup
  script installs software on it.
- **Not AWS-specific.** The same language manages Azure, GCP, Kubernetes,
  GitHub, Datadog and ~4,000 other providers. You learn HCL once.
- **Not a backup.** State is not a backup of your infrastructure, and deleting
  code really does delete resources.

## Where this sits in your workspace

[`arifa-infra/`](../arifa-infra) is a working example of all of this:

| Piece | Where |
|-------|-------|
| Configuration | `envs/dev/*.tf`, `modules/*/`, `bootstrap/*.tf` |
| State | an S3 bucket, configured in `envs/dev/backend.tf` |
| Real world | AWS account `483075907105`, region `eu-central-1` |

It currently manages one SageMaker notebook and three Secrets Manager
containers — small enough to read in one sitting.

## Why "declare, then plan" is safer than it sounds

The plan is the review artefact. Before anything changes, you get an exact
list:

```
Plan: 7 to add, 0 to change, 0 to destroy.
```

A human reads that and approves it. In `arifa-infra` this is formalised: CI runs
`plan` on every pull request and posts it to the job summary, and nothing in CI
is allowed to `apply`. A person applies from `main`, deliberately.

That is the real value. Not the automation — the **reviewability**.

## Try it

Nothing to install yet. Open these three files and just read them:

- `arifa-infra/envs/dev/main.tf` — what exists in dev
- `arifa-infra/modules/secrets/main.tf` — a reusable component
- `arifa-infra/bootstrap/state.tf` — the state bucket, defined in Terraform

You will not understand all the syntax. You should be able to guess what each
file is trying to achieve, which is the point.

## Check yourself

1. What are the three things `terraform plan` compares?
2. You delete a `resource` block and run `apply`. What happens to the real resource?
3. Why does running `apply` twice in a row not create two buckets?
4. Is `terraform plan` safe to run on production?

---

Next: [02 — Install and set up](02-install-and-setup.md)
