# 08 — Messaging and streaming

Asynchrony is how you decouple services, absorb spikes, and stop one slow
component taking down the system.

## Why asynchronous

```
Synchronous:   client → service A → service B → service C → response
               (latency adds up; any failure fails the request)

Asynchronous:  client → service A → queue → response "accepted"
                                      ↓
                              worker processes later
```

| Benefit | Explanation |
|---------|-------------|
| **Decoupling** | A does not need B to be up |
| **Load levelling** | Queue absorbs spikes; workers run at steady rate |
| **Retry** | Failed work stays queued |
| **Fan-out** | One event, many consumers |
| **Responsiveness** | Return 202 immediately |

> **Trade-off.** You gain resilience and lose **immediacy and simplicity**.
> The user no longer knows the work succeeded, so you need status endpoints,
> notifications, or optimistic UI. Debugging becomes distributed tracing.

## Queue vs pub/sub vs log

| | **Queue** | **Pub/Sub** | **Log** |
|---|---|---|---|
| Delivery | One consumer per message | All subscribers | All consumers, independent offsets |
| Retention | Until consumed | None (fire and forget) | **Time/size based, replayable** |
| Ordering | Usually per-queue or per-group | No | **Per partition** |
| Examples | SQS, RabbitMQ | SNS, Redis pub/sub | **Kafka**, Pulsar, Kinesis |

> **Interview tip.** The three-line rule:
> - **Work must be done once** → queue
> - **Many systems react to one event** → pub/sub
> - **Replay, ordering, or stream processing** → log

## Delivery semantics

| Semantics | Meaning | Reality |
|-----------|---------|---------|
| **At most once** | Fire and forget | Messages can be lost |
| **At least once** | Retry until acknowledged | **Duplicates possible** — the default |
| **Exactly once** | Each message processed once | Not achievable end to end |

> **Interview tip.** Say it precisely: *"Exactly-once delivery is impossible;
> what we build is at-least-once delivery with idempotent consumers, which is
> operationally equivalent."* Kafka's "exactly-once semantics" is
> transactional read-process-write **within Kafka**, not across external
> systems.

### Making consumers idempotent

```python
def handle(message):
    if dedup_store.seen(message.id):       # 1. dedup table
        return
    with db.transaction():
        apply(message)
        dedup_store.mark(message.id)       # same transaction
```

Other techniques: **upserts** keyed on a business ID, **conditional writes**
(`UPDATE ... WHERE version = ?`), and naturally idempotent operations
(`SET status = 'paid'` rather than `balance += x`).

## Ordering

Global ordering across a distributed queue is expensive and usually
unnecessary.

| Approach | Guarantee |
|----------|-----------|
| Single partition / FIFO queue | Total order, **throughput limited** |
| **Partition by key** | **Order per key** — the usual answer |
| Sequence numbers | Consumer reorders or detects gaps |

> **Interview tip.** "Order events per user by partitioning on `user_id`" gives
> you the ordering that matters with full parallelism. Global ordering is
> almost always over-specified.

## Kafka

Worth knowing in detail — it appears in most streaming answers.

```
Topic "orders"
├── Partition 0: [msg0][msg1][msg2] ...   ← append-only, ordered
├── Partition 1: [msg0][msg1] ...
└── Partition 2: [msg0][msg1][msg2] ...

Consumer group "billing":  c1 → P0,  c2 → P1,  c3 → P2
Consumer group "analytics": c1 → P0,P1,P2      ← independent offsets
```

| Concept | Meaning |
|---------|---------|
| **Topic** | A named stream |
| **Partition** | Unit of parallelism and ordering |
| **Offset** | Consumer's position — the consumer owns it |
| **Consumer group** | Partitions are divided among members |
| **Replication factor** | Copies per partition |
| **ISR** | In-sync replicas; `acks=all` waits for them |
| **Retention** | Time or size based; can be `compact` |

**Key properties:**

- **Parallelism is bounded by partition count.** More consumers than
  partitions means idle consumers.
- **Ordering is per partition**, not per topic.
- **Consumers are independent** — adding a new consumer group replays history
  without affecting others.
- **Log compaction** keeps the latest value per key — effectively a changelog.

```
acks=0    → fire and forget, fastest, can lose data
acks=1    → leader acknowledges, can lose on leader failure
acks=all  → all in-sync replicas, safest, slowest
```

> **Interview tip.** `acks=all` + `min.insync.replicas=2` + replication
> factor 3 is the standard durable configuration. Being able to state it is a
> good signal.

### Kafka vs traditional queue

| Use Kafka when | Use SQS/RabbitMQ when |
|---|---|
| Multiple independent consumers | One consumer per message |
| Replay matters | Fire and forget is fine |
| Ordering per key matters | Order does not matter |
| Very high throughput | Moderate throughput |
| Stream processing downstream | Simple background jobs |
| | **You want zero operational burden** |

> **Trap.** Kafka is operationally heavy (brokers, ZooKeeper/KRaft, partition
> planning, rebalances). Do not propose it for "send a welcome email". That is
> a queue.

## Dead letter queues

After N failed attempts, move the message aside.

```
main queue → consumer fails 3× → DLQ → alert, inspect, replay
```

> **Interview tip.** Without a DLQ, a **poison message** blocks the queue
> forever or loops endlessly, consuming capacity. Always mention a DLQ and a
> retry policy with **exponential backoff and jitter**.

## The dual-write problem

A subtle and important failure:

```python
db.save(order)              # succeeds
queue.publish(order_event)  # fails → event lost, DB and queue disagree
```

There is no transaction across two systems. Three solutions:

**1. Transactional outbox** — write the event to an `outbox` table in the
**same transaction**, then a relay publishes it.

```python
with db.transaction():
    db.save(order)
    db.save_outbox(order_event)     # one atomic commit
# separate process polls outbox → publishes → marks sent
```

**2. CDC (Change Data Capture)** — tail the database's replication log
(Debezium) and publish changes. No application change; the database is the
source of truth.

**3. Event sourcing** — the event log *is* the database; state is derived.

> **Interview tip.** Naming the **dual-write problem** and offering the
> **outbox pattern** is one of the strongest things you can say in a
> microservices design question. Most candidates write `db.save(); publish()`
> and never notice.

## Saga

Covered in [chapter 07](07-consistency-and-consensus.md). Messaging is how
sagas are implemented — each step publishes an event, the next step reacts, and
failures trigger compensating events.

## Backpressure

When producers outpace consumers:

| Strategy | Behaviour |
|----------|-----------|
| **Buffer** | Queue grows — bounded, or you run out of memory |
| **Block** | Slow the producer down |
| **Drop** | Shed load (oldest, newest, or by priority) |
| **Scale** | Add consumers — only if partitions allow |

> **Trap.** An **unbounded** queue does not solve overload; it converts a fast
> failure into a slow one with enormous latency. Bound your queues and decide
> what to do when full.

## Stream processing

| Concept | Meaning |
|---------|---------|
| **Windowing** | Tumbling (fixed), sliding (overlapping), session (gap-based) |
| **Event time vs processing time** | When it happened vs when you saw it |
| **Watermarks** | How long to wait for late events |
| **Stateful processing** | Aggregations, joins — needs checkpointed state |

> **Interview tip.** **Event time** is almost always what you want (a mobile
> client may upload events hours late), and watermarks are how you bound the
> wait. Mentioning late-arriving data shows real streaming experience.

Tools: Kafka Streams, Flink, Spark Streaming.

## Decision table

| Requirement | Answer |
|-------------|--------|
| Background job, done once | **Queue** (SQS/RabbitMQ) |
| Several systems react to an event | **Pub/sub** or Kafka |
| Replay last 7 days | **Kafka** / log |
| Order per user | Partition by `user_id` |
| No duplicates | At-least-once + **idempotent consumer** |
| Poison message | **DLQ** + backoff |
| DB and event must both happen | **Outbox** or CDC |
| Transaction across services | **Saga** |
| Smooth a traffic spike | Queue + autoscaled consumers |
| Producers too fast | **Bounded queue** + backpressure/shedding |
| Aggregate over time windows | **Stream processor** (Flink/Kafka Streams) |

## Practice

1. Why can't "exactly once" be guaranteed, and what do you build instead?
2. A topic has 4 partitions and a consumer group of 10. What happens?
3. Describe the dual-write problem and two fixes.
4. One message fails forever. What happens without a DLQ? With one?
5. You need per-user ordering with high throughput. Design it.
6. When is Kafka the wrong choice?
7. A queue grows without bound during a spike. What is wrong with that, and
   what are your options?
8. Event timestamps arrive 6 hours late from mobile clients. How do you
   aggregate hourly counts correctly?

<details><summary>Answers</summary>

1. A sender cannot distinguish a lost message from a lost acknowledgement, so
   it must either risk loss or risk duplication. Build **at-least-once
   delivery plus idempotent consumers** (dedup table, upserts, conditional
   writes).
2. **6 consumers sit idle.** Parallelism is capped by partition count;
   increase partitions to increase parallelism.
3. Writing to the database and publishing an event are not atomic, so a crash
   between them leaves them inconsistent. Fix with the **transactional
   outbox** (write event in the same transaction, relay publishes) or **CDC**
   from the replication log.
4. Without: it blocks the queue (FIFO) or retries forever, consuming capacity
   and possibly looping. With: after N attempts it moves to the DLQ, the queue
   drains, and you alert, inspect and replay.
5. Kafka topic partitioned by `user_id` — order is guaranteed within a
   partition, and you get parallelism across users. Scale by adding partitions
   (noting that changing partition count rehashes key→partition mapping).
6. Low volume, simple background work, one consumer, no replay need — and
   especially when you lack the operational capacity to run it. Use a managed
   queue instead.
7. An unbounded queue turns a fast failure into unbounded latency and
   eventually exhausts memory. Options: bound the queue and shed load, apply
   backpressure to producers, autoscale consumers, or prioritise and drop
   low-value work.
8. Use **event time** with **windowing** and a **watermark** that allows late
   arrivals up to your tolerance; emit a provisional result and **update it**
   when late data arrives (or write corrections downstream). Keep the window
   state until the watermark passes.
</details>

---

Previous: [07 — Consistency and consensus](07-consistency-and-consensus.md) ·
Next: [09 — Storage and search](09-storage-and-search.md)
