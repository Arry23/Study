# 10 — Application integration

Decoupling is a core exam theme. Tightly coupled components fail together;
decoupled ones degrade gracefully.

This workspace is built around a publisher/subscriber split — `ingestion_pub` /
`ingestion_sub`, `product_pub` / `product_sub`, `release_pub` / `release_sub`
and more. That is the pattern this chapter is about, whatever the underlying
transport.

## The four main services

| | **SQS** | **SNS** | **EventBridge** | **Kinesis Data Streams** |
|---|---|---|---|---|
| Model | **Queue** (pull) | **Pub/sub** (push) | **Event bus** (push, filtered) | **Stream** (pull, replayable) |
| Consumers | One per message | Many, fan-out | Many, rule-based | Many, independent position |
| Retention | 1 min – **14 days** | None (fire and forget) | None (archive optional) | 1 – **365 days** |
| Ordering | FIFO queues only | FIFO topics only | No | **Per shard** |
| Replay | No | No | From archive | **Yes** |

> **Exam tip — the one-line rule:**
> - **Decouple two components / buffer work** → **SQS**
> - **One message, many subscribers** → **SNS**
> - **Route events by content, many AWS sources/targets** → **EventBridge**
> - **Ordered, replayable, real-time analytics** → **Kinesis**

## SQS

### Standard vs FIFO

| | Standard | FIFO |
|---|---|---|
| Throughput | Nearly unlimited | 300 TPS (3,000 batched); high-throughput mode higher |
| Ordering | **Best effort** | **Strict**, per message group |
| Delivery | **At least once** (duplicates possible) | **Exactly once** |
| Queue name | any | must end **`.fifo`** |

> **Exam tip.** "Order must be preserved" or "no duplicates" → **FIFO**.
> Otherwise **Standard**, which is cheaper and scales further.

> **Trap.** Standard queues can deliver duplicates. Consumers must be
> **idempotent**. A question describing duplicate processing is asking either
> for FIFO or for idempotency.

### Key settings

| Setting | Meaning |
|---------|---------|
| **Visibility timeout** | How long a received message is hidden (default 30 s) |
| **Message retention** | 1 minute – 14 days (default 4 days) |
| **Delivery delay** | 0 – 15 minutes |
| **Long polling** | `ReceiveMessageWaitTimeSeconds` up to 20 s |
| **DLQ** | After `maxReceiveCount` failures, move the message aside |

> **Exam tip — visibility timeout.** "Messages are processed more than once" →
> the visibility timeout is **shorter than processing time**. Increase it, or
> call `ChangeMessageVisibility` while working.

> **Exam tip — long polling.** "Reduce empty receives and API costs" →
> **long polling**.

> **Exam tip — DLQ.** "Isolate messages that repeatedly fail so they do not
> block the queue" → **dead-letter queue**.

Message size is 256 KB; for larger payloads put the object in **S3** and send
the reference (the Extended Client Library does this).

### SQS as a buffer

The classic pattern:

```
Producers → SQS → ASG of consumers
                   ↑
           scale on ApproximateNumberOfMessagesVisible
```

> **Exam tip.** "Handle unpredictable spikes without losing requests" →
> put SQS in front and scale consumers on **queue depth**, not CPU.

## SNS

Push-based pub/sub. One publish, many deliveries.

Subscriber types: SQS, Lambda, HTTP/S, email, SMS, mobile push, **Kinesis Data
Firehose**.

| Feature | Purpose |
|---------|---------|
| **Message filtering** | Subscribers receive only matching messages |
| **FIFO topics** | Ordering and dedup (SQS FIFO subscribers only) |
| **Message Data Protection** | Detect/redact PII in messages |
| DLQ | For failed deliveries |

### Fan-out

```
          ┌→ SQS queue A → consumer A
SNS topic ┼→ SQS queue B → consumer B
          └→ Lambda
```

> **Exam tip.** "One event must be processed by several independent systems,
> each at its own pace, without losing messages" → **SNS fan-out to SQS
> queues**. SNS alone has no retention; the queues provide durability.

This is the pattern the `*_pub` / `*_sub` split in this workspace implements.

## EventBridge

An event bus with content-based routing.

```
Source (AWS service | your app | SaaS partner)
   → Event bus (default | custom | partner)
     → Rule (event pattern)
       → Target (Lambda, SQS, SNS, Step Functions, ECS, 20+ others)
```

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": { "bucket": { "name": ["my-bucket"] } }
}
```

| Feature | Purpose |
|---------|---------|
| **Schema registry** | Discover and codegen event shapes |
| **Archive and replay** | Re-run past events |
| **Scheduler** | Cron/rate at scale — replaces CloudWatch Events rules |
| **Pipes** | Point-to-point source → filter → enrich → target |
| **API destinations** | Deliver to external HTTP endpoints |

> **Exam tip.** "React to AWS service events" (EC2 state change, CodePipeline
> result, GuardDuty finding) → **EventBridge**. It is the successor to
> CloudWatch Events and the default answer for AWS-native event routing.

> **Exam tip.** "Run something on a schedule" → **EventBridge Scheduler**
> (or a rule with a cron expression).

### SNS vs EventBridge

| Choose | When |
|--------|------|
| **SNS** | Simple fan-out, very high throughput, mobile/SMS/email endpoints |
| **EventBridge** | Content-based routing, many AWS sources, SaaS integrations, archive/replay |

## Kinesis

| Service | Purpose |
|---------|---------|
| **Data Streams** | Real-time ingest, shards, replay, custom consumers |
| **Data Firehose** | **Fully managed delivery** to S3, Redshift, OpenSearch, Splunk |
| **Managed Service for Apache Flink** | Real-time SQL/Flink analytics |
| **Video Streams** | Media ingest |

**Data Streams:** ordering per shard; capacity = shards × 1 MB/s in,
2 MB/s out. **Enhanced fan-out** gives each consumer its own 2 MB/s.

> **Exam tip — Streams vs Firehose.**
> - Need **replay**, sub-second latency, custom processing → **Data Streams**
> - Just need data **landed in S3/Redshift/OpenSearch** with no code →
>   **Firehose** (near-real-time, ~60 s buffer, fully managed)

> **Exam tip — Kinesis vs SQS.** Kinesis when **multiple consumers must read
> the same records independently**, order matters, or you need replay. SQS when
> each message is processed once and then gone.

**MSK** is managed Apache Kafka — choose it when the question says Kafka,
existing Kafka tooling, or migration from self-managed Kafka.

## Amazon MQ

Managed ActiveMQ / RabbitMQ.

> **Exam tip.** "Migrate an existing application using **JMS / AMQP / MQTT /
> STOMP**" → **Amazon MQ**. For anything new and cloud-native, SQS/SNS.

## AppFlow and Step Functions

- **AppFlow** — no-code data transfer between SaaS (Salesforce, Slack) and AWS
- **Step Functions** — orchestration, covered in [chapter 09](09-serverless-and-containers.md)

## Patterns the exam rewards

**Queue-based load levelling** — buffer bursts so the backend runs at a steady
rate.

**Fan-out** — SNS → multiple SQS queues.

**Idempotent consumers** — required with at-least-once delivery.

**Dead-letter queues** — everywhere; never let a poison message block a queue.

**Event-driven over polling** — S3 event → Lambda beats a cron job scanning the
bucket.

> **Exam tip.** If an answer involves an EC2 instance polling for work on a
> timer, there is almost always a better event-driven option.

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| Decouple producer and consumer | **SQS** |
| Absorb traffic spikes | **SQS** + ASG on queue depth |
| Strict ordering, no duplicates | **SQS FIFO** |
| Messages failing repeatedly | **DLQ** |
| Processed twice | Increase **visibility timeout** / idempotency |
| Reduce empty polls | **Long polling** |
| One event, several systems | **SNS fan-out to SQS** |
| Route by event content | **EventBridge** |
| React to an AWS service event | **EventBridge** |
| Run a task every hour | **EventBridge Scheduler** |
| Replay the last 7 days of events | **Kinesis Data Streams** or EventBridge archive |
| Stream logs to S3 with no code | **Kinesis Data Firehose** |
| Kafka workload | **MSK** |
| Existing JMS/AMQP app | **Amazon MQ** |
| Payload over 256 KB | S3 + message reference |

## Check yourself

1. SQS vs SNS in one sentence each.
2. Why must Standard SQS consumers be idempotent?
3. What causes duplicate processing, and how do you fix it?
4. What does a DLQ protect against?
5. Why fan out to SQS queues rather than Lambda directly?
6. Kinesis Data Streams vs Firehose?
7. When is Kinesis right and SQS wrong?
8. When is Amazon MQ the answer?

---

Previous: [09 — Serverless and containers](09-serverless-and-containers.md) ·
Next: [11 — DNS, CDN and global reach](11-dns-cdn-and-global.md)
