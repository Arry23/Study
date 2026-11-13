# 16 — Migration and transfer

Questions here are usually "move X TB from on-premises to AWS, given Y
bandwidth and Z time" — a calculation — or "which migration service".

## The bandwidth calculation

Learn to do this quickly.

```
Time (seconds) = Size (bits) / Bandwidth (bits per second)
```

A rough table at ~80% utilisation:

| Data | 100 Mbps | 1 Gbps | 10 Gbps |
|------|----------|--------|---------|
| 1 TB | ~1 day | ~2.5 h | ~15 min |
| 10 TB | ~12 days | ~1 day | ~2.5 h |
| 100 TB | ~4 months | ~12 days | ~1 day |
| 1 PB | ~3 years | ~4 months | ~12 days |

> **Exam tip — the rule of thumb.** If the transfer would take **more than
> about a week** over the available link, the answer is a **Snow device**.
> Otherwise use the network (DataSync / Direct Connect).

## Snow family

Physical devices shipped to you, filled, and shipped back.

| Device | Capacity | Use |
|--------|----------|-----|
| **Snowcone** | 8–14 TB | Small, rugged, edge; can ship or use DataSync |
| **Snowball Edge Storage Optimized** | ~80 TB usable | **Bulk data transfer** |
| **Snowball Edge Compute Optimized** | ~42 TB + compute | Edge processing, GPU option |
| **Snowmobile** | **Up to 100 PB** | Exabyte-scale datacentre evacuation |

All encrypted with KMS, tracked with SNS notifications.

> **Exam tip.**
> - 10–80 TB, limited bandwidth → **Snowball Edge**
> - "Petabytes" / "entire data centre" → **Snowmobile**
> - "Edge location, intermittent connectivity, process locally then ship" →
>   **Snowball Edge Compute** or **Snowcone**

## DataSync

Agent-based service for moving **files and objects** at scale.

| From | To |
|------|-----|
| NFS, SMB, HDFS, self-managed object storage | S3, EFS, FSx |
| S3 | Another region/account, EFS, FSx |

- Up to **10 Gbps** per agent, with built-in encryption and validation
- **Incremental** transfers and scheduling
- Preserves metadata and permissions

> **Exam tip.** "Ongoing or scheduled replication of file data to AWS" →
> **DataSync**. "One-time move of a huge dataset with poor bandwidth" →
> **Snow**. They are also combined: DataSync can run **on a Snowcone**.

> **Trap.** DataSync is for **file/object transfer**. For **database**
> migration, it is DMS.

## Transfer Family

Managed **SFTP, FTPS, FTP, AS2** endpoints backed by S3 or EFS.

> **Exam tip.** "Partners upload via SFTP and we cannot change their process"
> → **AWS Transfer Family**. The keyword is the *protocol*.

## Storage Gateway

Hybrid storage — covered in [chapter 07](07-block-and-file-storage.md).

> **Exam tip.** Use Storage Gateway when on-premises systems must **keep
> working** with local protocols while data lands in AWS. Use DataSync when
> you are **moving** data. Gateway = ongoing hybrid access; DataSync =
> transfer.

## Database Migration Service (DMS)

Migrates databases with **minimal downtime**; the source stays available.

| Mode | Use |
|------|-----|
| **Full load** | One-time copy |
| **Full load + CDC** | Copy, then replicate ongoing changes |
| **CDC only** | Continuous replication |

**Homogeneous** (Oracle → Oracle) needs DMS alone. **Heterogeneous**
(Oracle → PostgreSQL) also needs the **Schema Conversion Tool (SCT)** to
convert schema and code.

> **Exam tip.** "Migrate from Oracle to Aurora PostgreSQL with minimal
> downtime" → **SCT** (schema) + **DMS with CDC** (data). Both halves.

DMS can also target S3, Kinesis, OpenSearch and Redshift — useful for building
data lakes.

## Application migration

| Service | Purpose |
|---------|---------|
| **Application Migration Service (MGN)** | **Lift-and-shift** servers into EC2 via block replication |
| **Application Discovery Service** | Inventory and dependency mapping before migrating |
| **Migration Hub** | Track progress across services and accounts |
| **Elastic Disaster Recovery (DRS)** | Same replication tech, for DR rather than migration |

> **Exam tip.** "Rehost / lift-and-shift physical or VMware servers with
> minimal changes" → **MGN**. "Understand dependencies before we move" →
> **Application Discovery Service**.

## The 7 Rs

| Strategy | Meaning |
|----------|---------|
| **Rehost** | Lift and shift (MGN) |
| **Replatform** | Lift and reshape — e.g. self-managed MySQL → RDS |
| **Repurchase** | Move to SaaS |
| **Refactor** | Re-architect — monolith → serverless |
| **Retire** | Turn it off |
| **Retain** | Leave it where it is |
| **Relocate** | Move hypervisor-level, e.g. VMware Cloud on AWS |

> **Exam tip.** "Fastest migration with least change" → **rehost**.
> "Reduce operational overhead by moving to a managed database" →
> **replatform**.

## Direct Connect for migration

| | Setup time | Use |
|---|---|---|
| **Site-to-Site VPN** | Minutes | Immediate need, or DX backup |
| **Direct Connect** | **Weeks to months** | Sustained high bandwidth |

> **Trap.** A question with an urgent deadline and no existing Direct Connect
> rules DX out — provisioning takes too long. Use VPN, DataSync or Snow.

## Choosing

```
Databases, minimal downtime?            -> DMS (+ SCT if heterogeneous)
Whole servers, lift and shift?          -> MGN
File/object data, ongoing or scheduled? -> DataSync
Huge dataset, poor bandwidth?           -> Snow family
Partners need SFTP/FTPS?                -> Transfer Family
On-prem apps must keep local access?    -> Storage Gateway
Sustained hybrid bandwidth?             -> Direct Connect
Need connectivity today?                -> Site-to-Site VPN
```

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| 500 TB, 100 Mbps link | **Snowball Edge** (network would take months) |
| 5 TB, 1 Gbps, one-off | **DataSync** over the network |
| Nightly sync of an NFS share to S3 | **DataSync**, scheduled |
| Oracle → Aurora PostgreSQL, minimal downtime | **SCT + DMS with CDC** |
| MySQL → RDS MySQL, minimal downtime | **DMS full load + CDC** |
| 200 VMware servers into EC2 | **MGN** |
| Map dependencies before migrating | **Application Discovery Service** |
| Partners upload over SFTP | **Transfer Family** |
| On-prem app needs local file access, data in AWS | **Storage Gateway** |
| Replace tape backups | **Tape Gateway** |
| 100 PB datacentre evacuation | **Snowmobile** |
| Edge site, process then ship | **Snowball Edge Compute** |
| Hybrid link needed this week | **Site-to-Site VPN** |
| Track a multi-service migration | **Migration Hub** |

## Check yourself

1. What is the rough threshold for choosing Snow over the network?
2. DataSync vs Storage Gateway — what distinguishes them?
3. What does SCT do that DMS does not?
4. Which DMS mode keeps the source available during migration?
5. MGN vs DRS — same technology, different what?
6. Why might Direct Connect be the wrong answer despite being the fastest link?
7. Name the 7 Rs.
8. Which service handles partners who insist on SFTP?

---

Previous: [15 — Cost optimization](15-cost-optimization.md) ·
Next: [17 — Analytics](17-analytics.md)
