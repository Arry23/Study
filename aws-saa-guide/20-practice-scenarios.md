# 20 — Practice scenarios

Twenty worked questions in exam style. Cover the answer, decide, then read the
reasoning — especially **why the distractors are wrong**, which is where the
marks are.

---

### 1. Decoupling under load

A retail site on EC2 behind an ALB writes orders directly to RDS. During flash
sales the database saturates and orders are lost. Which change best protects
orders while absorbing spikes?

A. Enable RDS Multi-AZ
B. Add read replicas
C. Put an SQS queue between the web tier and order processing
D. Increase the instance size

<details><summary>Answer</summary>

**C.** The problem is **write** throughput during bursts and **lost** orders.
SQS buffers writes, decouples the tiers, and durably retains messages while
consumers catch up. Scale consumers on queue depth.

**A** is availability, not throughput. **B** scales reads; orders are writes.
**D** is vertical scaling — delays the problem, does not prevent loss.
</details>

---

### 2. Private subnet reaching S3

EC2 instances in private subnets download large objects from S3 via a NAT
gateway. The NAT data-processing charge is now the biggest line item. Cheapest
fix?

A. Move instances to public subnets
B. Create a **gateway VPC endpoint** for S3
C. Create an interface endpoint for S3
D. Use S3 Transfer Acceleration

<details><summary>Answer</summary>

**B.** A gateway endpoint for S3 is **free** and removes NAT processing and
data charges entirely.

**A** breaks the security posture. **C** works but costs hourly + per GB —
gateway is free and S3 supports it. **D** speeds up long-distance uploads and
adds cost.
</details>

---

### 3. HA for a stateful web tier

A web app stores session state in instance memory. It must survive an AZ
failure. Least-disruptive change?

A. Enable sticky sessions on the ALB
B. Store sessions in **ElastiCache Redis** and run the ASG across AZs
C. Multi-AZ RDS
D. Larger instances

<details><summary>Answer</summary>

**B.** Externalising session state makes the tier stateless, so any instance
can serve any request and an AZ loss costs nothing.

**A** keeps state on instances — losing one still loses sessions. **C**
addresses the database, not sessions. **D** does not help availability.
</details>

---

### 4. Cross-account S3, no credentials

A partner must download one object per week. They have no AWS account. Most
secure approach?

A. Make the bucket public
B. Create an IAM user and send access keys
C. Generate a **presigned URL** for each object
D. Enable static website hosting

<details><summary>Answer</summary>

**C.** A presigned URL grants time-limited access to **one object**, with no
AWS identity for the partner and no standing permission.

**A** and **D** expose the bucket. **B** creates long-lived credentials for an
external party — the thing the exam always rejects.
</details>

---

### 5. Database migration, different engine

Oracle on-premises → Aurora PostgreSQL. Downtime must be minimal. Which
combination?

A. DMS alone
B. **SCT** then **DMS with CDC**
C. Native dump and restore
D. Snowball then restore

<details><summary>Answer</summary>

**B.** Heterogeneous migration needs **two** things: SCT converts schema and
stored code; DMS with **change data capture** copies data and keeps replicating
until cutover.

**A** moves data but not the converted schema. **C** requires a long outage.
**D** addresses bulk transfer, not engine conversion or ongoing changes.
</details>

---

### 6. Blocking one attacker

A single IP is scraping an application behind an ALB. Security groups cannot
express the rule. What do you use?

A. A Deny rule in the security group
B. A **NACL** deny rule
C. Route 53 health checks
D. Remove the IGW

<details><summary>Answer</summary>

**B.** Security groups are **allow-only**; NACLs support explicit **Deny**.
(WAF with an IP set is also correct and more precise — but of the options
given, NACL is the one that works.)

**A** is impossible. **C** and **D** are unrelated or catastrophic.
</details>

---

### 7. Lambda timing out

A Lambda processes video files; some runs exceed 15 minutes. Least-change fix?

A. Increase the timeout
B. Add memory
C. Move to **ECS on Fargate**
D. Split with Step Functions Express

<details><summary>Answer</summary>

**C.** 15 minutes is a **hard limit**. Fargate runs containers with no time
limit and no server management.

**A** is impossible. **B** may help marginally but does not remove the ceiling.
**D** Express workflows are capped at 5 minutes, and splitting video processing
may not be feasible.
</details>

---

### 8. Compliance archive

Financial records must be retained for seven years, unmodifiable **by anyone
including administrators**, with lowest storage cost. Retrieval may take hours.

A. S3 Standard with versioning
B. S3 Glacier Deep Archive + **Object Lock Compliance mode**
C. S3 Glacier Flexible + Governance mode
D. EBS snapshots

<details><summary>Answer</summary>

**B.** Deep Archive is the cheapest class and hours of retrieval are
acceptable. **Compliance mode** prevents deletion by everyone, including the
root user.

**A** is expensive and deletable. **C** Governance mode allows privileged
override — fails "including administrators". **D** is not archival storage.
</details>

---

### 9. Global users, static IPs

A UDP game server runs in three regions. Players need the lowest latency,
automatic regional failover, and a **fixed set of IPs** for corporate
whitelisting.

A. CloudFront
B. Route 53 latency routing
C. **Global Accelerator**
D. NLB in each region

<details><summary>Answer</summary>

**C.** Global Accelerator provides **two static anycast IPs**, supports
**UDP**, routes over the AWS backbone, and fails over in **seconds** without
DNS TTL dependency.

**A** is HTTP-only and caches — wrong for gaming. **B** depends on DNS TTL and
gives no static IPs. **D** is per-region with separate IPs and no global
routing.
</details>

---

### 10. Unexpected S3 bill

A bucket holds 2 TB of current objects but is billed for 11 TB. Versioning is
enabled and multipart uploads are used. What explains it, and what fixes it?

A. Cross-region replication; disable it
B. Old **noncurrent versions** and **incomplete multipart uploads**; add
   lifecycle rules
C. Wrong storage class; move to IA
D. Too many requests; enable Transfer Acceleration

<details><summary>Answer</summary>

**B.** Both are billed and neither is visible in a normal object listing.
Lifecycle rules with `noncurrent_version_expiration` and
`abort_incomplete_multipart_upload` fix it — exactly what
[`arifa-infra/bootstrap/state.tf`](../arifa-infra/bootstrap/state.tf) does.

**A** would appear as a separate bucket. **C** reduces rate, not volume.
**D** is unrelated and adds cost.
</details>

---

### 11. Pod-level AWS access

An EKS pod needs to read one S3 bucket. Other pods on the same node must not.

A. Attach a policy to the node instance profile
B. Store access keys in a Kubernetes Secret
C. **IRSA** — map the service account to an IAM role
D. Run the pod with host networking

<details><summary>Answer</summary>

**C.** IRSA uses the cluster's OIDC provider to give **one service account** an
IAM role — exactly the pattern in
[`common/code/backend/common_utilities/dae_utils.py`](../common/code/backend/common_utilities/dae_utils.py).

**A** grants every pod on the node the same access. **B** is long-lived
credentials. **D** is irrelevant.
</details>

---

### 12. Read-heavy reporting

An RDS PostgreSQL instance is slow because analysts run heavy reports against
production. Minimal application change.

A. Multi-AZ
B. **Read replica**, point reports at it
C. Bigger instance
D. Migrate to DynamoDB

<details><summary>Answer</summary>

**B.** Read replicas offload read traffic. Reports use the replica endpoint;
the application is otherwise unchanged.

**A** standby is **not readable** (for Multi-AZ instance deployments). **C** is
costly and temporary. **D** is a rewrite, and wrong for analytical SQL.
</details>

---

### 13. Cold starts

A latency-sensitive API on Lambda shows occasional multi-second first
responses.

A. Increase memory
B. **Provisioned concurrency**
C. Reserved concurrency
D. Move to EC2

<details><summary>Answer</summary>

**B.** Provisioned concurrency keeps initialised environments warm, removing
cold starts.

**A** speeds init slightly but does not remove it. **C** **limits**
concurrency — it does not pre-warm. **D** is a large change with more overhead.
</details>

---

### 14. 400 TB over a 200 Mbps link

A data centre must move 400 TB to S3 within a month. The link is 200 Mbps and
also carries production traffic.

A. DataSync over the internet
B. **Snowball Edge** devices
C. Direct Connect
D. S3 Transfer Acceleration

<details><summary>Answer</summary>

**B.** 400 TB at 200 Mbps is well over a year of saturated link — impossible,
and it would starve production. Snowball ships physically.

**A** and **D** are bandwidth-bound. **C** takes weeks to provision and still
needs a very large circuit.
</details>

---

### 15. Least-privilege CI

A CI pipeline must run `terraform plan` against AWS on every pull request,
with no stored credentials, and must never read secret values.

A. IAM user with access keys in CI secrets
B. **OIDC federation** to a role with `ReadOnlyAccess` plus an explicit **Deny**
   on `secretsmanager:GetSecretValue`
C. Admin role assumed by CI
D. Shared credentials file on the runner

<details><summary>Answer</summary>

**B.** OIDC gives short-lived credentials with no stored keys; the trust policy
pins the repository; the explicit Deny overrides the broad managed policy
because **Deny always wins**. This is precisely
[`arifa-infra/bootstrap/github_oidc.tf`](../arifa-infra/bootstrap/github_oidc.tf).

**A** and **D** are long-lived credentials. **C** violates least privilege.
</details>

---

### 16. Fan-out with durability

An order event must be processed by billing, shipping and analytics. Each
consumes at its own rate, and no event may be lost if a consumer is down.

A. SNS → three Lambda subscriptions
B. **SNS → three SQS queues**, each with its own consumer
C. One SQS queue read by three consumers
D. EventBridge → one Lambda that calls all three

<details><summary>Answer</summary>

**B.** SNS fans out; each SQS queue gives **durable retention** so a consumer
can be down without loss, and each consumes at its own pace.

**A** has no retention if a Lambda fails past its retries. **C** gives each
message to **one** consumer only. **D** couples the three systems into one
failure domain.
</details>

---

### 17. Compliance routing

A streaming service is licensed per country. German users must be served from
`eu-central-1` **for legal reasons**, regardless of latency.

A. Latency routing
B. **Geolocation routing**
C. Weighted routing
D. Multivalue answer

<details><summary>Answer</summary>

**B.** The requirement is **legal**, not performance. Geolocation routes by the
user's country.

**A** optimises latency and could send a German user elsewhere. **C** is for
traffic shifting. **D** is crude load distribution.
</details>

---

### 18. Patch and access without SSH

Hundreds of instances in private subnets need OS patching and occasional
interactive access. Security forbids bastion hosts and inbound ports.

A. Bastion in a public subnet
B. **Systems Manager — Session Manager + Patch Manager**
C. Open 22 to the corporate CIDR
D. Site-to-Site VPN plus SSH

<details><summary>Answer</summary>

**B.** Session Manager gives audited shell access with **no inbound ports and
no bastion**; Patch Manager handles scheduled patching and compliance
reporting.

**A** and **C** are explicitly excluded. **D** still requires SSH and open
ports.
</details>

---

### 19. Cheapest archive query

Five years of CloudTrail logs sit in S3. Auditors query them perhaps twice a
year. Minimise cost while keeping SQL access.

A. Load into Redshift
B. **Keep in S3 (Glacier Instant/IA) and query with Athena**
C. Load into RDS
D. OpenSearch cluster

<details><summary>Answer</summary>

**B.** Athena is serverless and charges **per TB scanned** — ideal for rare
queries, with **no idle cost**. Convert to Parquet and partition to cut the
scan further.

**A**, **C** and **D** all run provisioned infrastructure continuously for two
queries a year.
</details>

---

### 20. RPO near zero, relational, two regions

A relational workload must survive the loss of an entire region with **under
one second** of data loss and recovery in minutes.

A. RDS Multi-AZ
B. Cross-region read replica
C. **Aurora Global Database**
D. Nightly snapshot copies

<details><summary>Answer</summary>

**C.** Aurora Global Database replicates cross-region with typically **< 1 s**
lag and promotes a secondary region in about a minute.

**A** is a single region — AZ protection only. **B** is asynchronous with
higher and more variable lag, and promotion is manual. **D** has an RPO of up
to 24 hours.
</details>

---

## Scoring yourself

| Correct | Verdict |
|---------|---------|
| 18–20 | Ready. Do timed full-length practice exams |
| 14–17 | Close. Review the chapters behind your misses |
| 10–13 | Keep studying — focus on the decision tables |
| < 10 | Re-read chapters 02, 03, 06, 08 and 14 |

## What to do with mistakes

For every wrong answer, write one sentence explaining **why each distractor is
wrong**. If you cannot, you have not learned the concept — only the answer.

The exam reuses the same distractor patterns:

- Vertical scaling where horizontal is needed
- Availability where throughput is the problem
- Long-lived credentials where a role fits
- Multi-region where multi-AZ suffices
- Self-managed where managed exists
- A design change where the question forbade one

---

Previous: [19 — Service cheat sheet](19-service-cheatsheet.md) ·
Next: [21 — Hands-on labs](21-hands-on-labs.md) ·
Back to the [index](README.md)
