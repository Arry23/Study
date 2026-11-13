# 16 — Case studies: core

Four worked designs using the framework from
[chapter 15](15-hld-framework.md). Attempt each yourself before reading on.

---

# 1. URL shortener

The classic opener. Simple enough to finish, deep enough to probe.

## Requirements

**Functional:** shorten a long URL; redirect a short URL; optional custom
alias; optional expiry. **Out of scope:** analytics dashboard, user accounts.

**Non-functional:** redirects must be **fast** (p99 < 50 ms) and **highly
available** (a dead link is a broken product). Reads vastly exceed writes.
Short codes must not be guessable in bulk.

## Estimation

```
100 M new URLs/month → 100M / 2.6M s ≈ 40 writes/s   (peak ~120)
Read:write 100:1      → 4,000 reads/s                (peak ~12,000)

Storage per row ≈ 500 bytes (short_code, long_url, owner, created, expiry)
100 M/month × 500 B = 50 GB/month → 600 GB/year → ~3 TB over 5 years
```

→ Tiny data, huge read ratio. **This is a caching problem, not a storage
problem.**

## API

```
POST /urls        {long_url, custom_alias?, ttl?} → {short_url}
GET  /{code}      → 302 redirect
```

> **Interview tip.** 301 (permanent) is cached by browsers forever — great for
> load, but you lose click tracking and can never change the target. **302** is
> the usual choice. Mentioning this trade-off is a quick win.

## Short code generation

| Approach | Verdict |
|----------|---------|
| Hash the URL (MD5 → base62, take 7) | Collisions need handling; same URL → same code (may be good or bad) |
| Random base62 | Needs a uniqueness check |
| **Counter + base62** | No collisions, but **sequential and enumerable** |
| **Counter + base62, with the counter from a distributed range allocator** | **Best** |

Base62 (`a-zA-Z0-9`) gives 62⁷ ≈ **3.5 trillion** 7-character codes.

> **Trap — enumerability.** A pure counter produces `aaaaaab`, `aaaaaac`… so
> anyone can scrape every link. Fix by encoding the counter with a
> **bijective permutation** (e.g. Feistel/XOR-shift on the integer before
> base62), which keeps uniqueness with no collision checks while making codes
> unguessable.

**Distributed counter:** a **range allocator** — each app instance grabs a
block (say 10,000 IDs) from a central store, then allocates locally with zero
coordination. Survives restarts by just wasting a block.

## Design

```
                        ┌─── CDN / edge cache (hot codes)
Client ──► Load Balancer ┤
                        └─► Redirect service ──► Redis ──► Database
                            Write service   ──► Range allocator
```

- **Redirect path**: Redis lookup → 302. Database only on a cache miss.
- **Write path**: allocate ID → encode → write to DB → write-through to cache.
- **Database**: key-value by `short_code`. DynamoDB/Cassandra fit perfectly;
  Postgres also fine at this volume.

## Deep dives

**Caching.** Hot links dominate (one viral link can be 50% of traffic). Cache
hit rate should exceed 95%. Use LRU, long TTL (URLs are immutable), and for
extreme hot keys add an **in-process cache** in the redirect service.

**Why not shard immediately?** 3 TB and 12k QPS is within reach of a single
replicated store plus cache. If sharding is needed, shard by `short_code`
hash — lookups are always by code, so there are no cross-shard queries.

**Deletion/expiry.** TTL in the cache; a background job or DB-level TTL for
storage.

**Availability.** Redirects are read-only, so serve from multiple regions with
replicas. Writes can be single-region; a brief write outage is tolerable, a
read outage is not.

---

# 2. Rate limiter

Appears both as its own question and as a sub-problem everywhere.

## Requirements

Limit each client to N requests per window; work across a **distributed** fleet;
add minimal latency; fail **open** (prefer serving traffic to blocking it if
the limiter breaks); return 429 with `Retry-After`.

## Algorithm

From [chapter 10](10-reliability-patterns.md): fixed window, sliding log,
sliding counter, token bucket, leaky bucket.

**Choose token bucket** — allows legitimate bursts, bounds the average, and is
two numbers per key.

```python
# Redis + Lua for atomicity
LUA = """
local tokens_key, ts_key = KEYS[1], KEYS[2]
local rate, capacity, now, cost = tonumber(ARGV[1]), tonumber(ARGV[2]),
                                  tonumber(ARGV[3]), tonumber(ARGV[4])
local tokens = tonumber(redis.call('get', tokens_key) or capacity)
local last   = tonumber(redis.call('get', ts_key) or now)
tokens = math.min(capacity, tokens + (now - last) * rate)
local allowed = tokens >= cost
if allowed then tokens = tokens - cost end
redis.call('set', tokens_key, tokens, 'EX', 3600)
redis.call('set', ts_key, now, 'EX', 3600)
return allowed and 1 or 0
"""
```

> **Interview tip.** The **Lua script** matters: read-modify-write across two
> keys must be atomic, or two concurrent requests both see the same token
> count. Saying this shows you have thought about concurrency, not just
> algorithms.

## Where it runs

```
Client → API Gateway (rate limit here) → services
                 │
              Redis cluster
```

At the **gateway**, so every service is protected and the limit is enforced
once.

## Deep dives

**Latency.** A Redis round trip is ~1 ms. For very high throughput, limit
**locally per instance** with a share of the budget and reconcile
periodically — approximate but much faster. Trade accuracy for latency
deliberately.

**Key design.** `rl:{user_id}`, `rl:{ip}`, `rl:{api_key}:{endpoint}`. Different
tiers get different limits.

**Redis failure.** **Fail open** — if the limiter is unavailable, allow the
request. Blocking all traffic because the rate limiter died is a worse outage
than brief over-admission. (For abuse-prevention limits, you might fail closed
— say which and why.)

**Hot key.** A single abusive client hammering one key creates a hot shard.
Shard the key (`rl:{user}:{bucket}`) and divide the limit, or handle at the
edge/WAF.

---

# 3. News feed (Twitter/Instagram timeline)

The canonical fan-out question.

## Requirements

Post; view a home timeline of followed users, newest first; follow/unfollow.
Timeline may be **seconds stale**. Reads greatly exceed writes.

## Estimation

From [chapter 01](01-fundamentals-and-estimation.md):

```
150 M DAU, 2 posts/day   → ~3,500 writes/s  (peak ~10,000)
10 feed views/day        → ~17,000 reads/s  (peak ~50,000)
Read:write ≈ 5:1
```

## The core decision: fan-out

| | **Fan-out on write (push)** | **Fan-out on read (pull)** |
|---|---|---|
| On post | Write into every follower's timeline | Do nothing |
| On read | Read one precomputed list — **fast** | Query all followees, merge — **slow** |
| Cost | O(followers) per post | O(followees) per read |
| Breaks when | A user has 100 M followers | A user follows thousands |

```
Push:  post → [fan-out workers] → timeline:userA, timeline:userB, ...
Pull:  read → fetch recent posts of all followees → merge → sort
```

> **Interview tip.** The answer is a **hybrid**, and saying so with the
> threshold is what scores:
>
> *"Fan-out on write by default, because reads are 5× writes and must be fast.
> But for accounts above ~100k followers, a single post means 100k writes —
> so for those I don't fan out. At read time I merge the precomputed timeline
> with a live pull of the celebrities the user follows. That bounds both
> sides."*

## Design

```
Post → write service → posts DB
                    └─► queue ──► fan-out workers ──► timeline cache (Redis)
                                        │
                                  (skip if author is a celebrity)

Read → timeline service → Redis timeline (precomputed)
                        + live pull for followed celebrities
                        → merge, sort by time, hydrate post bodies
```

**Data model:**

```
posts(post_id PK, author_id, text, created_at)     -- post_id is Snowflake (time-sortable)
follows(follower_id, followee_id)                   -- sharded by follower_id
timeline:{user_id} → Redis list of post_ids (capped at ~800)
```

> **Interview tip.** Store **IDs** in the timeline, not post bodies. Hydrate
> from a post cache at read time. Otherwise editing or deleting a post means
> rewriting millions of timelines.

## Deep dives

**Timeline cap.** Nobody scrolls past a few hundred items. Cap the list and
fall back to a pull for deep pagination.

**Fan-out cost.** 3,500 posts/s × average 200 followers = **700k writes/s** to
Redis. That is why the work is queued and done asynchronously, and why the
celebrity cut-off matters.

**Ordering and pagination.** Snowflake IDs sort by time, so cursor pagination
is `WHERE id < cursor ORDER BY id DESC LIMIT n` — correct even as new posts
arrive.

**Unfollow.** Do not rewrite timelines. Filter at read time, or let the entry
age out.

**Ranked feeds.** If the feed is ranked rather than chronological, the
timeline becomes a **candidate set** and a ranking service scores it at read
time — which pushes you towards pull or a hybrid with a larger candidate pool.

---

# 4. Chat system (WhatsApp/Slack)

Tests real-time delivery, ordering and presence.

## Requirements

1:1 and group messaging; **delivery and read receipts**; online presence;
message history; push notifications when offline. Messages must **not be
lost** and must appear **in order** within a conversation.

## Estimation

```
50 M DAU × 40 messages/day = 2 B messages/day
2 B / 86,400 ≈ 23,000 messages/s  (peak ~70,000)
Message ≈ 300 bytes → 600 GB/day → ~220 TB/year
Concurrent connections ≈ 50 M × 20% online = 10 M
```

→ 10 M concurrent connections at ~50k per server = **~200 gateway servers**.

## Protocol

**WebSocket** — bidirectional, low latency
([chapter 02](02-networking-and-protocols.md)). Fall back to long polling
where WebSockets are blocked.

## Design

```
Client ──WebSocket──► Connection gateway (stateful, ~200 nodes)
                            │
                      Session registry (user_id → gateway node)  [Redis]
                            │
                      Message service ──► Message store (Cassandra)
                            │
                            ├──► Pub/sub backplane (Kafka/Redis)
                            └──► Push notification service (APNs/FCM)
```

**Sending a message:**

1. Client sends over its WebSocket with a **client-generated message ID**
2. Gateway → message service: persist first, then deliver
3. Look up the recipient's gateway in the session registry
4. Deliver via the backplane to that gateway → recipient's socket
5. If offline, enqueue and send a push notification
6. Acknowledge back to the sender (sent → delivered → read)

> **Interview tip.** **Persist before acknowledging.** If you acknowledge then
> crash, the message is lost and the user believes it was sent. This ordering
> is the heart of "messages must not be lost".

## Data model

```
messages(conversation_id, message_id, sender_id, body, created_at)
  partition key: conversation_id
  clustering key: message_id DESC      -- recent messages are one partition read
```

Cassandra fits: write-heavy, partition by conversation, time-ordered within.

## Deep dives

**Ordering.** Per-conversation, not global. Use a per-conversation sequence
number assigned by the message service, or Snowflake IDs. Clients sort by it
and can detect gaps.

**Deduplication.** The client generates a UUID per message and retries with
the same ID; the server upserts on it. This makes at-least-once delivery safe
— the pattern from [chapter 08](08-messaging-and-streaming.md).

**Presence.** Heartbeat every ~30 s into Redis with a TTL of ~60 s. Absence of
a heartbeat means offline. Do **not** broadcast every presence change to every
contact — fan-out is quadratic. Push only to users with the conversation open,
and let others poll.

**Group messages.** For small groups, fan out to each member. For large groups
(10k+), treat it like a feed — write once and let members read, or the fan-out
cost explodes.

**Multi-device.** Each device holds its own connection; the session registry
maps `user_id → [device sessions]`. Sync state per device with a per-device
cursor.

**Gateway failure.** 50k clients reconnect at once. Require **reconnect with
exponential backoff and jitter**, and resume from the last received message ID
rather than refetching history.

---

## What to take from these

| Pattern | Appears in |
|---------|------------|
| Cache the read path aggressively | All four |
| Precompute when reads dominate | Feed |
| Async the expensive part | Feed fan-out, chat delivery |
| Persist before acknowledging | Chat |
| Client-generated IDs for idempotency | Chat |
| Hybrid strategy for skewed data | Feed celebrities |
| Fail open on non-critical infrastructure | Rate limiter |
| Time-sortable IDs | Shortener, feed, chat |

## Practice

1. Redesign the shortener if codes must be **6** characters. What changes?
2. Rate limiting must be exact across 50 regions. What breaks, and what do you
   trade?
3. A user follows 5,000 accounts and has 10 M followers. Which fan-out, and
   why does the hybrid help?
4. In chat, the server crashes after persisting but before acknowledging. What
   does the client do, and why is that safe?
5. Feed: how would you support "edit a post" without rewriting timelines?
6. Chat presence for 10 M users — why is naive broadcast impossible? Compute
   it.

<details><summary>Answers</summary>

1. 62⁶ ≈ 56 billion — still plenty for 100 M/month over years, but the
   keyspace is denser so random generation collides more often. Counter +
   bijective encoding still works; you would monitor exhaustion and plan a
   migration to 7 characters, which the schema should allow from day one.
2. Exactness requires a single global counter, so every request pays a
   cross-region round trip (~150 ms). Trade: enforce **per-region** limits
   (sum ≈ global) and accept approximate global enforcement, or accept the
   latency. Almost always choose approximate.
3. Pull for their followers would mean 10 M reads doing a merge; push would
   mean 10 M writes per post. **Hybrid**: do not fan out this user's posts
   (too many followers); at read time, merge the reader's precomputed timeline
   with a live pull of the few celebrities they follow. Both sides stay
   bounded.
4. The client retries with the **same client-generated message ID**; the
   server upserts and returns the existing result. Safe because the write is
   **idempotent**, so at-least-once delivery cannot duplicate the message.
5. Timelines store **post IDs only**; bodies are hydrated from the post store
   at read time. Editing updates one row, and every timeline reflects it
   automatically.
6. Each presence change would notify every contact: 10 M users × ~200 contacts
   × a few changes an hour is billions of events. Instead, push presence only
   to users currently viewing that conversation, and serve everyone else from
   a Redis lookup on demand.
</details>

---

Previous: [15 — The HLD interview framework](15-hld-framework.md) ·
Next: [17 — Case studies: advanced](17-hld-case-studies-advanced.md)
