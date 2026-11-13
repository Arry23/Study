# 04 — Caching

The highest-leverage optimisation in most systems, and the one with the most
subtle failure modes.

> There are only two hard things in computer science: cache invalidation and
> naming things. — Phil Karlton

## Where caches live

```
Browser cache          ← free, closest, least control
   ↓
CDN / edge             ← static + cacheable dynamic
   ↓
Load balancer / proxy  ← Nginx/Varnish response cache
   ↓
Application (in-process) ← fastest, per-instance, not shared
   ↓
Distributed cache      ← Redis/Memcached, shared, the usual answer
   ↓
Database buffer pool   ← the DB's own cache
   ↓
Disk
```

> **Interview tip.** Name the layer you mean. "Add caching" is vague; "put a
> CDN in front of media and a Redis look-aside cache in front of the user
> profile query" is a design.

## In-process vs distributed

| | In-process | Distributed |
|---|---|---|
| Latency | **~100 ns** | ~0.5–1 ms (network) |
| Shared | No — per instance | **Yes** |
| Consistency | Hard (N copies drift) | Single source |
| Capacity | Bounded by instance RAM | Scales out |
| On restart | Cold | Survives |

> **Trade-off.** In-process is dramatically faster but each instance has its
> own copy, so invalidation is a broadcast problem and hit rates fall as you
> scale out. Most systems use **both**: a small in-process cache for
> ultra-hot, immutable data, backed by Redis.

## Caching patterns

### Cache-aside (lazy loading) — the default

```python
def get_user(user_id):
    key = f"user:{user_id}"
    cached = cache.get(key)
    if cached is not None:
        return cached                       # hit
    user = db.query("SELECT ... WHERE id=%s", user_id)   # miss
    cache.set(key, user, ttl=300)
    return user
```

| Pros | Cons |
|------|------|
| Only requested data is cached | Every miss pays DB latency |
| Cache failure is survivable | Data can go stale |
| Simple | Cold start is slow |

> **Interview tip.** This is the default answer. Mention that the cache can
> die without taking the system down — an important property.

### Read-through

The cache itself loads from the database on a miss. Same shape, but the
loading logic lives in the cache layer rather than the application.

### Write-through

```python
def update_user(user_id, data):
    db.update(user_id, data)
    cache.set(f"user:{user_id}", data, ttl=300)
```

Cache is always fresh; writes are slower; you cache data that may never be
read.

### Write-behind (write-back)

Write to the cache, flush to the database asynchronously.

> **Trap.** Fast writes, but **data loss if the cache dies before flushing**.
> Only acceptable where loss is tolerable (view counters, analytics).

### Refresh-ahead

Proactively refresh entries about to expire, so popular keys never miss.

> **Interview tip.** Good for predictable hot keys; wasteful if you guess
> wrong.

| Pattern | Use when |
|---------|----------|
| **Cache-aside** | Default. Read-heavy, tolerate some staleness |
| Read-through | Same, with cache-managed loading |
| **Write-through** | Reads must be fresh; writes can be slower |
| Write-behind | Write-heavy, loss tolerable |
| Refresh-ahead | Predictable hot keys, latency-critical |

## Eviction policies

| Policy | Evicts | Good for |
|--------|--------|----------|
| **LRU** | Least recently used | **General purpose** |
| LFU | Least frequently used | Stable popularity distributions |
| FIFO | Oldest inserted | Rarely right |
| TTL | Expired entries | Bounding staleness |
| Random | Any | Cheap; surprisingly acceptable |
| **W-TinyLFU** | Frequency + recency sketch | Modern default (Caffeine) |

> **Trap — LRU and scans.** A full table scan touches every row once and
> evicts your entire hot set. **LFU** or segmented LRU resists this. Mention
> scan resistance and you have shown real caching experience.

## Invalidation — the hard part

Three strategies, in increasing order of freshness and complexity:

**1. TTL** — let it expire.

```python
cache.set(key, value, ttl=300)
```

Simple and self-healing. The staleness window is the TTL. Nearly always the
right starting point.

**2. Explicit invalidation** — delete on write.

```python
def update_user(user_id, data):
    db.update(user_id, data)
    cache.delete(f"user:{user_id}")   # delete, do not update
```

> **Interview tip.** **Delete rather than update.** Updating the cache from
> the writer races with concurrent readers and can leave a stale value
> permanently. Deleting means the next read repopulates from the source of
> truth.

**3. Event-driven invalidation** — the database or a change stream publishes
changes; caches subscribe.

Most accurate, most machinery (CDC, Debezium, Kafka — see
[chapter 08](08-messaging-and-streaming.md)).

### Write-invalidate race

```
T1: reader gets a cache miss, reads DB  → value = A
T2: writer updates DB                   → value = B
T3: writer deletes the cache key
T4: reader writes its stale A into cache ← now permanently wrong
```

Mitigations: short TTL as a backstop, **delete-after-delay** (invalidate
twice), versioned keys, or a single-writer pattern.

> **Interview tip.** Being able to name this race is a strong signal. Most
> candidates never mention it.

## Cache stampede (thundering herd)

A hot key expires; a thousand concurrent requests all miss and hit the
database simultaneously.

```
       TTL expires
            ↓
1000 requests → all miss → all query DB → DB falls over
```

**Fixes:**

| Technique | How |
|-----------|-----|
| **Locking / single flight** | First miss acquires a lock, others wait for its result |
| **Probabilistic early expiry** | Refresh early with probability rising near TTL |
| **Stale-while-revalidate** | Serve stale, refresh in background |
| **Jittered TTL** | `ttl = base + random(0, spread)` so keys do not expire together |

```python
import random
cache.set(key, value, ttl=300 + random.randint(0, 60))   # jitter
```

> **Interview tip.** Jitter is one line and prevents synchronised expiry across
> millions of keys. Mention it whenever you set a TTL.

## Other failure modes

**Cache penetration** — requests for keys that do not exist bypass the cache
every time (a common attack). Fix: cache the negative result with a short TTL,
or use a **Bloom filter** to reject known-absent keys cheaply.

**Cache avalanche** — a large portion of the cache expires or the cache
cluster dies, and full traffic hits the database. Fix: jitter, multi-tier
caching, circuit breakers, and capacity planning that assumes the cache can
vanish.

**Hot key** — one key gets a disproportionate share of traffic (a celebrity
user). Fix: replicate the key across nodes with a suffix
(`celeb:123:{0..9}`), use in-process caching for that key, or shard by
something else.

## Consistency

> **Trade-off.** A cache is a **deliberate consistency compromise**. You trade
> freshness for latency and load. The design question is always "how stale can
> this be?" — and the answer differs per data type.

| Data | Acceptable staleness |
|------|---------------------|
| Account balance | **None** — do not cache, or cache with versioning |
| User profile | Seconds to minutes |
| Product catalogue | Minutes |
| Analytics dashboard | Minutes to hours |
| Static assets | Forever (use content hashing in the URL) |

**Content-hashed URLs** (`app.3f9a2b.js`) are the cleanest answer for static
assets: cache forever, and a new deploy produces a new URL. Mention this.

## Redis vs Memcached

| | **Redis** | **Memcached** |
|---|---|---|
| Data types | Strings, lists, sets, sorted sets, hashes, streams, HLL | Strings only |
| Persistence | **RDB + AOF** | None |
| Replication / HA | **Yes**, with Sentinel/Cluster | No |
| Threading | Mostly single-threaded | **Multi-threaded** |
| Extras | Pub/sub, Lua, transactions, geospatial | — |

> **Interview tip.** Redis for anything beyond a plain key-value cache —
> leaderboards (sorted sets), rate limiting (atomic counters + Lua),
> distributed locks, pub/sub. Memcached only for a simple, horizontally scaled
> cache where multi-threaded throughput matters.

## HTTP caching

```
Cache-Control: public, max-age=3600, stale-while-revalidate=60
ETag: "a1b2c3"
```

Client sends `If-None-Match: "a1b2c3"`; server replies **304 Not Modified**
with no body.

| Directive | Meaning |
|-----------|---------|
| `public` / `private` | Shared caches allowed / browser only |
| `max-age` | Seconds fresh |
| `no-cache` | Revalidate before using (still stores) |
| `no-store` | **Never store** — use for sensitive data |
| `immutable` | Never revalidate — for content-hashed assets |
| `stale-while-revalidate` | Serve stale while refreshing |

## Practice

1. Why delete a cache key on write instead of updating it?
2. Describe the write-invalidate race and two mitigations.
3. A product page gets 50k req/s. The key expires and the database collapses.
   Name three independent fixes.
4. Your LRU cache hit rate drops every night. What is the likely cause?
5. An attacker requests random non-existent user IDs. What breaks, and how do
   you defend?
6. One celebrity account is 30% of reads on a single Redis node. Options?
7. When would you accept write-behind caching?
8. A page shows account balance and a product catalogue. Caching strategy for
   each?

<details><summary>Answers</summary>

1. Updating races with concurrent readers and can leave a **permanently** stale
   value; deleting forces the next read to repopulate from the source of truth.
2. A reader fetches old data, the writer updates and invalidates, then the
   reader writes its stale value back. Mitigate with short TTL as a backstop,
   delayed double-delete, versioned keys, or single-writer ownership.
3. (i) **Single-flight/lock** so only one request refills; (ii)
   **stale-while-revalidate**; (iii) **jittered TTLs** so keys do not expire
   together. Bonus: a circuit breaker protecting the DB.
4. A nightly **batch job or analytics scan** touches every row, evicting the
   hot set under LRU. Use LFU/segmented LRU, or run the job against a replica.
5. **Cache penetration** — every request misses and hits the DB. Cache the
   negative result with a short TTL, and/or front it with a **Bloom filter**.
6. Replicate the key under several suffixes and pick randomly; add an
   in-process cache for that key; or move it to a dedicated node. Hot keys
   cannot be fixed by adding shards alone.
7. Only when losing recent writes is acceptable — view counts, likes,
   analytics. Never for money or orders.
8. Balance: do not cache, or cache per-user with very short TTL and explicit
   invalidation on transaction. Catalogue: cache aggressively with minutes-long
   TTL at the CDN and in Redis; invalidate on publish.
</details>

---

Previous: [03 — Load balancing and gateways](03-load-balancing-and-gateways.md) ·
Next: [05 — Databases](05-databases.md)
