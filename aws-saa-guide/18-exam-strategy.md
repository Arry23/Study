# 18 — Exam strategy

Knowing the services is necessary but not sufficient. Most people who fail
know the content and misread the questions.

## How questions are built

A typical question has three parts:

```
[Scenario]  A company runs a web application on EC2 behind an ALB...
[Problem]   ...and users report slow response times during peak hours.
[Constraint] Which solution meets these requirements with the LEAST
             operational overhead?
```

**Read the constraint first.** It is the last sentence, and it decides which
of several technically correct answers is the right one.

## The constraint keywords

| Constraint | Prefer |
|------------|--------|
| **"Least operational overhead"** / "minimal management" | **Managed/serverless**: Lambda, Fargate, Aurora Serverless, DynamoDB |
| **"Most cost-effective"** / "lowest cost" | Spot, S3 tiering, right-sizing, endpoints, serverless |
| **"Highly available"** | **Multi-AZ** — ASG across AZs, RDS Multi-AZ |
| **"Disaster recovery"** / "regional outage" | **Multi-region** |
| **"Minimal downtime"** (migration) | **DMS with CDC**, blue-green |
| **"Minimal changes to the application"** | Lift-and-shift, ALB, RDS — **not** refactoring to serverless |
| **"Real time"** | Kinesis, Lambda, EventBridge |
| **"Near real time"** | Firehose (~60 s buffer) |
| **"Fully managed"** | Exclude anything self-hosted on EC2 |
| **"Must not lose data"** | SQS, versioning, Multi-AZ, backups |

> Two questions with identical scenarios and different final sentences have
> **different answers**. The exam does this deliberately.

## Elimination tactics

Most questions have two obviously wrong options. Remove them, then decide
between two plausible ones.

**Eliminate immediately:**

- **IAM users with access keys** where a role would work
- **Storing credentials in code, environment variables or AMIs**
- **Classic Load Balancer** — legacy
- **EC2 running something AWS manages** (self-managed MySQL vs RDS) when the
  question wants low overhead
- **Making an S3 bucket public** to share with one party — use a presigned URL
  or bucket policy
- **Multi-region** when the requirement is only AZ resilience — over-engineered
  and more expensive
- **Changing the application** when the question says "no application changes"
- Anything **polling on a schedule** when an event-driven option exists

**Treat with suspicion:** answers containing "always", "all", "every", or
granting `"Action": "*"`.

## Common traps

| Trap | Reality |
|------|---------|
| Security group Deny rule | Security groups **cannot deny**. Use a NACL |
| NACL without ephemeral ports | Stateless — **return traffic needs a rule** |
| Single NAT gateway for all AZs | Zonal. **One per AZ** for HA |
| CNAME at the zone apex | Not allowed. Use an **Alias** |
| ACM cert in the wrong region | **us-east-1** for CloudFront |
| EBS across AZs | Zonal. Snapshot and restore |
| Multi-AZ gives read scaling | No — **read replicas** do |
| Read replica gives automatic failover | No — **Multi-AZ** does |
| Add an LSI later | **Impossible**. Use a GSI |
| Lambda over 15 minutes | Use Fargate/Batch/Step Functions |
| Lambda in a VPC has internet | Needs a **NAT gateway** |
| Instance store persists | **Lost** on stop/terminate |
| Gateway endpoints for any service | **S3 and DynamoDB only** |
| VPC peering is transitive | **It is not**. Use Transit Gateway |
| S3 is eventually consistent | **Strongly consistent** since 2020 |
| Config prevents changes | It **detects**; SCP prevents |
| Encrypt an existing RDS/EBS in place | Snapshot → copy encrypted → restore |

## Keyword → service

| Keyword in the question | Service |
|-------------------------|---------|
| "path-based routing", "host-based" | **ALB** |
| "static IP", "UDP", "millions of connections" | **NLB** |
| "third-party appliance" | **GWLB** |
| "decouple", "buffer", "spike" | **SQS** |
| "fan-out", "notify multiple" | **SNS** |
| "route by event content", "react to AWS events" | **EventBridge** |
| "replay", "ordered stream", "real time" | **Kinesis Data Streams** |
| "deliver to S3 with no code" | **Firehose** |
| "microsecond", "DynamoDB cache" | **DAX** |
| "session state", "cache" | **ElastiCache** |
| "graph", "relationships", "fraud ring" | **Neptune** |
| "SQL over S3, serverless" | **Athena** |
| "data warehouse", "BI", "petabyte SQL" | **Redshift** |
| "full-text search", "log dashboards" | **OpenSearch** |
| "shared Linux filesystem" | **EFS** |
| "Windows", "SMB", "Active Directory" | **FSx for Windows** |
| "HPC", "ML training throughput" | **FSx for Lustre** |
| "tape replacement" | **Tape Gateway** |
| "SFTP" | **Transfer Family** |
| "petabytes", "limited bandwidth" | **Snowball / Snowmobile** |
| "migrate database, minimal downtime" | **DMS** (+ SCT if heterogeneous) |
| "lift and shift servers" | **MGN** |
| "rotate credentials automatically" | **Secrets Manager** |
| "cheap config storage" | **Parameter Store** |
| "SQL injection", "XSS", "rate limit" | **WAF** |
| "DDoS with expert support" | **Shield Advanced** |
| "compromised credentials", "crypto mining" | **GuardDuty** |
| "PII in S3" | **Macie** |
| "CVEs", "vulnerability scan" | **Inspector** |
| "who called the API" | **CloudTrail** |
| "configuration compliance", "drift" | **AWS Config** |
| "no bastion, no SSH" | **Session Manager** |
| "find the slow microservice" | **X-Ray** |
| "guardrail across all accounts" | **SCP** |
| "cap what delegated admins can grant" | **Permissions boundary** |
| "pod needs AWS access" | **IRSA** |
| "no credentials in CI" | **OIDC federation** |

## Timing

130 minutes, 65 questions ≈ **2 minutes each**.

1. **First pass** — answer what you know. Flag anything that takes over
   ~90 seconds and move on.
2. **Second pass** — work the flagged ones with proper elimination.
3. **Third pass** — if time remains, re-check only questions you flagged as
   genuinely uncertain. Do not second-guess confident answers; first instincts
   are usually right.

**Never leave a blank.** There is no penalty for a wrong answer.

For multiple-response questions ("Select TWO"), the count is a hint: the
correct answers usually address **two different aspects** — for example one
networking change and one IAM change.

## The week before

- Take **two timed practice exams** under real conditions. Aim for **85%+**.
- Review **every** wrong answer until you can explain why each distractor is
  wrong, not just why the right one is right.
- Re-read the decision tables in chapters 02, 03, 06, 08 and 14.
- Redraw the standard VPC diagram from memory.
- Do **not** learn new services in the final two days. Consolidate.

## On the day

- Know whether it is at a test centre or online-proctored; online has strict
  environment rules.
- Arrive early; bring two forms of ID.
- Use the whiteboard/scratch area for the bandwidth calculation and for
  sketching architectures.
- If a question is genuinely unclear, pick the **most AWS-idiomatic** answer:
  managed over self-managed, roles over keys, multi-AZ over single, decoupled
  over tightly coupled.

## The mindset

The exam asks what a **solutions architect** would recommend, not what is
technically possible. When two answers both work:

1. Which uses **managed services**?
2. Which is **more secure** by default?
3. Which is **cheaper** to operate?
4. Which requires **fewer application changes**?
5. Which would you be comfortable defending in a design review?

## Check yourself

1. Where in a question is the deciding constraint usually found?
2. "Least operational overhead" steers you toward what?
3. "Highly available" vs "disaster recovery" — which scope each?
4. Name five answers you can eliminate on sight.
5. Why can't a security group deny traffic?
6. How long per question, and what is the three-pass approach?
7. What score should you reach on practice tests before booking?

---

Previous: [17 — Analytics](17-analytics.md) ·
Next: [19 — Service cheat sheet](19-service-cheatsheet.md)
