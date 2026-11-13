# 01 — Fundamentals and estimation

Everything in this guide rests on numbers. Learn these first and every later
decision becomes arguable rather than arbitrary.

## Latency numbers

The famous table, rounded to what you need to remember:

| Operation | Time | Relative |
|-----------|------|----------|
| L1 cache reference | 1 ns | 1 |
| Branch mispredict | 3 ns | |
| L2 cache reference | 4 ns | |
| Mutex lock/unlock | 17 ns | |
| **Main memory reference** | **100 ns** | 100× L1 |
| Compress 1 KB (Zippy) | 2 μs | |
| Send 1 KB over 1 Gbps | 10 μs | |
| **SSD random read** | **16 μs** | 160× RAM |
| Read 1 MB sequentially from memory | 100 μs | |
| Round trip within a datacentre | **500 μs** | |
| Read 1 MB sequentially from SSD | 1 ms | |
| **Disk seek (HDD)** | **2 ms** | |
| Read 1 MB sequentially from disk | 20 ms | |
| **Round trip CA → Netherlands** | **150 ms** | |

**What to actually memorise:**

```
Memory      ~100 ns      (fast)
SSD         ~100 μs      (1,000× slower than memory)
Disk seek   ~10 ms       (100× slower than SSD)
Same-DC RTT ~0.5 ms
Cross-continent RTT ~150 ms   (speed of light — cannot be optimised away)
```

> **Interview tip.** The ratios matter more than the absolutes. "Memory is
> ~1000× faster than SSD, which is ~100× faster than a disk seek" justifies
> caching without needing exact figures.

> **Trap.** Cross-continent latency is physics. No amount of engineering makes
> a London→Sydney round trip fast. The only answer is to **put data closer to
> users** — CDN, read replicas, regional deployments.

## Powers of two

| Power | Bytes | Name | Short |
|-------|-------|------|-------|
| 10 | 1,024 | Kilobyte | 1 KB |
| 20 | 1,048,576 | Megabyte | 1 MB |
| 30 | ~10⁹ | Gigabyte | 1 GB |
| 40 | ~10¹² | Terabyte | 1 TB |
| 50 | ~10¹⁵ | Petabyte | 1 PB |

Useful conversions:

```
1 day    = 86,400 s  ≈ 10⁵ s
1 month  ≈ 2.6M s    ≈ 2.5 × 10⁶ s
1 year   ≈ 31.5M s   ≈ 3 × 10⁷ s
```

> **Shortcut.** 1 request/second for a year ≈ 30 million requests.
> 1 KB/s for a year ≈ 30 GB.

## Back-of-envelope estimation

The single most important interview skill. The method:

```
1. Users           -> DAU (daily active users)
2. Actions/user    -> requests per day
3. Divide by 10⁵   -> average requests per second (QPS)
4. Multiply by 2-10 -> peak QPS
5. Bytes per item  -> storage per day, per year
6. Add replication -> actual storage
```

### Worked example — a Twitter-like feed

**Assumptions** (state them out loud; interviewers care that you do):

- 300 M monthly active users, 50% daily active → **150 M DAU**
- Each user posts 2 tweets/day → 300 M tweets/day
- Each user reads their feed 10 times/day → 1.5 B reads/day
- Tweet: 300 bytes text + metadata ≈ **1 KB**
- 10% of tweets carry a 200 KB image

**Write QPS**

```
300 M / 86,400 ≈ 3,500 writes/s
Peak (×3)      ≈ 10,000 writes/s
```

**Read QPS**

```
1.5 B / 86,400 ≈ 17,000 reads/s
Peak (×3)      ≈ 50,000 reads/s
```

→ **Read:write ratio ≈ 5:1.** That one number drives the whole design: cache
aggressively, use read replicas, consider fan-out-on-write.

**Storage**

```
Text:   300 M × 1 KB      = 300 GB/day  → ~110 TB/year
Images: 30 M × 200 KB     = 6 TB/day    → ~2.2 PB/year
With 3× replication:      → ~7 PB/year
```

→ Images dominate. They belong in **object storage behind a CDN**, not in the
database.

**Bandwidth**

```
Writes: 300 GB/day ÷ 86,400 ≈ 3.5 MB/s   (plus images ≈ 70 MB/s)
Reads:  17,000 × 1 KB        ≈ 17 MB/s   (plus images, much more)
```

**Memory for cache**

Apply the 80/20 rule — 20% of tweets serve 80% of reads:

```
Daily reads of distinct tweets ≈ 300 M
20% hot                        ≈ 60 M × 1 KB = 60 GB
```

→ Fits comfortably in a small Redis cluster. **This is a concrete, defensible
number**, and it is what separates a good answer from hand-waving.

### Estimation rules of thumb

| Thing | Rule |
|-------|------|
| DAU from MAU | 20–50% |
| Peak vs average | 2–10× (use 3 unless told otherwise) |
| Read:write | 10:1 for social, 100:1 for content sites, 1:1 for logging |
| Cache hit rate | 80–95% for hot data |
| Replication factor | 3 |
| One modern server | ~10k QPS simple, ~1k QPS with DB work |
| One SQL instance | ~5–10k reads/s, ~1–5k writes/s |
| One Redis instance | ~100k ops/s |
| One Kafka broker | ~100 MB/s |

> **Interview tip.** Round aggressively. 86,400 → 10⁵. 1 million users × 1 KB →
> 1 GB. Nobody wants four significant figures; they want to see you reason.

## Scalability vs performance

| | Performance | Scalability |
|---|---|---|
| Question | Is it fast for one user? | Does it stay fast as load grows? |
| Fix | Better algorithm, caching, indexes | More machines, sharding, async |

A system can be fast and unscalable (single server, great code) or scalable and
slow (massive cluster, bad queries).

### Vertical vs horizontal scaling

| | Vertical (scale up) | Horizontal (scale out) |
|---|---|---|
| How | Bigger machine | More machines |
| Ceiling | **Hard physical limit** | Effectively none |
| Complexity | Low | High — needs LB, distributed state |
| Failure | **Single point of failure** | Tolerant by design |
| Cost curve | Superlinear | Roughly linear |

> **Interview tip.** Start with vertical and a single database — it is the
> right answer for most real systems. Scale horizontally only when you can
> name the bottleneck. Jumping straight to microservices and sharding is a
> common and visible mistake.

## Availability

**Nines**, as allowed downtime:

| Availability | Per year | Per month | Per day |
|---|---|---|---|
| 99% | 3.65 days | 7.2 h | 14.4 min |
| 99.9% ("three nines") | 8.77 h | 43.8 min | 1.44 min |
| 99.99% | 52.6 min | 4.4 min | 8.6 s |
| 99.999% ("five nines") | 5.26 min | 26 s | 864 ms |

### Composing availability

**In series** (both must work) — multiply:

```
99.9% × 99.9% = 99.8%
```

More dependencies means lower availability. A service calling five 99.9%
dependencies is at best 99.5%.

**In parallel** (either works) — multiply the failure probabilities:

```
1 - (0.001 × 0.001) = 99.9999%
```

> **Interview tip.** This is why redundancy matters and why reducing
> synchronous dependencies matters. If an answer adds a sixth service to the
> critical path, say so.

### Related terms

- **Reliability** — probability of working correctly for a period
- **Durability** — probability data is not lost (S3: 11 nines)
- **MTBF** — mean time between failures
- **MTTR** — mean time to recovery. **Reducing MTTR is usually easier and
  cheaper than increasing MTBF.**

## Latency percentiles

Averages lie. Use percentiles.

```
p50 = median            — typical experience
p95, p99 = tail         — what slow users feel
p999 = extreme tail     — matters at scale
```

> **Trap.** At 100 requests per page load, a p99 of 1 s means **most page
> loads** contain at least one slow call. Tail latency compounds with fan-out.

> **Interview tip.** Say "p99 under 200 ms" rather than "fast". Specifying the
> percentile signals experience.

## Throughput, latency, concurrency

**Little's Law:**

$$ L = \lambda \times W $$

Concurrency = arrival rate × time in system.

```
1,000 req/s × 0.2 s latency = 200 concurrent requests
```

→ You need a thread/connection pool of at least 200. This is how you size pools
rather than guessing.

## CAP, briefly

Covered properly in [chapter 07](07-consistency-and-consensus.md), but the
one-liner:

> When the network partitions, you must choose **consistency** or
> **availability**. You cannot have both.

Partitions are not optional — networks fail. So every distributed system makes
this choice, explicitly or by accident.

## The design hierarchy

When something is too slow, work down this list — cheapest first:

```
1. Measure         — do not guess
2. Algorithm/query — O(n²) → O(n log n), add an index
3. Cache           — avoid the work entirely
4. Vertical scale  — bigger machine
5. Read replicas   — scale reads
6. Async           — queue the slow part
7. Horizontal      — shard, partition
8. Rearchitect     — different data model or storage engine
```

> **Interview tip.** Walking down this list out loud demonstrates judgement.
> Starting at step 8 demonstrates the opposite.

## Practice

1. Estimate storage and QPS for a photo-sharing app: 500 M MAU, 30% DAU,
   2 uploads/day, average photo 2 MB, 50 views per upload.
2. A service calls four dependencies synchronously, each 99.95% available.
   What is the best-case availability? How would you improve it?
3. A service handles 5,000 req/s with p99 latency of 400 ms. How many
   concurrent requests are in flight? How large should the thread pool be?
4. Your database does 8,000 reads/s and is at capacity. List three things to
   try, in order, before sharding.
5. How much Redis memory for an 80% hit rate if 200 M distinct items of 500
   bytes are read daily and 20% are hot?

<details><summary>Answers</summary>

1. 150 M DAU × 2 = 300 M uploads/day ÷ 10⁵ ≈ **3,500 writes/s** (peak ~10k).
   Storage 300 M × 2 MB = **600 TB/day** — enormous, so object storage + CDN,
   and consider compression/resizing. Views: 300 M × 50 = 15 B/day ≈
   **170k reads/s** — must be CDN-served.
2. 0.9995⁴ ≈ **99.8%**. Improve by making calls parallel (doesn't help
   availability), removing them from the critical path (async), adding
   caching/fallbacks, or making them optional with graceful degradation.
3. L = 5,000 × 0.4 = **2,000 concurrent**. Pool should exceed that, with
   headroom — plus a queue and a bulkhead so overload is shed, not queued
   forever.
4. (i) Find the slow query and add an index; (ii) cache hot reads; (iii) add
   read replicas. Then vertical scale. Shard last.
5. 200 M × 20% × 500 B = **20 GB**. Fits one large Redis node, so no cluster
   needed yet.
</details>

---

Next: [02 — Networking and protocols](02-networking-and-protocols.md)
