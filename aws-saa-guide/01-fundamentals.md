# 01 — Fundamentals

## The global infrastructure

```
Region (eu-central-1, Frankfurt)
├── Availability Zone eu-central-1a   ─┐
├── Availability Zone eu-central-1b    ├─ separate power, cooling, networking
└── Availability Zone eu-central-1c   ─┘   single-digit ms latency between them
```

**Region** — a geographic area with multiple AZs. Resources are region-scoped
unless stated otherwise. You choose a region for latency, compliance, service
availability and price.

**Availability Zone** — one or more discrete data centres within a region.
Physically separated, so a flood or power failure takes out one AZ, not the
region.

**Edge location** — hundreds of sites running CloudFront, Route 53 and Global
Accelerator. Caching and DNS, not compute.

**Local Zone / Wavelength / Outposts** — AWS infrastructure pushed closer to
users, telecom networks, or into your own data centre.

> **Exam tip.** "Highly available" nearly always means **multi-AZ**.
> "Disaster recovery" or "regional outage" means **multi-region**. The exam
> distinguishes these sharply.

This workspace runs entirely in **eu-central-1**, pinned deliberately:

```hcl
# arifa-infra/envs/dev/providers.tf
provider "aws" {
  region              = local.region
  allowed_account_ids = [local.account_id]
}
```

## Regional vs zonal vs global

| Scope | Services |
|-------|----------|
| **Global** | IAM, Route 53, CloudFront, WAF (for CloudFront), Organizations |
| **Regional** | S3, DynamoDB, Lambda, SQS, SNS, ELB, RDS (the service) |
| **Zonal** | EC2 instance, EBS volume, subnet, RDS instance, NAT gateway |

> **Trap.** An EBS volume lives in **one AZ** and cannot attach to an instance
> in another. Moving it means snapshot → create volume in the target AZ.
>
> A subnet is also in one AZ. "Spread across AZs" means multiple subnets.

## The shared responsibility model

```
          YOU                    │              AWS
 ─────────────────────────────── │ ───────────────────────────────
 Customer data                   │  Compute, storage, networking
 Platform, apps, identity        │  Database/managed service software
 OS, network, firewall config    │  Regions, AZs, edge locations
 Client & server encryption      │  Physical security
 Network traffic protection      │  Hypervisor
 ─────────────────────────────── │ ───────────────────────────────
 Security IN the cloud           │  Security OF the cloud
```

The line moves by service type:

| Model | You patch | AWS patches |
|-------|-----------|-------------|
| **IaaS** (EC2) | OS, runtime, app | Hypervisor and below |
| **PaaS** (RDS, Lambda) | App, data, IAM | OS, runtime, engine |
| **SaaS** (S3, DynamoDB) | Data, IAM, encryption choices | Everything else |

> **Exam tip.** Questions saying "reduce operational overhead" are steering you
> up this table — EC2 → ECS on Fargate → Lambda, or self-managed database →
> RDS → Aurora Serverless → DynamoDB.

**You always own:** IAM configuration, data classification, and whether
something is public.

## The Well-Architected Framework

Six pillars. The exam tests the trade-offs between them.

| Pillar | Core question |
|--------|---------------|
| **Operational excellence** | Can we run and monitor this, and improve it? |
| **Security** | Least privilege, defence in depth, encryption, traceability |
| **Reliability** | Does it recover automatically? Do we scale horizontally? |
| **Performance efficiency** | Right resource, right size, right place |
| **Cost optimization** | Are we paying for what we use, at the best rate? |
| **Sustainability** | Minimise the environmental impact of the workload |

Design principles worth memorising:

- **Stop guessing capacity** — scale automatically
- **Test recovery procedures** — do not assume them
- **Automate to make change easy and reversible**
- **Design for failure; assume everything fails eventually**
- **Decouple components** so failure is contained
- **Use managed services** to reduce undifferentiated work

## Account structure

**Root user** — created with the account, has unrestricted access.

> **Exam tip.** Never use root for daily work. Lock it away with MFA, and use
> it only for the handful of tasks that require it (closing the account,
> changing support plan, some billing settings).

**AWS Organizations** — group accounts, apply **Service Control Policies**
(guardrails), consolidate billing. Covered in chapter 02.

A common multi-account layout:

```
Organization root
├── Security OU      (log archive, security tooling)
├── Infrastructure OU (shared networking, CI/CD)
└── Workloads OU
    ├── dev
    ├── staging
    └── prod
```

Separate accounts give the strongest blast-radius isolation — stronger than
IAM policies within one account.

This workspace lives in a shared Roche Cloud Platform account (`483075907105`)
with platform guardrails imposed from above, which is why the Terraform
provider has to work around a denied tag:

```hcl
# RCP's guardrails deny changing the owner tag, so no plan may touch it.
ignore_tags {
  keys = ["owner", "Owner"]
}
```

That is an SCP-style guardrail in practice.

## Ways to interact with AWS

| Interface | Use |
|-----------|-----|
| Console | Learning, one-off inspection |
| CLI | Scripting, CI |
| SDK (boto3) | Application code |
| CloudFormation / CDK | AWS-native IaC |
| Terraform | Multi-cloud IaC |

All four appear here: Terraform in `arifa-infra/`, boto3 throughout `common/`
and `arifa/`, and the AWS CLI in the documented workflows.

```python
# common/code/backend/common_utilities/dae_utils.py
sts_client = boto3.client("sts", region_name=aws_region)
account_id = sts_client.get_caller_identity()["Account"]
```

> **Exam tip.** The exam does not test SDK syntax. It tests *which* service and
> *why*.

## Pricing in one page

Three fundamentals:

1. **Pay for what you use** — no upfront commitment required
2. **Pay less when you commit** — Reserved Instances, Savings Plans
3. **Pay less at scale** — tiered pricing

What is free vs charged, as a rule of thumb:

| Usually free | Usually charged |
|--------------|-----------------|
| Inbound data transfer | **Outbound** data transfer to internet |
| Data transfer within an AZ | Data transfer **between AZs** |
| S3 PUT to the same region | Cross-region replication |
| First 1 GB/month out | NAT gateway data processing |

> **Exam tip.** Data transfer is the hidden cost in most cost-optimization
> questions. Cross-AZ and NAT gateway charges are the usual culprits. Chapter 15.

## Support plans

| Plan | Response | Notable |
|------|----------|---------|
| Basic | — | Free; docs and forums only |
| Developer | 12–24 h | Business hours, one contact |
| Business | 1 h urgent | 24/7, full Trusted Advisor, API access |
| Enterprise On-Ramp | 30 min critical | Pool of TAMs |
| Enterprise | **15 min critical** | Dedicated TAM, concierge |

> **Exam tip.** "Dedicated Technical Account Manager" → Enterprise.
> "Full Trusted Advisor checks" → Business and above.

## Check yourself

1. Difference between a Region, an AZ and an edge location?
2. Which of these is global: IAM, S3, Route 53, EBS?
3. Why can't an EBS volume attach to an instance in another AZ?
4. Under shared responsibility, who patches the OS on EC2? On RDS?
5. Which pillar does "decouple components" belong to?
6. Name two things that are almost always charged for data transfer.
7. When is it legitimate to use the root user?

---

Next: [02 — IAM and identity](02-iam-and-identity.md)
