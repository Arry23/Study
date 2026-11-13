# 06 — Scaling data

One machine eventually runs out. This chapter is how you get past that, and
what it costs you.

## The escalation ladder

```
1. Indexes and query tuning      ← free, do this first
2. Caching                       ← chapter 04
3. Vertical scaling              ← bigger box, simple
4. Read replicas                 ← scales READS only
5. Functional partitioning       ← split by feature/table
6. Horizontal sharding           ← scales writes, expensive in complexity
```

> **Interview tip.** Walk down this list. Jumping to sharding when read
> replicas would do is the most common over-engineering mistake, and
> interviewers watch for it.

## Replication

One primary, N replicas.

```
         writes
           ↓
      ┌─ Primary ─┐
      │           │  replication
      ↓           ↓
  Replica-1   Replica-2
      ↑           ↑
         reads
```

### Synchronous vs asynchronous

| | Synchronous | Asynchronous |
|---|---|---|
| Primary waits for replica? | **Yes** | No |
| Write latency | Higher | **Low** |
| Data loss on failover | **None** | Possible (replication lag) |
| Availability | Replica failure can block writes | Unaffected |

**Semi-synchronous** — wait for *one* replica to acknowledge. The usual
compromise.

> **Trade-off.** This is a direct durability-vs-latency choice. Financial
> systems take synchronous; social feeds take asynchronous.

### Replication lag

The practical problem with read replicas:

```
T1: user posts a comment        → primary
T2: page reloads, reads replica → comment not there yet
```

**Fixes:**

| Technique | How |
|-----------|-----|
| **Read-your-writes** | Route that user's reads to the primary for N seconds after a write |
| **Monotonic reads** | Pin a user to one replica so time never goes backwards |
| Causal consistency | Track a version/LSN; read a replica at least that fresh |
| Write-through cache | Serve the user's own write from cache |

> **Interview tip.** "Read-your-writes consistency" is the exact term. Using it
> when a question mentions a user not seeing their own post signals real
> experience.

### Topologies

| Topology | Writes | Notes |
|----------|--------|-------|
| **Single leader** | One node | Simple, no write conflicts. **Default** |
| **Multi-leader** | Several | Multi-region writes, but **conflicts** |
| **Leaderless** | Any node | Dynamo/Cassandra style, quorums |

Multi-leader and leaderless require **conflict resolution** — last-write-wins
(lossy), version vectors, or CRDTs. Covered in
[chapter 07](07-consistency-and-consensus.md).

## Functional partitioning

Split by feature before splitting by key.

```
users_db      ← users, profiles, auth
orders_db     ← orders, line items
analytics_db  ← events
```

Simple, and often enough. You lose cross-domain joins and cross-domain
transactions — which is also the first step towards microservices
([chapter 13](13-architecture-styles.md)).

## Sharding

Splitting one logical table across many machines by a **shard key**.

### Strategies

**Range-based** — shard by key ranges.

```
shard-0: user_id 0–1,000,000
shard-1: user_id 1,000,001–2,000,000
```

Good for range scans. **Prone to hotspots** — sequential IDs all land on the
newest shard.

**Hash-based** — `shard = hash(key) % N`.

Even distribution, no range queries, and **resharding moves nearly everything**
when N changes.

**Directory-based** — a lookup service maps key → shard.

Flexible, supports rebalancing, but the directory is a dependency and a
potential SPOF.

**Geographic** — shard by region. Good for latency and data residency; uneven
if populations differ.

### Consistent hashing

Solves the "`% N` reshuffles everything" problem.

```
Hash ring 0 ──────────────────── 2^32
     │      │        │        │
   node-A  node-B  node-C   node-A'   ← virtual nodes
```

A key hashes onto the ring and belongs to the next node clockwise. Adding or
removing a node moves only **K/N** keys instead of nearly all of them.

**Virtual nodes** — each physical node occupies many ring positions, which
evens out distribution and makes rebalancing smoother when a node leaves.

> **Interview tip.** Name consistent hashing whenever you add or remove cache
> or storage nodes. State the benefit precisely: "only K/N keys move, instead
> of almost all of them with modulo hashing."

### Choosing a shard key

The single most important decision, and effectively irreversible.

**A good shard key:**

1. **High cardinality** — many distinct values
2. **Even distribution** — no hot values
3. **Matches query patterns** — most queries include it
4. **Stable** — never changes for a row

| Shard key | Verdict |
|-----------|---------|
| `user_id` (hashed) | Usually good — even, stable, in most queries |
| `created_at` | **Bad** — all writes hit the newest shard |
| `country` | **Bad** — skewed; India/US shards overwhelm Malta |
| `tenant_id` | Good for SaaS, but watch for one enormous tenant |
| `order_id` (random) | Even, but customer queries fan out to all shards |

> **Trap — the celebrity problem.** Sharding by `user_id` is even *per user*,
> but one account with 100 M followers makes its shard hot. Mitigate by
> splitting that user's data further (composite key with a bucket suffix) or
> handling such accounts on a separate path.

### What sharding costs you

| Lost | Consequence |
|------|-------------|
| **Cross-shard joins** | Do them in the application, or denormalise |
| **Cross-shard transactions** | Need 2PC or sagas ([ch. 08](08-messaging-and-streaming.md)) |
| **Global secondary indexes** | Must be maintained separately |
| **`AUTO_INCREMENT`** | Need UUIDs, Snowflake IDs or a ticket server |
| **Simple operations** | Backups, migrations and resharding all get harder |
| **Aggregate queries** | `COUNT(*)` must scatter-gather |

> **Interview tip.** Listing these costs unprompted is the difference between
> "I know the word sharding" and "I have thought about sharding."

### Distributed ID generation

Once sharded, you cannot use a single auto-increment.

| Approach | Notes |
|----------|-------|
| **UUIDv4** | No coordination; **random, so bad for B-tree locality** |
| **UUIDv7 / ULID** | Time-ordered — good index locality. **Modern default** |
| **Snowflake** | `timestamp | machine_id | sequence` — 64-bit, sortable |
| Ticket server | Central counter — simple, a SPOF |
| Range allocation | Each node pre-allocates a block |

> **Interview tip.** Snowflake IDs are worth being able to describe:
> 41 bits of timestamp, 10 bits of machine ID, 12 bits of sequence. Sortable
> by time, generated without coordination, 64 bits.

## Resharding

The operation everyone forgets to plan for.

**Strategies:**

- **Pre-split** — start with 1,024 logical shards on 8 physical nodes; move
  logical shards as you grow. **Much easier than splitting later.**
- **Consistent hashing** — add nodes, move K/N keys
- **Double-write and backfill** — write to old and new, backfill, verify, cut
  over, remove the old path

> **Interview tip.** "Start with more logical shards than physical nodes" is a
> strong, practical answer to "how do you grow?"

## Multi-region

| Pattern | Writes | Consistency | Use |
|---------|--------|-------------|-----|
| Single-region + CDN | One region | Strong | Simplest; latency for distant users |
| Read replicas abroad | One region | Eventual reads | Global reads, local writes |
| **Active-passive** | One region, failover | Strong | DR with RTO in minutes |
| **Active-active** | All regions | **Eventual + conflicts** | Lowest latency, highest complexity |

> **Trade-off.** Active-active sounds attractive and is genuinely hard: you
> need conflict resolution, you cannot have global transactions cheaply, and
> cross-region latency (~150 ms) makes synchronous coordination unacceptable.
> Only propose it when the requirement explicitly demands it, and say what it
> costs.

## Decision table

| Problem | Answer |
|---------|--------|
| Reads are the bottleneck | **Read replicas** (+ cache first) |
| User does not see their own write | **Read-your-writes** routing |
| Writes exceed one primary | **Shard** |
| Adding cache nodes invalidates everything | **Consistent hashing** |
| Sequential IDs create a hot shard | Hash the key, or **Snowflake/ULID** |
| One tenant is 40% of traffic | Dedicated shard, or sub-shard that tenant |
| Need to grow shard count later | **Pre-split into logical shards** |
| Global users, local latency | Regional replicas; active-active only if required |
| Cross-shard transaction | **Saga**, or redesign to avoid it |

## Practice

1. Reads are at 95% of capacity, writes at 20%. What do you do, in order?
2. Why is `created_at` a poor shard key? Give a concrete failure.
3. Modulo hashing with 10 cache nodes; you add an 11th. What fraction of keys
   move? With consistent hashing?
4. A user posts a comment and does not see it on refresh. Diagnose and give two
   fixes.
5. Design an ID scheme for a sharded system needing time-sortable, 64-bit,
   coordination-free IDs.
6. You shard by `user_id`. One account has 80 M followers and its shard is hot.
   Options?
7. What exactly do you lose when you shard? List five things.
8. When is active-active multi-region justified, and what must you build?

<details><summary>Answers</summary>

1. Cache hot reads first (cheapest), then add **read replicas**, routing reads
   away from the primary. Sharding is unnecessary — writes are fine.
2. All new rows have the newest timestamp, so **every write lands on the last
   shard** while others idle. Ingestion is limited to one shard's throughput.
3. Modulo: roughly **10/11 of keys move** (any key whose `hash % N` changes).
   Consistent hashing: about **1/11** — only keys in the new node's arc.
4. Replication lag — the read hit a replica that had not caught up. Fix with
   **read-your-writes** (route that user to the primary briefly) or serve their
   own write from cache/local state.
5. **Snowflake**: 41-bit millisecond timestamp, 10-bit machine ID, 12-bit
   per-ms sequence. Or **ULID/UUIDv7** if you prefer a standard — both are
   time-ordered, so index locality is good.
6. Sub-shard that user's data with a composite key (`user_id:bucket`),
   aggressively cache their timeline, move them to a dedicated shard, or use a
   different fan-out strategy for celebrities (pull instead of push — see
   [ch. 16](16-hld-case-studies-core.md)).
7. Cross-shard joins, cross-shard transactions, global secondary indexes,
   auto-increment IDs, easy aggregates/`COUNT(*)`, and simple backups and
   schema migrations.
8. When users are globally distributed **and** write latency matters **and**
   regional failure must not cause downtime. You must build conflict
   resolution (CRDTs/version vectors), accept eventual consistency, design
   around the absence of global transactions, and test failover regularly.
</details>

---

Previous: [05 — Databases](05-databases.md) ·
Next: [07 — Consistency and consensus](07-consistency-and-consensus.md)
