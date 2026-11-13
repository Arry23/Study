# AWS Solutions Architect – Associate, A to Z

A complete study guide for the AWS Certified Solutions Architect – Associate
exam, written as a course rather than a flashcard dump.

Wherever a concept appears in real code in this workspace, the guide points at
it. You are already using IAM roles, STS web-identity federation, S3 with
versioning and bucket policies, KMS, Secrets Manager, EKS with IRSA, SageMaker
and RDS — so a good deal of the syllabus is sitting in
[`arifa-infra/`](../arifa-infra) and [`common/`](../common) already.

> **Check the exam code.** This guide is written against **SAA-C03** and its
> domain weightings. AWS revises exams periodically. Download the current
> official exam guide from AWS Certification and confirm the blueprint before
> you book.

## The exam

| | |
|---|---|
| Code | SAA-C03 |
| Questions | 65 (50 scored, 15 unscored) |
| Time | 130 minutes |
| Format | Multiple choice, multiple response |
| Pass | 720 / 1000, scaled |
| Cost | 150 USD |

### Domains

| Domain | Weight | Chapters |
|--------|--------|----------|
| 1. Design Secure Architectures | **30%** | 02, 12, 13 |
| 2. Design Resilient Architectures | **26%** | 05, 08, 14 |
| 3. Design High-Performing Architectures | **24%** | 03, 04, 06, 07, 09, 10, 11, 17 |
| 4. Design Cost-Optimized Architectures | **20%** | 15, 16 |

Security is the largest single domain. If you are short of time, chapters 02
and 12 repay study more than any others.

## Chapters

| # | Chapter | Covers |
|---|---------|--------|
| 01 | [Fundamentals](01-fundamentals.md) | Regions, AZs, edge, shared responsibility, Well-Architected |
| 02 | [IAM and identity](02-iam-and-identity.md) | Users, roles, policies, evaluation logic, STS, Organizations, SCPs |
| 03 | [VPC networking](03-vpc-networking.md) | Subnets, routing, NAT, security groups, NACLs, endpoints, hybrid |
| 04 | [EC2 and compute](04-ec2-and-compute.md) | Instance families, purchasing models, AMIs, placement, metadata |
| 05 | [Load balancing and auto scaling](05-load-balancing-and-autoscaling.md) | ALB, NLB, GWLB, target groups, ASG policies |
| 06 | [S3](06-s3.md) | Storage classes, lifecycle, versioning, replication, encryption, access |
| 07 | [Block and file storage](07-block-and-file-storage.md) | EBS, snapshots, EFS, FSx, Storage Gateway |
| 08 | [Databases](08-databases.md) | RDS, Aurora, DynamoDB, ElastiCache, Redshift, Neptune |
| 09 | [Serverless and containers](09-serverless-and-containers.md) | Lambda, API Gateway, ECS, Fargate, EKS, Step Functions |
| 10 | [Application integration](10-application-integration.md) | SQS, SNS, EventBridge, Kinesis, MSK |
| 11 | [DNS, CDN and global reach](11-dns-cdn-and-global.md) | Route 53 policies, CloudFront, Global Accelerator |
| 12 | [Security services](12-security-services.md) | KMS, Secrets Manager, ACM, WAF, Shield, GuardDuty, Macie |
| 13 | [Monitoring and governance](13-monitoring-and-governance.md) | CloudWatch, CloudTrail, Config, X-Ray, Systems Manager |
| 14 | [Resilience and DR](14-resilience-and-dr.md) | RTO/RPO, the four DR strategies, AWS Backup, multi-region |
| 15 | [Cost optimization](15-cost-optimization.md) | Pricing models, Savings Plans, Spot, Budgets, data transfer |
| 16 | [Migration and transfer](16-migration-and-transfer.md) | DMS, MGN, Snow family, DataSync, Transfer Family |
| 17 | [Analytics](17-analytics.md) | Athena, Glue, EMR, Firehose, OpenSearch, QuickSight |
| 18 | [Exam strategy](18-exam-strategy.md) | Question patterns, keyword mapping, traps |
| 19 | [Service cheat sheet](19-service-cheatsheet.md) | One-page reference |
| 20 | [Practice scenarios](20-practice-scenarios.md) | Worked architecture questions |
| 21 | [Hands-on labs](21-hands-on-labs.md) | **15 labs, tiered by cost and risk — paper, read-only, LocalStack, build** |

## How to study

**Weeks 1–2 — foundations.** Chapters 01 to 07. These are the building blocks
and roughly half the exam. Do not move on until you can draw a VPC with public
and private subnets, a NAT gateway and an ALB from memory.

**Week 3 — data and compute.** Chapters 08 to 11.

**Week 4 — the cross-cutting domains.** Chapters 12 to 17. These are where the
"which service?" questions live.

**Week 5 — practice.** Chapters 18 to 20, plus timed practice exams. Aim for
85% on practice tests before booking; real exam questions are usually a little
harder.

### What actually works

- **Build things.** You have an AWS account in this workspace's world. A VPC
  built by hand teaches more than fifty flashcards.
- **Learn the decision tables**, not the service list. The exam rarely asks
  "what is SQS" — it asks "which of these four designs decouples this workload
  most cost-effectively".
- **Read the question's constraints first.** "Least operational overhead",
  "most cost-effective", "minimum downtime" and "highest availability" select
  different answers from the same four options.
- **Learn the defaults and limits.** Many questions hinge on one: a security
  group is stateful, a NACL is not; S3 is strongly consistent; a NAT gateway is
  per-AZ.

## Conventions

- **Exam tip** marks something frequently tested.
- **Trap** marks a distractor the exam uses deliberately.
- Code references point at real files in this workspace.

## Related guides here

- [Terraform, A to Z](../terraform-guide/README.md) — chapter 13 covers the AWS
  side of OIDC in depth
- [GitHub Actions, A to Z](../github-actions-guide/README.md)
- [GitLab CI/CD, A to Z](../gitlab-ci-guide/README.md)
