# 07 — Block and file storage

Three categories, and the exam tests which one fits.

| | **Block** (EBS) | **File** (EFS, FSx) | **Object** (S3) |
|---|---|---|---|
| Attach to | **One** instance (or a few, with Multi-Attach) | **Many** instances | Anything, over HTTP |
| Protocol | Raw device | NFS / SMB | REST API |
| Use | Boot volumes, databases | Shared app data, home directories | Backups, static assets, data lake |

## EBS

Network-attached block storage. **Lives in one AZ.**

### Volume types

| Type | Media | Max IOPS | Max throughput | Use |
|------|-------|----------|----------------|-----|
| **gp3** | SSD | 16,000 | 1,000 MB/s | **Default choice** — baseline 3,000 IOPS included |
| gp2 | SSD | 16,000 | 250 MB/s | Legacy; IOPS tied to size |
| **io2 Block Express** | SSD | **256,000** | 4,000 MB/s | Critical databases, SAP HANA |
| io1 | SSD | 64,000 | 1,000 MB/s | Older provisioned IOPS |
| **st1** | HDD | 500 | 500 MB/s | Big sequential — logs, data warehouse |
| **sc1** | HDD | 250 | 250 MB/s | Cold, infrequent |

> **Exam tip — gp3 over gp2.** gp3 lets you scale IOPS and throughput
> independently of size and is ~20% cheaper. "Improve performance without
> increasing volume size" → **gp3**.

> **Trap.** HDD types (st1, sc1) **cannot be boot volumes**.

> **Exam tip.** "Highest IOPS", "sub-millisecond latency", "99.999%
> durability", "Multi-Attach" → **io2 Block Express**.

### Key properties

- **Single AZ** — snapshot and restore to move to another AZ
- **Resizable** while in use (size, type, IOPS) via Elastic Volumes
- **Encryption** with KMS — encrypts data at rest, in transit to the instance,
  and all snapshots. Enable **EBS encryption by default** at account level
- **Multi-Attach** — io1/io2 only, up to 16 instances in the same AZ, and the
  filesystem must be cluster-aware

> **Trap.** You cannot directly encrypt an unencrypted volume. The path is:
> snapshot → copy the snapshot **with encryption** → create a volume from it.

### Snapshots

- Stored in S3 (you do not see the bucket), **incremental**
- Can be copied **cross-region** and **cross-account** — the basis of DR
- **Fast Snapshot Restore (FSR)** removes the first-access latency penalty
- **Recycle Bin** protects against accidental deletion
- **Data Lifecycle Manager (DLM)** automates create/retain/delete

> **Exam tip.** "Automate EBS backups with retention" → **DLM** or **AWS
> Backup**. AWS Backup is the broader answer when multiple service types are
> involved.

## Instance store

Physically attached NVMe. **Ephemeral** — data is lost on stop or terminate
(but survives a reboot).

> **Exam tip.** "Maximum IOPS for temporary data / cache / buffer" →
> **instance store**. Anything about persistence rules it out.

## EFS

Managed **NFS** for Linux. Multi-AZ, elastic, pay per GB used.

| Setting | Options |
|---------|---------|
| Storage class | Standard, **One Zone**, IA, Archive |
| Performance mode | General Purpose, **Max I/O** (higher latency, more throughput) |
| Throughput mode | Bursting, **Elastic**, Provisioned |

- Mount targets: one per AZ, each with a security group
- Encryption at rest (KMS) and in transit (TLS)
- **Access Points** enforce a POSIX user and root directory per application
- Lifecycle management moves cold files to IA/Archive automatically

> **Exam tip.** "Shared filesystem for a fleet of Linux instances" or "many EC2
> instances need the same files, concurrently" → **EFS**. Not EBS — EBS cannot
> be shared that way.

> **Trap.** EFS is **Linux/NFS only**. For Windows you need FSx for Windows.

## FSx

Managed third-party and specialised filesystems.

| Variant | Protocol | Use |
|---------|----------|-----|
| **FSx for Windows File Server** | **SMB**, NTFS, AD-integrated | Windows apps, home directories |
| **FSx for Lustre** | Lustre | **HPC, ML**, high throughput; links to S3 |
| **FSx for NetApp ONTAP** | NFS, SMB, iSCSI | Multi-protocol, snapshots, dedup |
| **FSx for OpenZFS** | NFS | ZFS snapshots and clones |

> **Exam tip.**
> - **"Windows" / "SMB" / "Active Directory"** → FSx for **Windows**
> - **"HPC" / "machine learning" / "process S3 data at high throughput"** →
>   FSx for **Lustre**
> - "Migrate NetApp from on-premises" → FSx for **ONTAP**

FSx for Lustre can present an S3 bucket as a filesystem and write results back
— the standard pattern for HPC over a data lake.

## Storage Gateway

Hybrid: on-premises applications use local protocols, data lands in AWS.

| Type | On-prem sees | Backed by |
|------|--------------|-----------|
| **S3 File Gateway** | NFS / SMB | S3 objects |
| **FSx File Gateway** | SMB | FSx for Windows |
| **Volume Gateway — cached** | iSCSI | Primary data in S3, hot cache local |
| **Volume Gateway — stored** | iSCSI | Primary data **local**, async backup to S3 |
| **Tape Gateway** | **VTL** | S3 / Glacier |

> **Exam tip.**
> - "Replace physical tape backup" → **Tape Gateway**
> - "Low-latency access to the full dataset on-premises, backed up to AWS" →
>   **Volume Gateway, stored**
> - "Dataset too large for local disk, keep only hot data locally" →
>   **Volume Gateway, cached**
> - "On-premises app writes files that must appear as S3 objects" →
>   **S3 File Gateway**

## AWS Backup

Centralised, policy-driven backup across EBS, EFS, FSx, RDS, Aurora, DynamoDB,
EC2, Storage Gateway, S3 and more.

- Backup plans with schedules and retention
- **Cross-region and cross-account** copies
- **Backup Vault Lock** — WORM for backups, immutable even to root
- Compliance reporting via AWS Backup Audit Manager

> **Exam tip.** "Centrally manage and audit backups across many services" →
> **AWS Backup**. "Backups must be immutable / protected from a malicious
> admin" → **Vault Lock**.

## Choosing

```
One instance, block device, boot or database?     -> EBS (gp3 default)
Temporary scratch, max IOPS, loss acceptable?     -> Instance store
Many Linux instances sharing files?               -> EFS
Windows / SMB / Active Directory?                 -> FSx for Windows
HPC or ML, extreme throughput, S3-linked?         -> FSx for Lustre
Objects, HTTP, unlimited, cheapest?               -> S3
On-premises app needs AWS storage behind it?      -> Storage Gateway
```

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| Boot volume, general workload | **gp3** |
| 200,000 IOPS database | **io2 Block Express** |
| Large sequential log processing, cheap | **st1** |
| Share files across 50 Linux instances | **EFS** |
| Windows file share with AD | **FSx for Windows** |
| ML training reading from S3 fast | **FSx for Lustre** |
| Encrypt an existing unencrypted volume | Snapshot → copy **encrypted** → restore |
| Move a volume to another AZ | Snapshot → create in target AZ |
| Automate EBS backups | **DLM** or **AWS Backup** |
| Immutable backups | **Backup Vault Lock** |
| Replace tape libraries | **Tape Gateway** |
| Remove first-read latency on restored volumes | **Fast Snapshot Restore** |

## Check yourself

1. Why can't an EBS volume be shared across AZs?
2. gp3 vs gp2 — what changed?
3. Which EBS types cannot be boot volumes?
4. How do you encrypt an existing unencrypted volume?
5. EFS vs FSx for Windows — what decides?
6. What does FSx for Lustre do with S3?
7. Volume Gateway cached vs stored?
8. What does Backup Vault Lock protect against?

---

Previous: [06 — S3](06-s3.md) ·
Next: [08 — Databases](08-databases.md)
