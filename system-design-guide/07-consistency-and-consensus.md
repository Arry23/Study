# 07 — Consistency and consensus

The theoretical core of distributed systems, and the chapter that most improves
interview answers.

## CAP

> In the presence of a network **P**artition, you must choose between
> **C**onsistency and **A**vailability.

```
        Partition happens
               │
      ┌────────┴────────┐
   CP: refuse          AP: serve
   requests to         possibly stale
   stay correct        data
```

| Choice | Behaviour on partition | Examples |
|--------|------------------------|----------|
| **CP** | Return an error rather than stale data | HBase, MongoDB (default), ZooKeeper, etcd |
| **AP** | Serve possibly stale data | Cassandra, DynamoDB, Riak |

> **Trap — the most common CAP mistake.** "We chose CA" is not a choice.
> Partitions are **not optional** — networks fail. Every distributed system is
> CP or AP. A single-node database is "CA" only in the trivial sense that it
> has no network to partition.

> **Trap.** CAP is about behaviour **during a partition**, not normal
> operation. Most of the time you have both C and A.

## PACELC — the better model

> **If P**artition: choose **A**vailability or **C**onsistency.
> **E**lse (normal operation): choose **L**atency or **C**onsistency.

| System | PACELC |
|--------|--------|
| DynamoDB, Cassandra | **PA/EL** — available on partition, low latency otherwise |
| HBase, BigTable | **PC/EC** — consistent always |
| MongoDB | PC/EC (configurable) |
| PostgreSQL (single node) | — (not distributed) |

> **Interview tip.** PACELC is the more useful model because the **EL/EC**
> trade-off applies every day, not just during rare partitions. Synchronous
> replication costs latency on every write; asynchronous costs consistency.
> Mentioning PACELC distinguishes a strong answer.

## Consistency models

From strongest to weakest:

| Model | Guarantee |
|-------|-----------|
| **Linearizability** (strong) | Operations appear to happen instantly, in real-time order. One copy illusion |
| **Sequential** | All nodes see the same order, not necessarily real time |
| **Causal** | Causally related operations are seen in order; concurrent ones may differ |
| **Read-your-writes** | You always see your own writes |
| **Monotonic reads** | You never see time go backwards |
| **Eventual** | If writes stop, replicas converge eventually |

> **Interview tip.** "Eventual consistency" is often used loosely. The useful
> middle ground is **causal consistency** plus **read-your-writes** — it feels
> correct to users without the cost of linearizability.

### Session guarantees — what users actually notice

| Guarantee | Without it, the user sees |
|-----------|---------------------------|
| **Read-your-writes** | Their own post missing after refresh |
| **Monotonic reads** | A comment appears, then disappears |
| **Monotonic writes** | Their edits applied out of order |
| **Writes-follow-reads** | A reply appearing before the thing it replies to |

These are cheap to provide (route to one replica, or track a version) and fix
most perceived "eventual consistency" bugs.

## Quorums

With N replicas, W write acknowledgements and R read responses:

$$ W + R > N \implies \text{read sees the latest write} $$

| Config | N | W | R | Property |
|--------|---|---|---|----------|
| Strong-ish | 3 | 2 | 2 | W+R=4 > 3 — overlap guaranteed |
| Fast writes | 3 | 1 | 3 | Writes cheap, reads expensive |
| Fast reads | 3 | 3 | 1 | Reads cheap, writes expensive and fragile |
| Eventual | 3 | 1 | 1 | W+R=2 < 3 — may read stale |

> **Interview tip.** Cassandra and DynamoDB expose this directly
> (`QUORUM`, `ONE`, `ALL`). Being able to write `W + R > N` and explain the
> overlap is a concrete, checkable piece of knowledge.

> **Trap.** Quorums give you **overlap**, not linearizability. Without
> additional machinery (read repair, anti-entropy, or a consensus protocol)
> you can still observe anomalies.

### Anti-entropy

How leaderless systems converge:

- **Read repair** — on read, detect divergent replicas and fix them
- **Hinted handoff** — a live node stores writes for a dead node and replays
- **Merkle trees** — compare large datasets efficiently to find differences

## Conflict resolution

With multi-leader or leaderless writes, concurrent updates conflict.

| Strategy | How | Problem |
|----------|-----|---------|
| **Last-write-wins (LWW)** | Highest timestamp wins | **Silent data loss**; clock skew |
| **Version vectors** | Track per-node versions, detect concurrency | Must resolve conflicts somewhere |
| **Application merge** | Hand both versions to the app | Needs domain logic |
| **CRDTs** | Data types that merge deterministically | Limited to certain structures |

**CRDT examples:** G-Counter (grow-only counter), PN-Counter (increment and
decrement), OR-Set (observed-remove set), LWW-Register, RGA (collaborative
text).

> **Interview tip.** CRDTs are the right answer for collaborative editing,
> shopping carts and presence — anything where merging is naturally
> commutative. Google Docs-style editing uses **OT** or CRDTs.

### Clocks

**Physical clocks drift** and are not reliable for ordering across machines.

| Mechanism | Gives |
|-----------|-------|
| **Lamport clocks** | Total order, but cannot distinguish concurrency |
| **Vector clocks** | Partial order; **detects concurrency** |
| **Hybrid logical clocks (HLC)** | Close to physical time, causally correct |
| **TrueTime** (Spanner) | Bounded uncertainty via GPS/atomic clocks |

> **Interview tip.** Spanner achieves external consistency by **waiting out
> the clock uncertainty** (commit wait) using TrueTime. Mentioning this shows
> you understand that strong consistency globally costs latency — physics
> again.

## Consensus

Getting N nodes to agree on a value despite failures.

| Protocol | Notes |
|----------|-------|
| **Paxos** | Correct, famously hard to understand/implement |
| **Raft** | Designed for understandability. **The standard choice** |
| **Zab** | ZooKeeper's protocol |
| **PBFT** | Tolerates Byzantine (malicious) nodes |

### Raft in one page

Three roles: **leader**, **follower**, **candidate**.

```
Follower ──timeout──► Candidate ──majority votes──► Leader
   ▲                                                  │
   └──────── higher term discovered ◄─────────────────┘
```

1. **Leader election** — a follower that hears nothing times out, becomes a
   candidate, requests votes. Majority wins; it becomes leader for a term.
2. **Log replication** — clients write to the leader; it appends and
   replicates. Once a **majority** acknowledge, the entry is **committed**.
3. **Safety** — a candidate can only win if its log is at least as up to date
   as the voter's, so committed entries are never lost.

**Needs a majority** — a 5-node cluster tolerates 2 failures. This is why
consensus clusters have **odd** sizes (3, 5, 7).

> **Interview tip.** You will not implement Raft in an interview, but you
> should be able to say: *"etcd/ZooKeeper give us consensus, used for leader
> election, configuration and distributed locks. They need a majority, so I'd
> run 3 or 5 nodes."*

### Where consensus is used

- **Leader election** — who is the primary?
- **Distributed locks** — but see the warning below
- **Configuration and service discovery** — etcd, Consul, ZooKeeper
- **Distributed transactions** — the commit decision

> **Trap — distributed locks are harder than they look.** A lock held by a
> process that pauses (GC, VM migration) may expire while the process believes
> it still holds it, and two holders then proceed. Mitigate with **fencing
> tokens**: the lock service issues a monotonically increasing token, and the
> downstream resource rejects any token lower than the highest it has seen.

## Distributed transactions

**Two-phase commit (2PC):**

```
Coordinator → prepare → all participants
            ← vote yes/no
Coordinator → commit/abort → all participants
```

> **Trap.** 2PC is **blocking**: if the coordinator dies after `prepare`,
> participants hold locks indefinitely. It also couples availability — any
> participant failing blocks everyone. Avoid it in modern designs.

**Saga** — the practical alternative. A sequence of local transactions, each
with a **compensating action**.

```
Order placed → Payment taken → Inventory reserved → Shipment created
     ↓ fail                ↓ fail              ↓ fail
Cancel order ← Refund payment ← Release inventory
```

| | Choreography | Orchestration |
|---|---|---|
| Control | Each service reacts to events | A central coordinator drives |
| Coupling | Loose | Coordinator knows the flow |
| Visibility | **Hard to trace** | **Easy to see state** |
| Good for | 2–3 steps | Complex flows |

> **Interview tip.** Sagas give **atomicity without isolation** — intermediate
> states are visible. You must design for that (pending states, idempotent
> compensations). Saying this proves you understand the trade, not just the
> word.

## Idempotency

The practical antidote to "exactly once" being impossible.

```python
def charge(idempotency_key, amount):
    existing = store.get(idempotency_key)
    if existing:
        return existing              # replay — return the original result
    result = payment_gateway.charge(amount)
    store.put(idempotency_key, result)
    return result
```

> **Interview tip.** "Exactly-once delivery" does not exist end to end. What
> exists is **at-least-once delivery + idempotent processing**, which is
> equivalent in effect. Say it this way.

## Decision table

| Requirement | Answer |
|-------------|--------|
| Bank balance must be correct | **CP / linearizable**, single leader |
| Shopping cart must always accept writes | **AP** + CRDT merge |
| User must see their own post | **Read-your-writes** |
| Timeline must not jump backwards | **Monotonic reads** |
| Who is the primary? | **Consensus** (Raft/etcd) |
| Tunable staleness | **Quorum** with W+R > N |
| Order across microservices | **Saga** with compensations |
| Avoid double charging | **Idempotency key** |
| Collaborative editing | **CRDT** or OT |
| Globally strong consistency | Spanner-style (commit wait) — accept latency |

## Practice

1. Why is "CA" not a real choice?
2. N=5. Give W and R for strong-ish reads. What about fast writes?
3. A user's comment appears then vanishes on refresh. Which guarantee is
   missing?
4. Why is last-write-wins dangerous? When is it acceptable?
5. A distributed lock holder is paused by GC for 20 s and the lease expires.
   What breaks, and what fixes it?
6. Why is 2PC avoided? What replaces it, and what does that cost?
7. Explain why exactly-once delivery is impossible, and what to build instead.
8. A 5-node etcd cluster loses 3 nodes. What happens, and why?

<details><summary>Answers</summary>

1. Partitions are not something you choose — networks fail. During a
   partition you must either refuse requests (CP) or serve possibly stale data
   (AP). "CA" just means "we have not thought about partitions."
2. Strong-ish: **W=3, R=3** (or W=3,R=3 / W=4,R=2 — any pair with W+R>5).
   Fast writes: **W=1, R=5**. Fast reads: W=5, R=1.
3. **Monotonic reads** — successive reads hit replicas with different
   staleness. Fix by pinning the session to one replica or tracking a version.
4. LWW silently discards a concurrent update, and clock skew can make the
   "later" write the one that arrived first. Acceptable when writes to the
   same key are genuinely rare or when losing one is harmless (e.g. a cached
   profile field).
5. Another process acquires the lock and both believe they hold it, producing
   concurrent mutation. Fix with **fencing tokens** — monotonically increasing
   numbers that the protected resource checks and rejects if stale.
6. 2PC blocks: a coordinator failure after prepare leaves participants holding
   locks, and any participant failure stalls everyone. **Sagas** replace it —
   at the cost of no isolation, so intermediate states are visible and you
   must write compensating actions.
7. Network failures make it impossible to distinguish "message lost" from
   "acknowledgement lost", so a sender must either risk losing or risk
   duplicating. Build **at-least-once delivery with idempotent consumers**
   (idempotency keys, dedup tables, upserts).
8. It **loses quorum** (3 of 5 needed) and becomes read-only/unavailable for
   writes until nodes return. Consensus requires a majority precisely so two
   disjoint halves can never both make progress.
</details>

---

Previous: [06 — Scaling data](06-scaling-data.md) ·
Next: [08 — Messaging and streaming](08-messaging-and-streaming.md)
