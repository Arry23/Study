# 15 — The HLD interview framework

Knowing the building blocks is necessary. Driving a 45-minute conversation is
a separate skill, and it is the one people fail on.

## What is actually being assessed

| Signal | What it looks like |
|--------|--------------------|
| **Structure** | You drive; the interviewer does not have to rescue you |
| **Requirements** | You clarify before designing |
| **Quantification** | You put numbers on things |
| **Trade-offs** | You present options and choose, with reasons |
| **Depth** | You can go three levels down on at least one component |
| **Communication** | The interviewer can follow your diagram |

> You are **not** being assessed on producing the "right" design. There isn't
> one. You are being assessed on how you reason.

## The timeline

| Phase | Minutes | Output |
|-------|---------|--------|
| **1. Requirements** | 5–8 | Functional, non-functional, explicit scope cuts |
| **2. Estimation** | 3–5 | QPS, storage, bandwidth, read:write ratio |
| **3. API and data model** | 5 | Endpoints, core entities |
| **4. High-level design** | 10 | The boxes-and-arrows diagram |
| **5. Deep dive** | 10–15 | One or two components, properly |
| **6. Wrap-up** | 3–5 | Bottlenecks, failure modes, what you would do next |

> **Trap.** The commonest failure is spending 25 minutes on requirements and
> never drawing a system. The second commonest is drawing a system in minute
> two without asking a single question. Watch the clock out loud:
> *"I've used about 8 minutes on requirements — let me move to estimation."*

## Phase 1 — Requirements

### Functional

Ask what the system must **do**, then cut aggressively.

```
"Design Twitter"
→ Post a tweet
→ View a home timeline
→ Follow a user
→ (out of scope: search, DMs, ads, trends, notifications, media transcoding)
```

> **Interview tip.** Proposing the scope cut yourself is a strong move:
> *"I'll focus on posting, timeline and follow. I'll leave search and DMs out
> unless you'd like me to cover them."* It shows judgement and buys you time.

### Non-functional

These drive the architecture far more than the features do.

| Question | Why it matters |
|----------|----------------|
| **How many users? DAU?** | Everything |
| **Read:write ratio?** | Caching, replicas, fan-out strategy |
| **Latency target?** | Sync vs async, caching, geography |
| **Consistency needs?** | CP vs AP, which parts can be stale |
| **Availability target?** | Redundancy, multi-region |
| **Global or one region?** | CDN, replication, data residency |
| **Data retention?** | Storage cost, tiering |
| **Growth rate?** | How much headroom to design for |

> **Interview tip.** Ask the **consistency** question explicitly. It is the
> one candidates forget, and it changes the design more than any other answer.
> *"Is it acceptable for a follower to see a tweet a few seconds late?"*

### Write them down

Keep a visible list in the corner of the board. You will refer back to it, and
it shows the interviewer you are designing **to requirements** rather than
from memory.

## Phase 2 — Estimation

Use the method from [chapter 01](01-fundamentals-and-estimation.md). State
assumptions out loud; round aggressively.

Produce four numbers:

```
Write QPS (average and peak)
Read QPS  (average and peak)
Storage per day / per year
Bandwidth
```

Then **say what they imply**:

> *"A 5:1 read:write ratio and 50k peak reads means caching and read replicas
> matter more than write sharding. Images dominate storage, so they belong in
> object storage behind a CDN, not the database."*

> **Interview tip.** The numbers are only useful if you draw a conclusion from
> them. Candidates who compute QPS and then ignore it get no credit.

## Phase 3 — API and data model

A handful of endpoints, not an OpenAPI spec.

```
POST   /tweets            {text, media_ids}        → {tweet_id}
GET    /timeline?cursor=  → {tweets[], next_cursor}
POST   /users/{id}/follow
```

> **Interview tip.** Use **cursor pagination**, not offset
> ([chapter 21](21-api-design.md)). On a feed, `OFFSET 10000` is both slow and
> wrong when new items arrive. Mentioning this unprompted is a nice signal.

Then the core entities:

```
User(id, handle, created_at)
Tweet(id, author_id, text, created_at)      -- id is time-sortable (Snowflake)
Follow(follower_id, followee_id, created_at)
```

Say which store each lives in and why.

## Phase 4 — High-level design

Draw the skeleton first, then specialise.

```
Clients → CDN → API Gateway → Load Balancer
                                   │
                        ┌──────────┼──────────┐
                     service    service    service
                        │          │          │
                     Cache      Queue       Blob store
                        │          │
                      Database  Workers
```

**Narrate as you draw.** Say what each box is for and why it is there. A
silent diagram teaches the interviewer nothing.

> **Interview tip.** Start **simple and correct**, then evolve it. "Here's the
> single-server version. The bottleneck is X, so I'll add Y." That narrative is
> worth more than a complex diagram produced all at once.

## Phase 5 — Deep dive

The interviewer will pick a component, or you should offer one.

> *"The interesting part here is timeline generation — shall I go deeper on
> that?"*

Good deep-dive topics, in roughly descending frequency:

| Topic | Typical question |
|-------|------------------|
| **Fan-out strategy** | Push vs pull, and the celebrity problem |
| **Sharding** | Shard key choice and its consequences |
| **Caching** | What, where, TTL, invalidation, stampede |
| **Consistency** | Which operations need strong consistency |
| **Hot keys / skew** | One user or key dominating |
| **Failure handling** | What happens when each box dies |
| **Idempotency** | Retries and duplicates |

Go **three levels deep** on one thing rather than one level on everything.

## Phase 6 — Wrap-up

Volunteer the weaknesses. It reads as confidence, not doubt.

```
"Bottlenecks: the timeline cache is the hot path — I'd watch its hit rate.
 Single points of failure: the ID generator; I'd run it as a quorum or move
 to Snowflake IDs generated in-process.
 If I had more time: media pipeline, search, and multi-region.
 Trade-offs I made: chose eventual consistency for timelines to keep writes
 cheap; that's acceptable because a few seconds of staleness is invisible."
```

## Driving the conversation

**Think aloud.** Silence is unreadable.

**Offer options, then choose.**

> *"Two options for the timeline: fan-out on write, which makes reads trivial
> but costs a lot for users with millions of followers; or fan-out on read,
> which is cheap to write but expensive to read. Given 5:1 reads, I'd default
> to fan-out on write and handle celebrities with a hybrid. Shall I go into
> the hybrid?"*

That sentence contains: alternatives, a decision, a reason tied to a number,
and an invitation. It is the shape to aim for.

**Check in at phase boundaries.** *"Does that structure make sense before I
go deeper?"*

**Handle hints.** If the interviewer says "what about X?", they are telling
you X matters. Do not defend your design — explore X.

**Recover from mistakes openly.** *"Actually, that shard key creates a hotspot
on recent data. Let me change it to hash(user_id)."* Self-correction scores
well; defending a flawed choice does not.

## Common failure modes

| Mistake | Fix |
|---------|-----|
| Designing before clarifying | Always ask requirements first |
| No numbers | Estimate, and state the implication |
| Jumping to microservices | Start simple, justify each split |
| One level deep everywhere | Pick one component and go deep |
| Silent drawing | Narrate continuously |
| Ignoring failures | Ask "what if this box dies?" of each box |
| Defending a bad choice | Change it; say why |
| Running out of time | Watch the clock, announce transitions |
| Buzzwords without substance | Only name a technology you can justify |

> **Trap.** Saying "I'd use Kafka" and being unable to explain partitions,
> consumer groups and why not SQS is worse than saying "a queue". Only reach
> for a specific technology if you can defend it.

## A reusable checklist

```
REQUIREMENTS
  □ Functional, scoped down
  □ Scale: DAU, QPS, read:write
  □ Latency, consistency, availability targets
  □ Global? Retention? Growth?

ESTIMATE
  □ Write QPS, read QPS, peak
  □ Storage/day, /year
  □ Bandwidth
  □ State the implication

API + DATA
  □ 3–5 endpoints, cursor pagination
  □ Core entities, ID strategy
  □ Store per entity, justified

DESIGN
  □ Simple version first
  □ Identify the bottleneck
  □ Add: cache, replicas, queue, CDN, shards — each justified

DEEP DIVE
  □ One component, three levels

WRAP
  □ Bottlenecks, SPOFs, failure modes
  □ Trade-offs made
  □ What's next
```

## Practice

1. You are asked to "design Instagram". Write the first five questions.
2. Reads are 100× writes. Name three design consequences.
3. The interviewer says "what if that cache goes down?" What do you do?
4. You are 20 minutes in and still on requirements. What now?
5. You realise your shard key is wrong at minute 35. Say it out loud — how?
6. The interviewer asks for Kafka and you are unsure. What do you say?
7. Give the "options, decision, reason, invitation" sentence for choosing a
   database for a feed.

<details><summary>Answers</summary>

1. DAU and growth? Read:write ratio? Is the feed chronological or ranked? Do
   we handle media upload and transcoding, or just metadata? What staleness is
   acceptable on the feed? (Also: global or single region; retention.)
2. Cache aggressively; add read replicas and serve reads from them; consider
   precomputing/denormalising the read path (fan-out on write). Write sharding
   is probably **not** the priority.
3. Answer honestly: latency rises and load shifts to the database, which may
   not survive it. Mitigate with a circuit breaker, request coalescing,
   gradual warm-up, and capacity planning that assumes the cache can vanish.
   Say that a cold cache is often the hardest part of a cache outage.
4. Announce the transition and move: *"I've spent longer than I'd like here —
   let me lock the scope to posting, timeline and follow, and move to
   estimation."* Cutting scope yourself is better than being rescued.
5. *"I want to revisit something — sharding by `created_at` puts all writes on
   the newest shard. I'd switch to hashing on `user_id`, which costs me range
   scans but removes the hotspot."* Self-correction with a reason.
6. *"I'd use a durable log so multiple consumers can read independently and
   replay — Kafka is the usual choice. I'm more confident on the properties I
   need than on Kafka's operational details; shall I describe the properties?"*
   Honesty beats bluffing, and you still showed what matters.
7. *"Two options: Postgres with a denormalised feed table, which gives me
   transactions and easy ad-hoc queries but caps write throughput on one
   primary; or Cassandra, which handles the write volume and partitions
   naturally by user but gives up joins and strong consistency. At 10k peak
   writes a second, Postgres with sharding is still viable and much simpler,
   so I'd start there and keep Cassandra as the escape hatch. Want me to size
   the Postgres option?"*
</details>

---

Previous: [14 — Deployment and operations](14-deployment-and-operations.md) ·
Next: [16 — Case studies: core](16-hld-case-studies-core.md)
