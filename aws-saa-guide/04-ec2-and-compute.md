# 04 — EC2 and compute

## Instance families

The letter tells you the workload.

| Family | Optimised for | Examples |
|--------|---------------|----------|
| **T** | Burstable, general purpose | t3, t4g — dev, low traffic |
| **M** | Balanced general purpose | m6i, m7g — app servers |
| **C** | Compute (high CPU:RAM) | c6i, c7g — batch, HPC, gaming |
| **R** | Memory | r6i — in-memory caches, big databases |
| **X** | Extreme memory | x2 — SAP HANA |
| **I / D / H** | Storage, high local IOPS | i4i — NoSQL, data warehouse |
| **P / G / Inf / Trn** | GPU / ML accelerators | p5, g5, inf2 — training, inference |

`g` in the name means **Graviton** (ARM) — typically ~20% better
price/performance, if your software runs on ARM.

> **Exam tip.** "CPU credits exhausted" / "performance degrades after a while"
> → a **T-family** instance running out of burst credits. Fix: switch to
> unlimited mode, or move to M-family.

The SageMaker notebook in this workspace is `ml.t3.medium` — a burstable
instance, appropriate for intermittent notebook work.

## Purchasing options

The most examined cost topic.

| Option | Discount | Commitment | Use for |
|--------|----------|------------|---------|
| **On-Demand** | — | None | Spiky, unpredictable, short-lived |
| **Savings Plans** | up to ~72% | 1 or 3 years, $/hour | **Flexible** steady-state |
| **Reserved Instances** | up to ~72% | 1 or 3 years, specific | Steady-state, known shape |
| **Spot** | up to **90%** | None, **2-minute warning** | Fault-tolerant, interruptible |
| **Dedicated Instance** | — | — | Hardware isolation |
| **Dedicated Host** | — | Optional | **BYOL**, socket licensing, compliance |
| **Capacity Reservation** | — | None (pay regardless) | Guarantee capacity in an AZ |

### Savings Plans vs Reserved Instances

| | Savings Plans | RIs |
|---|---|---|
| Commit to | $/hour of compute | Specific instance type/family |
| Flexibility | Change family, size, OS, region (Compute SP) | Limited; Convertible RIs allow exchange |
| Covers | EC2, **Fargate, Lambda** (Compute SP) | EC2, RDS, Redshift, ElastiCache, OpenSearch |
| Capacity reservation | No | Zonal RIs only |

> **Exam tip.** Modern answers favour **Savings Plans** for flexibility. Choose
> **Standard RIs** when the question stresses maximum discount with a fixed
> shape, or **zonal RI** when it also needs **capacity reservation**.

> **Exam tip — Spot.** "Can tolerate interruption", "batch", "stateless
> workers", "big data processing" → **Spot**. Never for a stateful database or
> anything requiring guaranteed uptime.

**Spot Fleet / EC2 Fleet** mix Spot and On-Demand across pools.
**Spot Instance interruption notice** arrives 2 minutes ahead via instance
metadata or EventBridge.

## AMIs

An **Amazon Machine Image** is the template: OS, software, permissions, block
device mapping.

- Region-scoped — **copy** to use in another region
- Can be shared with specific accounts or made public
- **Golden AMI** pattern: bake dependencies in, so instances boot fast and
  identically

> **Exam tip.** "Reduce instance boot/configuration time" → pre-baked AMI
> (often via **EC2 Image Builder**). "Run the same config everywhere" → AMI +
> launch template + ASG.

## Storage for EC2

| | EBS | Instance store |
|---|---|---|
| Persistence | **Survives** stop/terminate (if configured) | **Lost** on stop/terminate |
| Attachment | Network-attached, one AZ | Physically attached |
| Performance | Up to 256,000 IOPS (io2 Block Express) | Highest possible, NVMe |
| Snapshots | Yes, to S3 | No |

> **Exam tip.** "Temporary scratch / cache / buffer with maximum IOPS" →
> **instance store**. "Data must survive a stop" → **EBS**.

Full detail in [chapter 07](07-block-and-file-storage.md).

## Placement groups

| Type | Layout | Use |
|------|--------|-----|
| **Cluster** | Same rack, same AZ | Lowest latency, highest throughput — HPC |
| **Spread** | Distinct hardware, up to 7 per AZ | Small number of critical instances |
| **Partition** | Groups on distinct racks | HDFS, Cassandra, Kafka |

> **Exam tip.** "Tightly coupled HPC, lowest network latency" → **cluster**.
> "Maximise availability for a few critical instances" → **spread**.
> "Large distributed data store aware of racks" → **partition**.

Cluster placement groups trade availability for performance — a rack failure
takes everything.

## Instance metadata (IMDS)

```
http://169.254.169.254/latest/meta-data/
```

Reachable from the instance only. Exposes instance ID, AZ, IAM credentials,
user data.

> **Exam tip.** **IMDSv2** requires a session token (`PUT` then `GET`), which
> defeats SSRF attacks that trick an app into reading instance credentials.
> "Protect against SSRF reading IAM credentials" → **enforce IMDSv2**.

This workspace hardcodes it on the SageMaker notebook — deliberately not a
variable:

```hcl
# arifa-infra/modules/sagemaker-notebook/main.tf
instance_metadata_service_configuration {
  minimum_instance_metadata_service_version = "2"
}
```

**User data** is a script run at first boot (as root). Base64-encoded, 16 KB
limit.

## Instance lifecycle

| State | Billed? | Notes |
|-------|---------|-------|
| Running | Yes | |
| Stopped | **No** (EBS still charged) | Public IP released; moves host on start |
| Stopped-hibernated | No (EBS charged) | RAM saved to EBS root volume |
| Terminated | No | Root volume deleted by default |

> **Exam tip.** "Preserve in-memory state across a stop" → **hibernation**
> (requires encrypted EBS root large enough for RAM).

**Termination protection** prevents accidental termination.
`DeleteOnTermination` controls whether the root volume survives.

## Right-sizing and compute choice

The exam loves "reduce operational overhead". The ladder:

```
EC2 (you manage OS)
  → ECS on EC2       (you manage instances)
    → ECS on Fargate (no instances)
      → Lambda       (no servers, event-driven)
```

| Signal in the question | Answer |
|---|---|
| "Minimal operational overhead", short tasks | **Lambda** |
| Containers, no servers to manage | **Fargate** |
| Long-running, needs OS control, specific licensing | **EC2** |
| Steady predictable load, lowest cost | EC2 + Savings Plan |
| Bursty, interruptible batch | **Spot** |
| Existing Kubernetes investment | **EKS** |

Chapter 09 covers these properly.

## Licensing and tenancy

| Need | Answer |
|------|--------|
| BYOL with socket/core licensing | **Dedicated Host** |
| Physical isolation, no licence concern | **Dedicated Instance** |
| Track licence usage | **AWS License Manager** |

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| Interruptible batch, cheapest | **Spot** |
| Steady 3-year workload, flexible | **Compute Savings Plan** |
| Guaranteed capacity in an AZ | **Zonal RI** or **Capacity Reservation** |
| Lowest inter-node latency | **Cluster placement group** |
| Critical instances on separate hardware | **Spread placement group** |
| Defend against SSRF stealing credentials | **IMDSv2** |
| Fast, identical instance launches | **Golden AMI** |
| Scratch space, maximum IOPS | **Instance store** |
| Keep RAM across stop | **Hibernation** |
| Socket-based BYOL | **Dedicated Host** |

## Check yourself

1. What does `g` in `m7g` mean, and why choose it?
2. Why does a t3 instance slow down under sustained load?
3. Savings Plan vs Reserved Instance — which is more flexible, and what else
   does a Compute SP cover?
4. How much warning does a Spot interruption give?
5. Cluster vs spread placement group — what is the trade-off?
6. What attack does IMDSv2 mitigate, and how?
7. Is an instance store volume preserved across a stop?
8. Which costs money while an instance is stopped?

---

Previous: [03 — VPC networking](03-vpc-networking.md) ·
Next: [05 — Load balancing and auto scaling](05-load-balancing-and-autoscaling.md)
