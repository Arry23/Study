# 05 — Databases

Choosing the data store is usually the most consequential decision in a design,
and the hardest to reverse.

## The families

| Type | Model | Examples | Good at |
|------|-------|----------|---------|
| **Relational** | Tables, rows, joins | PostgreSQL, MySQL | Transactions, joins, flexible queries |
| **Key-value** | `key → blob` | Redis, DynamoDB | O(1) lookup, massive scale |
| **Document** | JSON documents | MongoDB, DocumentDB | Flexible schema, nested data |
| **Wide-column** | Row key → column families | Cassandra, HBase, Bigtable | Huge write volume, time series |
| **Graph** | Nodes and edges | Neo4j, Neptune | Relationship traversal |
| **Time series** | Timestamped points | InfluxDB, TimescaleDB | Metrics, IoT |
| **Search** | Inverted index | Elasticsearch, OpenSearch | Full-text, relevance |
| **Columnar / OLAP** | Column-oriented | ClickHouse, Redshift, BigQuery | Analytical scans |

> **Interview tip.** The honest default is **PostgreSQL**. It does JSON,
> full-text search, geospatial and arrays. Most "we need NoSQL" conclusions are
> premature. Say that, then justify moving away from it when a specific access
> pattern demands it.

## SQL vs NoSQL

| | SQL | NoSQL |
|---|---|---|
| Schema | Fixed, enforced | Flexible |
| Joins | **Yes** | Usually not — denormalise |
| Transactions | **ACID across rows/tables** | Often single-item only |
| Scaling | Vertical first; sharding is manual work | **Horizontal by design** |
| Query flexibility | **Ad-hoc, any column** | Designed around known access patterns |
| Consistency | Strong by default | Often tunable/eventual |

> **The real distinction:** SQL lets you design the **data** and figure out
> queries later. NoSQL requires you to know the **queries** and design the data
> around them. Choosing NoSQL without knowing your access patterns is how
> teams end up rewriting.

### When NoSQL genuinely wins

- Write volume beyond a single primary (Cassandra, hundreds of thousands/s)
- Access is always by a single key (DynamoDB)
- Schema varies per record and genuinely cannot be normalised
- Data is naturally a graph and queries are deep traversals
- Scale is global and multi-region active-active is required

## ACID

| Property | Meaning |
|----------|---------|
| **Atomicity** | All of a transaction happens, or none |
| **Consistency** | Constraints hold before and after |
| **Isolation** | Concurrent transactions do not corrupt each other |
| **Durability** | Committed data survives a crash |

> **Trap.** The "C" in ACID (constraints) is **not** the "C" in CAP
> (linearizability). Interviewers notice if you conflate them.

## Isolation levels

The classic table. Know the anomalies each level permits.

| Level | Dirty read | Non-repeatable read | Phantom read |
|-------|------------|--------------------|--------------|
| Read Uncommitted | **possible** | possible | possible |
| **Read Committed** | no | possible | possible |
| **Repeatable Read** | no | no | possible* |
| **Serializable** | no | no | no |

\* PostgreSQL's Repeatable Read (snapshot isolation) also prevents phantoms.

**The anomalies:**

- **Dirty read** — you read another transaction's uncommitted write
- **Non-repeatable read** — you read the same row twice, with different values
- **Phantom read** — you run the same query twice and get different *rows*
- **Write skew** — two transactions each read, then write, producing a state
  neither would allow alone (only prevented by Serializable)

> **Interview tip.** Default is **Read Committed** in PostgreSQL/Oracle,
> **Repeatable Read** in MySQL/InnoDB. Knowing the defaults differ is a good
> signal.

**Example of write skew:** two doctors are on call; each checks "is at least
one other on call?", sees yes, and both go off call. Neither transaction broke
a rule alone. Only Serializable prevents this.

## Indexes

An index trades **write speed and storage** for **read speed**.

```sql
CREATE INDEX idx_users_email ON users(email);
```

| Type | Use |
|------|-----|
| **B-tree** | Default. Equality, ranges, ordering, prefix |
| **Hash** | Equality only, no ranges |
| **Composite** | Multiple columns — **order matters** |
| **Covering** | Includes all selected columns — no table lookup |
| **Partial** | `WHERE status = 'active'` — smaller, faster |
| **Full-text / GIN** | Text search, JSON containment |
| **Geospatial (R-tree)** | Bounding-box and nearest queries |

### Composite index ordering

```sql
CREATE INDEX idx ON orders(customer_id, created_at);
```

Serves `WHERE customer_id = ?`, and `WHERE customer_id = ? AND created_at > ?`.
Does **not** efficiently serve `WHERE created_at > ?` alone.

> **Interview tip.** This is the **leftmost prefix rule**. Name it. Put the
> **equality** column first and the **range** column second.

### When indexes hurt

- Every write must update every index
- Low-cardinality columns (a `gender` index is usually useless)
- Too many indexes bloat storage and slow writes
- The planner may ignore an index if it estimates a scan is cheaper

> **Trap.** Indexing a column used in a function (`WHERE LOWER(email) = ?`)
> does not help unless you create a **functional index** on `LOWER(email)`.

## B-tree vs LSM-tree

The storage-engine question, and a strong differentiator in interviews.

| | **B-tree** | **LSM-tree** |
|---|---|---|
| Used by | PostgreSQL, MySQL/InnoDB | Cassandra, RocksDB, LevelDB, HBase |
| Writes | **In place** — random I/O | **Append-only** — sequential |
| Write throughput | Lower | **Much higher** |
| Reads | Predictable, one path | May check several SSTables + Bloom filters |
| Space | Fragmentation | **Compression-friendly**, but compaction overhead |
| Write amplification | Lower | Higher (compaction) |

> **Interview tip.** "Write-heavy workload" → LSM (Cassandra, RocksDB).
> "Read-heavy with complex queries" → B-tree (PostgreSQL). Explaining that
> LSM converts random writes into sequential ones is the core insight.

## Normalisation vs denormalisation

| | Normalised | Denormalised |
|---|---|---|
| Duplication | None | Deliberate |
| Writes | Simple, one place | Must update several places |
| Reads | **Joins** | **Single lookup** |
| Consistency | Enforced by schema | Application's problem |

> **Trade-off.** Normalise until a read path is too slow, then denormalise that
> path deliberately — not the whole schema. In NoSQL, denormalisation is the
> default because joins do not exist.

## Concurrency control

**Pessimistic locking** — lock the row, block others.

```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
```

**Optimistic locking** — read a version, check it on write.

```sql
UPDATE accounts
SET balance = 50, version = version + 1
WHERE id = 1 AND version = 7;     -- 0 rows updated means someone beat you
```

> **Interview tip.** Optimistic for low-contention (most web apps); pessimistic
> for high-contention short transactions (inventory decrement, seat booking).
> Optimistic avoids holding locks across a user's thinking time — which is why
> it is right for "edit and save" flows.

**MVCC** — readers see a consistent snapshot and never block writers. How
PostgreSQL and InnoDB achieve high concurrency.

## Connection pooling

Each connection costs memory on the database (PostgreSQL: a process, several
MB). A pool bounds this.

> **Trap.** 100 app instances × a pool of 20 = 2,000 connections, which will
> flatten PostgreSQL. Use an external pooler (**PgBouncer**, RDS Proxy) in
> transaction mode. This is a very common real-world failure and a good thing
> to raise unprompted.

Size pools with Little's Law ([chapter 01](01-fundamentals-and-estimation.md)).

## Decision table

| Requirement | Choose |
|-------------|--------|
| Default, unsure | **PostgreSQL** |
| Financial transactions, joins, reporting | **Relational** |
| Simple key lookups, enormous scale | **DynamoDB / Redis** |
| Hundreds of thousands of writes/s, time series | **Cassandra** |
| Flexible nested documents | **MongoDB / Postgres JSONB** |
| Relationship traversal | **Graph (Neo4j/Neptune)** |
| Full-text relevance search | **Elasticsearch** |
| Analytical scans over billions of rows | **Columnar (ClickHouse/Redshift)** |
| Metrics and monitoring | **Time series** |
| Session store, leaderboard, lock | **Redis** |

> **Interview tip.** Real systems use several — **polyglot persistence**.
> Postgres as the source of truth, Redis for cache and sessions, Elasticsearch
> for search, S3 for blobs, Kafka as the log. Saying this shows maturity; just
> be ready to explain how they stay in sync
> ([chapter 08](08-messaging-and-streaming.md)).

## Practice

1. Why is "we need NoSQL for scale" usually premature?
2. Composite index `(a, b)` — which queries does it serve efficiently?
3. Explain write skew, and which isolation level prevents it.
4. Write-heavy ingestion of 200k events/s. B-tree or LSM? Why?
5. Two users edit the same document. Optimistic or pessimistic? Why?
6. 150 app instances, pool size 20, PostgreSQL falls over. Diagnose and fix.
7. When does an index make things worse?
8. Design the table and index for "last 20 orders for a customer, newest
   first".

<details><summary>Answers</summary>

1. A single well-tuned Postgres handles thousands of writes/s and terabytes.
   Most teams hit **bad queries, missing indexes or no caching** long before a
   genuine scale limit — and NoSQL costs you joins, transactions and ad-hoc
   querying.
2. `WHERE a = ?`, `WHERE a = ? AND b = ?`, `WHERE a = ? ORDER BY b`. **Not**
   `WHERE b = ?` alone (leftmost prefix rule).
3. Two transactions read overlapping data, then each write based on what they
   read, producing a state neither would permit alone. Only **Serializable**
   prevents it.
4. **LSM.** Appends are sequential, giving far higher write throughput; reads
   cost a little more via Bloom filters and multiple SSTables, which is the
   right trade for ingestion.
5. **Optimistic** with a version column. Pessimistic would hold a lock while a
   human types, blocking everyone; optimistic lets both proceed and detects
   the conflict at save time.
6. 3,000 connections exceeds what Postgres can handle (process-per-connection).
   Add **PgBouncer** in transaction mode, cut per-instance pool size, and size
   from Little's Law rather than by habit.
7. On write-heavy tables (every insert updates every index), on low-cardinality
   columns, when there are so many that storage and write cost dominate, and
   when queries wrap the column in a function without a matching functional
   index.
8. `orders(customer_id, created_at DESC, …)` with
   `CREATE INDEX ON orders(customer_id, created_at DESC)`; query with
   `WHERE customer_id = ? ORDER BY created_at DESC LIMIT 20`. Ideally make it
   **covering** by including the selected columns so no heap lookup is needed.
</details>

---

Previous: [04 — Caching](04-caching.md) ·
Next: [06 — Scaling data](06-scaling-data.md)
