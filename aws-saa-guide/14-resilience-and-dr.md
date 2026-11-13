# 14 — Resilience and disaster recovery

The Resilient Architectures domain is 26% of the exam. Most of it reduces to
two numbers and four strategies.

## RTO and RPO

```
        disaster
           │
  ◄────────┼────────►
    RPO    │   RTO
 how much  │  how long
data can   │  until we're
 we lose   │   back up
```

- **RPO — Recovery Point Objective.** Maximum acceptable **data loss**,
  measured backwards from the disaster. Driven by **backup/replication
  frequency**.
- **RTO — Recovery Time Objective.** Maximum acceptable **downtime**, measured
  forwards. Driven by **how fast you can stand things up**.

> **Exam tip.** Read which one the question constrains. "Cannot lose more than
> 5 minutes of data" is **RPO** → replication frequency. "Must be back within
> 15 minutes" is **RTO** → how much is pre-provisioned.

## The four DR strategies

| Strategy | RTO / RPO | Cost | What is running |
|----------|-----------|------|-----------------|
| **Backup & Restore** | Hours | **$** | Nothing. Backups only |
| **Pilot Light** | Tens of minutes | $$ | Core data replicating; servers **off** |
| **Warm Standby** | Minutes | $$$ | A **scaled-down** full copy, running |
| **Multi-Site Active/Active** | **Near zero** | **$$$$** | Full capacity in both regions |

> **Exam tip — the trade-off is always cost vs RTO.** The question will give
> you one constraint and expect you to pick the cheapest option that meets it:
> - "Lowest cost, hours acceptable" → **Backup & Restore**
> - "Minimise cost but recover in minutes" → **Pilot Light** or **Warm Standby**
> - "Zero downtime, cost is not a concern" → **Multi-Site**

### Backup & Restore

Snapshots and backups copied cross-region. Restore on demand.

Services: **AWS Backup** (cross-region, cross-account copies), EBS snapshots,
RDS snapshots, S3 **CRR**.

### Pilot Light

Data replicates continuously; compute exists as AMIs/templates but is switched
off.

```
Region A (live)                Region B (pilot light)
ASG + ALB + RDS primary  ──►   RDS read replica (running)
                               AMIs + launch templates (not running)
                               Route 53 failover record
```

### Warm Standby

A smaller but **functional** copy. Scale it up on failover.

### Multi-Site Active/Active

Both regions serve traffic. Requires multi-region data: **Aurora Global
Database**, **DynamoDB Global Tables**, S3 CRR.

> **Exam tip.** Active/active with a relational database almost always means
> **Aurora Global Database**; with NoSQL it means **DynamoDB Global Tables**.

## Multi-AZ vs multi-region

| | Multi-AZ | Multi-region |
|---|---|---|
| Protects against | AZ failure | **Region** failure, or latency/compliance |
| Complexity | Low — often a checkbox | High |
| Typical use | **High availability** | **Disaster recovery** |

> **Exam tip.** The exam distinguishes "highly available" (multi-AZ) from
> "disaster recovery" (multi-region) very deliberately. Do not over-engineer:
> if the requirement is survival of an AZ failure, multi-region is the wrong
> (and more expensive) answer.

## High availability by service

| Service | How |
|---------|-----|
| EC2 | ASG across **≥ 2 AZs** behind an ELB |
| RDS | **Multi-AZ** (sync standby, automatic failover) |
| Aurora | 6 copies across 3 AZs; replicas as failover targets |
| DynamoDB | Multi-AZ by default; **Global Tables** for multi-region |
| S3 | Multi-AZ by default (except One Zone classes) |
| EFS | Multi-AZ by default (except One Zone) |
| ElastiCache Redis | Multi-AZ with automatic failover |
| Lambda / SQS / SNS | Multi-AZ by design |
| NAT gateway | **One per AZ** — it is zonal |

> **Trap — the single points of failure the exam plants:**
> - A **single NAT gateway** serving all AZs
> - An ASG with **min capacity 1**
> - An ASG in **one AZ**
> - **One Zone-IA** or One Zone EFS where durability matters
> - A **self-managed** database on a single EC2 instance

## Designing for failure

**Decouple** — SQS between tiers so a slow consumer does not break producers.

**Make it stateless** — session state in ElastiCache or DynamoDB, so any
instance can serve any request.

**Health checks everywhere** — ELB target health, Route 53 health checks, ASG
health check type `ELB`.

**Retry with exponential backoff and jitter** — AWS SDKs do this by default.

**Graceful degradation** — serve cached or partial results rather than failing.

**Circuit breakers** — stop hammering a failing dependency.

> **Exam tip.** "One slow component brings down the whole application" →
> decouple with a **queue**, and add timeouts/circuit breakers.

## AWS Backup

Central, policy-driven backup across EBS, EC2, RDS, Aurora, DynamoDB, EFS,
FSx, Storage Gateway, S3 and more.

- Backup plans: schedule, retention, lifecycle to cold storage
- **Cross-region** and **cross-account** copy
- **Vault Lock** — WORM, immutable even to root
- Audit Manager integration for compliance reporting

> **Exam tip.** "Centralised backup policy across many services and accounts"
> → **AWS Backup**. "Backups must survive a malicious administrator" →
> **Vault Lock**.

## Data replication summary

| Data | Multi-region mechanism | Typical RPO |
|------|------------------------|-------------|
| S3 | **CRR** (RTC for a 15-min SLA) | Minutes |
| RDS | Cross-region **read replica**, or snapshot copy | Minutes to hours |
| Aurora | **Global Database** | **< 1 second** |
| DynamoDB | **Global Tables** | **< 1 second** |
| EBS | Snapshot copy (via DLM / AWS Backup) | Hours |
| EFS | AWS Backup, or EFS replication | Minutes |

## Failover mechanics

| Mechanism | Failover speed | Notes |
|-----------|----------------|-------|
| **Route 53 failover** | DNS TTL dependent (tens of seconds to minutes) | Needs health checks |
| **Global Accelerator** | **Seconds** | No DNS dependency |
| **RDS Multi-AZ** | 60–120 s | CNAME swap, automatic |
| **Aurora** | **< 30 s** | Promote a replica |
| **ASG** | Minutes | Replace failed instances |

> **Exam tip.** "Failover must be faster than DNS allows" → **Global
> Accelerator**.

## Testing

> **Exam tip.** The Well-Architected principle is "**test recovery
> procedures**". An untested DR plan is an assumption. **AWS Fault Injection
> Service (FIS)** runs controlled chaos experiments — the right answer to
> "validate that the architecture actually tolerates failure".

**AWS Elastic Disaster Recovery (DRS)** continuously replicates servers into
AWS for low-RTO recovery, including from on-premises.

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| Survive an AZ failure | **Multi-AZ**: ASG + ELB, RDS Multi-AZ |
| Survive a region failure, cheapest | **Backup & Restore** cross-region |
| Region failure, recover in ~30 min, keep cost low | **Pilot Light** |
| Region failure, recover in minutes | **Warm Standby** |
| Zero downtime across regions | **Multi-Site** + Aurora Global / Global Tables |
| RPO < 1 second, relational | **Aurora Global Database** |
| RPO < 1 second, NoSQL | **DynamoDB Global Tables** |
| Restore a database to 14:32 | **PITR** |
| Immutable backups | **Backup Vault Lock** |
| Central backup across services | **AWS Backup** |
| Fail over faster than DNS | **Global Accelerator** |
| Prove the design survives failure | **FIS** |
| Replicate on-prem servers into AWS | **Elastic Disaster Recovery** |

## Check yourself

1. Define RTO and RPO. Which does replication frequency drive?
2. Name the four DR strategies in cost order.
3. What distinguishes Pilot Light from Warm Standby?
4. When is multi-region the wrong answer?
5. Why is a single NAT gateway a single point of failure?
6. Which gives the lowest RPO for a relational database?
7. Why is Global Accelerator failover faster than Route 53?
8. What does Vault Lock protect against that versioning does not?

---

Previous: [13 — Monitoring and governance](13-monitoring-and-governance.md) ·
Next: [15 — Cost optimization](15-cost-optimization.md)
