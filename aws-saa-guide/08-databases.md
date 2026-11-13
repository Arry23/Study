# 08 — Databases

A large slice of the Resilient and High-Performing domains. The exam mostly
asks *which database* and *how to make it highly available*.

## Picking a database

| Need | Service |
|------|---------|
| Relational, standard engine | **RDS** |
| Relational, cloud-native, high performance | **Aurora** |
| Key-value, serverless, single-digit ms | **DynamoDB** |
| In-memory cache | **ElastiCache** (Redis / Memcached) |
| Data warehouse, analytics, SQL over TB–PB | **Redshift** |
| Graph | **Neptune** |
| Document / MongoDB-compatible | **DocumentDB** |
| Time series | **Timestream** |
| Wide-column, Cassandra-compatible | **Keyspaces** |
| Ledger, immutable, cryptographically verifiable | **QLDB** |

This workspace uses **RDS PostgreSQL** (every Dockerfile pulls the RDS global
certificate bundle for TLS) and **Neptune** — the SageMaker notebook's
execution role is `AWSNeptuneNotebookRole`, and the project is a graph
database application.

## RDS

Managed relational: MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2.

AWS handles provisioning, patching, backup, recovery. **You do not get OS
access.**

### Multi-AZ vs read replicas

The most-tested distinction in this chapter.

| | **Multi-AZ** | **Read replica** |
|---|---|---|
| Purpose | **Availability** | **Scalability** (reads) |
| Replication | **Synchronous** | **Asynchronous** |
| Standby readable? | **No** (Multi-AZ instance) | **Yes** |
| Failover | **Automatic**, 60–120 s, DNS CNAME swap | Manual promotion |
| Location | Another AZ | Same AZ, cross-AZ, or **cross-region** |
| Cost | 2× instance | Per replica |

> **Exam tip.** "High availability / automatic failover / survive AZ failure"
> → **Multi-AZ**. "Read-heavy, offload reporting queries" → **read replicas**.
> "Disaster recovery in another region" → **cross-region read replica**.

They are not alternatives — production usually has both.

**Multi-AZ DB cluster** (newer) gives two **readable** standbys and faster
failover, blurring the line.

### Backups

| | Automated backups | Manual snapshots |
|---|---|---|
| Retention | 0–35 days | Until you delete |
| PITR | **Yes**, to any second in the window | No |
| On instance deletion | Deleted (unless final snapshot) | Kept |

> **Exam tip.** "Restore to a specific point in time" → automated backups with
> **PITR**. "Keep a backup indefinitely / copy to another account" →
> **manual snapshot**.

Restoring always creates a **new instance** with a new endpoint.

### Security

- **Encryption at rest** with KMS — must be enabled **at creation**
- To encrypt an existing unencrypted instance: snapshot → copy encrypted →
  restore
- **In transit**: TLS. This workspace bakes the RDS CA bundle into every
  image:

```dockerfile
RUN wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

- **IAM database authentication** — connect with an IAM token instead of a
  password (MySQL/PostgreSQL)
- Put RDS in **private subnets**; control access with security groups

> **Exam tip.** "Avoid storing database passwords" → **IAM database
> authentication**, or **Secrets Manager with automatic rotation**.

### RDS Proxy

Connection pooling in front of RDS.

> **Exam tip.** "Lambda exhausts database connections" → **RDS Proxy**. It
> also improves failover time and can enforce IAM auth.

## Aurora

AWS-built MySQL- and PostgreSQL-compatible engine.

- Storage auto-grows to **128 TB**, **6 copies across 3 AZs**
- Up to **15 read replicas**, ~10 ms replica lag
- Failover typically **under 30 seconds**
- **Backtrack** — rewind the cluster in place without a restore

**Endpoints:**

| Endpoint | Points to |
|----------|-----------|
| **Writer** | Current primary |
| **Reader** | Load-balanced across replicas |
| **Custom** | A chosen subset |

**Variants:**

| | Use |
|---|---|
| **Aurora Serverless v2** | Variable or unpredictable load; scales in fine increments |
| **Aurora Global Database** | Cross-region, < 1 s replication, < 1 min RTO |
| **Aurora Multi-Master** | Multiple writers |

> **Exam tip.** "Cross-region DR for a relational database with the lowest
> RPO/RTO" → **Aurora Global Database**. "Unpredictable, spiky, intermittent
> database load" → **Aurora Serverless v2**.

## DynamoDB

Managed NoSQL key-value and document store. Single-digit millisecond latency
at any scale.

### Data model

- **Partition key** alone, or **partition key + sort key**
- Items up to **400 KB**
- Schemaless apart from the key

### Indexes

| | **LSI** | **GSI** |
|---|---|---|
| Key | Same partition key, different sort key | **Any** attributes |
| Created | **At table creation only** | Any time |
| Consistency | Strong or eventual | **Eventual only** |
| Capacity | Shares the table's | **Its own** |

> **Trap.** An LSI cannot be added later. If a question involves a new access
> pattern on an existing table, the answer is a **GSI**.

### Capacity modes

| Mode | Use |
|------|-----|
| **On-demand** | Unpredictable, spiky, new applications |
| **Provisioned** | Predictable; cheaper; supports auto scaling and reserved capacity |

> **Exam tip.** `ProvisionedThroughputExceededException` → enable auto scaling,
> switch to on-demand, or fix a **hot partition** by improving key
> distribution.

### Features

| Feature | Purpose |
|---------|---------|
| **DAX** | In-memory cache, **microsecond** reads, DynamoDB-specific |
| **Streams** | Change data capture → Lambda, Kinesis |
| **Global Tables** | Multi-region, multi-active replication |
| **TTL** | Auto-delete expired items, free |
| **PITR** | Restore to any second in the last 35 days |
| **Transactions** | ACID across multiple items/tables |

> **Exam tip.** "DynamoDB is fast but we need microseconds" → **DAX**. Not
> ElastiCache — DAX is purpose-built and write-through.

> **Exam tip.** "React to every item change" → **DynamoDB Streams + Lambda**.
> "Multi-region active-active" → **Global Tables**.

## ElastiCache

| | **Redis** | **Memcached** |
|---|---|---|
| Data structures | Rich (lists, sets, sorted sets) | Strings only |
| Persistence | **Yes** | No |
| Replication / HA | **Yes**, Multi-AZ failover | No |
| Multi-threaded | No (mostly) | **Yes** |
| Use | Leaderboards, sessions, pub/sub, HA cache | Simple, horizontally scaled cache |

> **Exam tip.** "Cache must survive a node failure / needs replication" →
> **Redis**. "Simple, pure cache, scale out with more nodes" → **Memcached**.

Caching strategies:

- **Lazy loading / cache-aside** — read miss populates the cache
- **Write-through** — write updates cache and database together
- **TTL** — bound staleness

> **Exam tip.** "Reduce read load on RDS" → ElastiCache, or read replicas.
> Cache for repeated identical queries; replicas for general read scaling.

**ElastiCache Serverless** removes node management.

## Redshift

Petabyte-scale columnar data warehouse for OLAP.

- **Redshift Spectrum** queries S3 directly without loading
- **RA3 nodes** separate compute from managed storage
- **Concurrency Scaling** adds capacity for query bursts
- **Redshift Serverless** for intermittent analytics

> **Trap.** Redshift is **not** for OLTP. "Thousands of small transactional
> writes" → RDS/Aurora/DynamoDB, never Redshift.

## Neptune

Managed **graph** database — Gremlin, openCypher, SPARQL.

> **Exam tip.** "Highly connected data", "social network", "fraud ring
> detection", "knowledge graph", "recommendations based on relationships" →
> **Neptune**.

This workspace is a knowledge-graph application: the SageMaker notebook uses
`AWSNeptuneNotebookRole`, and the codebase is full of SPARQL and ontology
handling.

## OLTP vs OLAP

| | OLTP | OLAP |
|---|------|------|
| Pattern | Many small reads/writes | Few large analytical scans |
| Services | RDS, Aurora, DynamoDB | Redshift, Athena, EMR |

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| Automatic failover for RDS | **Multi-AZ** |
| Offload reporting reads | **Read replicas** |
| Cross-region relational DR, lowest RTO | **Aurora Global Database** |
| Unpredictable relational load | **Aurora Serverless v2** |
| Lambda exhausting DB connections | **RDS Proxy** |
| Restore to 14:32 yesterday | Automated backups + **PITR** |
| Keep a backup after deleting the DB | **Manual snapshot** |
| Microsecond reads on DynamoDB | **DAX** |
| New access pattern on an existing table | **GSI** |
| React to item-level changes | **DynamoDB Streams** |
| Multi-region active-active NoSQL | **Global Tables** |
| Session store for a stateless web tier | **ElastiCache Redis** or DynamoDB |
| Cache that survives node failure | **Redis** |
| SQL over petabytes for BI | **Redshift** |
| Query S3 with SQL, no loading | **Athena** or Redshift Spectrum |
| Relationship-heavy queries | **Neptune** |
| Immutable, verifiable ledger | **QLDB** |
| Avoid DB passwords in code | IAM DB auth, or **Secrets Manager** rotation |

## Check yourself

1. Multi-AZ vs read replica — purpose, replication type, readable?
2. How do you encrypt an existing unencrypted RDS instance?
3. What does Aurora Backtrack do that a restore does not?
4. LSI vs GSI — which can be added later?
5. When would you choose DAX over ElastiCache?
6. Redis vs Memcached — two deciding factors.
7. Why is Redshift wrong for OLTP?
8. What problem does RDS Proxy solve?

---

Previous: [07 — Block and file storage](07-block-and-file-storage.md) ·
Next: [09 — Serverless and containers](09-serverless-and-containers.md)
