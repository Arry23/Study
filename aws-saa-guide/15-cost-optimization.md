# 15 — Cost optimization

20% of the exam. Most questions give you a working architecture and ask for
the cheapest change that keeps it working.

## Compute pricing

| Option | Saving | Commitment | Use |
|--------|--------|------------|-----|
| **On-Demand** | — | None | Spiky, short, unpredictable |
| **Savings Plans** | up to ~72% | 1 or 3 yr, **$/hour** | Steady-state, flexible |
| **Reserved Instances** | up to ~72% | 1 or 3 yr, **specific shape** | Steady-state, known shape |
| **Spot** | up to **90%** | None | **Interruptible**, fault-tolerant |
| **Dedicated Host** | — | Optional | BYOL, compliance |

### Savings Plans

| Type | Covers | Flexibility |
|------|--------|-------------|
| **Compute Savings Plan** | EC2, **Fargate, Lambda**, any region/family/OS | **Most flexible** |
| **EC2 Instance Savings Plan** | One family in one region | Bigger discount, less flexible |

> **Exam tip.** "Commitment-based savings that also covers Fargate and Lambda"
> → **Compute Savings Plan**. This is now usually preferred over RIs.

### Reserved Instances

| | Standard RI | Convertible RI |
|---|---|---|
| Discount | Higher | Lower |
| Exchange | No | **Yes**, for different attributes |
| Marketplace | **Can sell** | No |

**Zonal RIs** additionally **reserve capacity** in an AZ. Regional RIs do not.

> **Exam tip.** "Guarantee capacity is available in a specific AZ" → **zonal
> RI** or an **On-Demand Capacity Reservation**. A Savings Plan does **not**
> reserve capacity.

### Spot

- Up to 90% off, **2-minute** interruption warning
- **Spot Fleet / EC2 Fleet** diversify across pools
- Pair with ASG mixed instances policy: baseline On-Demand + Spot for burst

> **Exam tip.** Spot appears whenever a question says "**batch**",
> "**stateless**", "**fault tolerant**", "**can be restarted**", "**flexible
> start and end times**". Never for a database or a stateful singleton.

## Storage cost

| Lever | Saving |
|-------|--------|
| **S3 lifecycle** to IA / Glacier | Large |
| **S3 Intelligent-Tiering** | Automatic, when patterns are unknown |
| Delete **old versions** and **incomplete multipart uploads** | Often surprising |
| **gp3 instead of gp2** | ~20% on EBS |
| Delete **unattached EBS volumes** and stale snapshots | Pure waste |
| **One Zone-IA** for reproducible data | ~20% vs Standard-IA |
| Right-size over-provisioned IOPS | Large on io1/io2 |

> **Exam tip.** "Storage bill is higher than expected despite few objects" →
> **old versions** and **incomplete multipart uploads**. Lifecycle rules fix
> both.

```hcl
# arifa-infra/bootstrap/state.tf — both levers in one rule
noncurrent_version_expiration { noncurrent_days = 90 }
abort_incomplete_multipart_upload { days_after_initiation = 7 }
```

## Data transfer — the hidden cost

This is where cost questions usually hide.

| Path | Cost |
|------|------|
| Inbound from internet | **Free** |
| Outbound to internet | **Charged** (the big one) |
| Within the same AZ, private IPs | **Free** |
| **Between AZs** | **Charged both directions** |
| Between regions | Charged |
| **Via NAT gateway** | Hourly **+ per GB processed** |
| **Via VPC gateway endpoint (S3/DynamoDB)** | **Free** |
| Out via **CloudFront** | Cheaper than direct |

> **Exam tip — the three classic savings:**
> 1. Private subnet pulling from S3 through a NAT gateway → **gateway VPC
>    endpoint**, removing both NAT processing and the data charge.
> 2. Chatty cross-AZ traffic → keep callers and callees **in the same AZ**
>    where availability allows.
> 3. High egress to end users → put **CloudFront** in front.

> **Trap.** Cross-AZ traffic is charged in **both** directions. A chatty
> microservice mesh spread across AZs can cost more than the compute.

## Database cost

| Lever | Note |
|-------|------|
| **Reserved Instances** for RDS/ElastiCache/Redshift/OpenSearch | Same model as EC2 |
| **Aurora Serverless v2** | Pay for capacity actually used |
| **DynamoDB on-demand vs provisioned** | Provisioned + auto scaling is cheaper for predictable load |
| **DynamoDB reserved capacity** | For steady provisioned throughput |
| Stop non-production databases | RDS can be stopped up to 7 days |
| Read replicas only where needed | Each is a full instance |

> **Exam tip.** "DynamoDB cost is high and traffic is predictable" → switch
> from **on-demand to provisioned** with auto scaling. The reverse for spiky
> or unknown traffic.

## Serverless and container cost

- Lambda: **more memory can be cheaper** if it shortens duration enough
- **Graviton** (ARM) for Lambda, Fargate, EC2, RDS — better price/performance
- **Fargate Spot** for interruptible tasks
- Right-size Fargate task CPU/memory

## The cost tooling

| Tool | Purpose |
|------|---------|
| **Cost Explorer** | Visualise and forecast spend; RI/SP recommendations |
| **AWS Budgets** | Alert or act on cost, usage, RI/SP coverage |
| **Cost and Usage Report (CUR)** | The most granular data, to S3 for analysis |
| **Compute Optimizer** | ML right-sizing recommendations |
| **Trusted Advisor** | Idle resources, low utilisation, unused IPs |
| **Billing Conductor / Cost Categories** | Chargeback and showback |
| **Savings Plans recommendations** | What to commit to |

> **Exam tip.**
> - "Alert when spend exceeds $X" → **AWS Budgets**
> - "Analyse and forecast trends" → **Cost Explorer**
> - "Most detailed line-item data for custom analysis" → **CUR**
> - "Which instance type should we move to" → **Compute Optimizer**

## Tagging and allocation

**Cost allocation tags** (user-defined, activated in Billing) attribute spend
to teams and projects. Enforce them with **Tag Policies** or an SCP.

This workspace tags consistently through the provider:

```hcl
default_tags {
  tags = {
    Project     = "Arifa"
    Environment = "dev"
    ManagedBy   = "terraform"
    Repository  = "roche-innersource/arifa-infrastructure"
  }
}
```

> **Exam tip.** "Attribute cost by department" → **cost allocation tags**
> activated in the Billing console, enforced with a **tag policy**.

## Organizations and billing

- **Consolidated billing** — one bill, and **volume discounts aggregate**
  across accounts
- RIs and Savings Plans are **shared across the organization** by default
  (can be disabled per account)

> **Exam tip.** "Get volume discounts across many accounts" →
> **consolidated billing** under Organizations.

## A cost-optimization checklist

1. **Right-size** — Compute Optimizer
2. **Commit** — Savings Plans for the steady baseline
3. **Use Spot** for anything interruptible
4. **Tier storage** — lifecycle rules, Intelligent-Tiering
5. **Delete waste** — unattached volumes, old snapshots, idle load balancers,
   unassociated Elastic IPs, old object versions
6. **Fix data transfer** — endpoints, same-AZ, CloudFront
7. **Turn off non-production** out of hours
8. **Tag and attribute**, then make teams accountable
9. **Move up the managed-service ladder** — fewer servers, less idle capacity

> **Trap.** An **unassociated Elastic IP** is charged. So is an idle
> load balancer, a provisioned-IOPS volume at low utilisation, and a NAT
> gateway that nothing uses.

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| Steady 3-year compute, keep flexibility | **Compute Savings Plan** |
| Guarantee capacity in an AZ | **Zonal RI** / Capacity Reservation |
| Interruptible batch, cheapest | **Spot** |
| Unknown S3 access patterns | **Intelligent-Tiering** |
| Archive for 7 years, cheapest | **Glacier Deep Archive** |
| Unexpected S3 bill | Old versions + incomplete multipart uploads |
| EBS cost reduction, same performance | **gp3** |
| Private subnet pulling from S3 | **Gateway VPC endpoint** |
| High internet egress | **CloudFront** |
| Predictable DynamoDB traffic | **Provisioned** + auto scaling |
| Spiky DynamoDB traffic | **On-demand** |
| Variable relational load | **Aurora Serverless v2** |
| Alert at 80% of budget | **AWS Budgets** |
| Granular line-item analysis | **CUR** in S3, queried with Athena |
| Attribute spend to teams | **Cost allocation tags** |
| Volume discounts across accounts | **Consolidated billing** |

## Check yourself

1. Compute SP vs EC2 Instance SP vs Standard RI — rank by flexibility.
2. Does a Savings Plan reserve capacity?
3. Which two S3 settings commonly cause a surprise bill?
4. Why is cross-AZ traffic expensive, and how do you avoid it?
5. How does a gateway endpoint save money twice over?
6. When is DynamoDB provisioned cheaper than on-demand?
7. Name three resources that cost money while doing nothing.
8. Budgets vs Cost Explorer vs CUR?

---

Previous: [14 — Resilience and DR](14-resilience-and-dr.md) ·
Next: [16 — Migration and transfer](16-migration-and-transfer.md)
