# 17 — Case studies: advanced

Five designs that go beyond the standard CRUD-plus-cache shape. Each one has a
distinct *hard part* — find it before you start drawing.

| Problem | The hard part |
|---------|---------------|
| Video streaming | Petabytes of blobs; transcoding; delivery cost |
| Ride sharing | **Geospatial** indexing and real-time matching |
| Payments | **Exactly-once** money movement; auditability |
| Autocomplete | Sub-50 ms p99 on every keystroke |
| Metrics system | Enormous write volume; **time-series** storage |

---

# 1. Video streaming (YouTube/Netflix)

## Requirements

**Functional:** upload a video; transcode to multiple resolutions; stream with
adaptive quality; search and recommend; view counts. **Out of scope:**
comments, monetisation, live streaming (mention it as an extension).

**Non-functional:** playback must start in **under 2 seconds** and not
rebuffer; uploads may take minutes (async is fine); read:write is extreme —
millions of views per upload; global audience.

## Estimation

```
500 hours uploaded/minute      → 30,000 hours/hour
1 hour of 1080p source ≈ 2 GB  → 60 TB/hour ingested → 1.4 PB/day

Transcoding multiplies: 5 renditions ≈ 1.5× source  → ~2 PB/day stored
Playback: 5 M concurrent viewers × 5 Mbps = 25 Tbps egress
```

> **Interview tip.** Say the egress number out loud. **25 Tbps** is the single
> fact that forces the design: no origin fleet serves that, so **CDN is not an
> optimisation, it is the architecture**. Bandwidth, not storage or compute, is
> the dominant cost.

## Upload and transcode pipeline

```
Client ──pre-signed URL──► Blob store (raw)
                                │
                          upload event
                                │
                        ┌───────▼────────┐
                        │ Transcode      │  split into chunks (e.g. 10 s GOPs)
                        │ orchestrator   │  → parallel worker fleet
                        └───────┬────────┘
                                │
          240p 360p 480p 720p 1080p 4K  +  thumbnails + captions
                                │
                        Packager (HLS/DASH manifests)
                                │
                        Blob store (processed) ──► CDN
                                │
                        Metadata DB: status = READY
```

> **Interview tip.** Two details score well here:
>
> 1. **Pre-signed URLs** — the client uploads **directly** to object storage.
>    Never proxy 2 GB files through your API servers.
> 2. **Chunked parallel transcoding** — split the video into independent
>    segments, transcode them in parallel across hundreds of workers, then
>    stitch. A 1-hour video takes minutes instead of an hour.

**Pipeline as a DAG**, not a monolith: validate → split → transcode (fan-out)
→ package → thumbnail → publish. Each step is idempotent and retryable; a
failed segment re-runs alone.

## Adaptive bitrate streaming

The video is stored as **segments** (2–10 s) at each quality, described by a
manifest:

```
master.m3u8
 ├─ 1080p/index.m3u8 → seg0.ts, seg1.ts, ...
 ├─ 720p/index.m3u8  → seg0.ts, seg1.ts, ...
 └─ 480p/index.m3u8  → ...
```

The **client** measures throughput and buffer level and chooses the next
segment's quality. The server is dumb; it just serves files over HTTP — which
is precisely why a CDN can cache it.

> **Trap.** Candidates often reach for WebSockets or a custom UDP protocol for
> video. Real ABR streaming is **plain HTTP GETs of static files**. That is the
> whole trick: it makes video cacheable at the edge.

## Delivery

| Layer | Holds |
|-------|-------|
| **Edge CDN** | The ~5% of content that is 95% of views |
| **Regional cache** | Mid-tail content |
| **Origin (blob store)** | Everything |

Popularity follows a power law, so a modest edge cache gives a very high hit
rate. Pre-push anticipated hits (a new release) to the edge before launch.

## Metadata and views

```
videos(video_id PK, uploader_id, title, status, duration, created_at)
view_counts → approximate, incremented via a stream, not a DB UPDATE
```

> **Interview tip.** Never `UPDATE videos SET views = views + 1` per view —
> that is a hot row and a write bottleneck. Emit a view **event** to Kafka,
> aggregate in a stream processor, and write counts periodically. Exact counts
> are not required; **timeliness and cost** are.

## Deep dives

**Resumable uploads.** Chunk the upload client-side; the server tracks which
chunks landed. A dropped connection resumes rather than restarting.

**Deduplication.** Hash the content; identical uploads reuse the existing
transcode output.

**Cost control.** Transcode lazily for rare renditions (4K on demand), store
cold videos in cheaper tiers, and delete renditions nobody requests.

**Live streaming** (if asked): the same pipeline with low-latency chunks
(1–2 s), transcoding in real time, and a much shallower buffer — you trade
latency against rebuffering.

---

# 2. Ride sharing (Uber/Lyft)

## Requirements

Riders request a ride from A to B; nearby drivers are found and matched;
both see live location; fare is computed. Location updates every ~4 seconds
from every active driver. Matching should take **seconds**, and a
double-assigned driver is unacceptable.

## Estimation

```
1 M active drivers, location every 4 s → 250,000 writes/s
1 M rides/day peaking at ~50/s matched

Location record ≈ 50 bytes → 12 MB/s → ~1 TB/day if all retained
```

> **Interview tip.** 250k writes/s of location is the forcing number. It says:
> **do not put live location in your relational database.** Keep current
> position in an in-memory geospatial store, and stream the history to
> cold storage separately.

## The hard part: "find drivers near me"

A naive `WHERE lat BETWEEN ... AND lng BETWEEN ...` cannot be indexed well in
two dimensions. Three real options:

| Approach | How it works | Notes |
|----------|--------------|-------|
| **Geohash** | Interleave lat/lng bits into a base-32 string; shared prefix = nearby | Simple, string-prefix searchable, **edge problems** at cell boundaries |
| **Quadtree** | Recursively subdivide space until each cell has ≤ N drivers | Adapts to density; in-memory; rebuild cost |
| **S2 / H3** | Sphere-aware cells (Google S2, Uber's H3 hexagons) | **Production answer**; uniform neighbours, no boundary pathologies |

```
Geohash "9q8yy" ≈ a ~5 km cell
Query: driver's cell + its 8 neighbours → candidates → exact distance filter
```

> **Trap — the boundary problem.** Two drivers 10 metres apart can sit in
> different geohash cells with *no shared prefix*. You must always search the
> **neighbouring cells**, not just your own. Hexagons (H3) are popular partly
> because every neighbour is equidistant, which makes this cleaner.

## Design

```
Driver app ──location every 4s──► Location service ──► Redis (GEOADD, in-memory)
                                        └──► Kafka ──► history store (cold)

Rider app ──request ride──► Matching service
                                 │ 1. geo query: candidate drivers in cell + neighbours
                                 │ 2. rank: ETA, rating, acceptance rate
                                 │ 3. offer to best driver, lock with TTL
                                 │ 4. on accept → create trip (transactional)
                                 │    on timeout → next driver
                                 ▼
                            Trip service ──► Trip DB (SQL, strongly consistent)
                                 │
                            Pub/sub ──► live location to the rider
```

> **Interview tip.** Split the two halves explicitly:
>
> - **Location** — massive write volume, ephemeral, **eventual consistency is
>   fine**, lives in memory.
> - **Trips and payments** — low volume, must be **strongly consistent and
>   transactional**, lives in SQL.
>
> Recognising that one system has two very different consistency regimes is
> exactly the judgement being tested.

## Deep dives

**Preventing double assignment.** The offer takes a **distributed lock** on
the driver with a short TTL (say 15 s). Accepting converts it into a trip row
in a transaction; a timeout releases it. Alternatively, a conditional update
(`UPDATE drivers SET state='OFFERED' WHERE id=? AND state='AVAILABLE'`) gives
you the same guarantee with compare-and-set rather than a lock — simpler and
safer. Mention both.

**Surge pricing.** Compute demand/supply per cell on a stream, update a
multiplier every minute, and **lock the quoted price** at request time so the
fare cannot change under the rider.

**Matching is an optimisation problem.** Greedy nearest-driver is the baseline.
Batched matching over a few seconds lets you solve a small assignment problem
and reduce total ETA across all waiting riders — better globally, slightly
slower individually.

**Sharding.** Shard by **geography** (city/region). Rides are local, so
cross-shard traffic is nearly zero, and each city scales independently.

**Driver offline/crash.** Stale location entries expire by TTL; a driver with
no update for 30 s is removed from the candidate pool.

---

# 3. Payment system

## Requirements

Charge a customer; pay out a merchant; handle refunds; integrate external
payment processors. **Money must never be created or destroyed.** Every
movement must be **auditable**. The system must survive a crash at any point
without double-charging.

## The non-negotiables

| Property | Mechanism |
|----------|-----------|
| No double charge | **Idempotency keys** |
| No lost money | **Double-entry ledger** |
| Survives partial failure | **Saga** with compensations |
| Auditable | **Append-only, immutable** records |
| Reconcilable | Daily comparison with the processor |

## Double-entry ledger

Every transaction writes **at least two rows that sum to zero**.

```
ledger_entries(entry_id, transaction_id, account_id, amount, currency, created_at)

Charge £50 from customer C to merchant M:
   (txn_1, account=C,            amount = -50.00)
   (txn_1, account=M_receivable, amount = +50.00)
```

> **Interview tip.** Say "**append-only, never update, never delete**". A
> refund is a *new* pair of entries, not a mutation of the original. Balance is
> the sum of entries (materialised into a balance table for speed). This single
> constraint is what makes the system auditable, and interviewers in fintech
> listen specifically for it.

**Store money as integer minor units** (pence/cents) or a fixed-point decimal.
Never floating point — `0.1 + 0.2 != 0.3`.

## Idempotency

```
POST /payments
Idempotency-Key: 7f3c9b1e-...
{amount: 5000, currency: "GBP", source: "card_x"}
```

```python
def charge(key, request):
    existing = store.get(key)
    if existing:
        if existing.request_hash != hash(request):
            raise Conflict("key reused with different parameters")
        return existing.response          # replay the original result
    store.put(key, state="IN_PROGRESS", request_hash=hash(request))
    result = processor.charge(request)    # pass the same key downstream
    store.complete(key, result)
    return result
```

> **Trap.** Checking "has this key been seen?" and then inserting is a race —
> two concurrent retries both see nothing. The insert must be an **atomic
> conditional insert** (unique constraint on the key), and the in-progress
> state must be stored *before* calling the processor, so a crash mid-call is
> recoverable rather than ambiguous.

Pass the idempotency key **through to the external processor** too, so their
retry handling aligns with yours.

## Design

```
Client ──► Payment API ──► Idempotency store
                │
                ├──► Ledger service ──► Ledger DB (SQL, ACID)
                │
                ├──► Risk/fraud service (sync, fast rules; async deep scoring)
                │
                ├──► Processor adapters ──► Stripe / Adyen / bank rails
                │         (outbox → queue → workers, with retries)
                │
                └──► Webhook handler ◄── processor callbacks (async settlement)
```

**State machine**, persisted on every transition:

```
CREATED → AUTHORISED → CAPTURED → SETTLED
    │           │           │
    └──► FAILED └──► VOIDED └──► REFUNDED
```

> **Interview tip.** A payment is **not** a single synchronous call. Authorise
> and capture are separate, settlement is asynchronous and may arrive hours
> later via webhook, and the terminal state is confirmed by reconciliation.
> Modelling it as a durable state machine is the correct answer.

## Deep dives

**Outbox pattern.** Writing the ledger row and publishing the event must not
diverge. Write both in one transaction — the event into an `outbox` table —
and let a relay publish it ([chapter 08](08-messaging-and-streaming.md)).

**Saga for multi-step flows.** Debit customer → credit merchant → notify. If a
later step fails, run compensating transactions (a reversing ledger pair), not
a rollback — the earlier steps are already committed.

**Exactly-once, honestly.** There is no exactly-once delivery. There is
**at-least-once delivery plus idempotent processing**, which is
indistinguishable from exactly-once at the ledger. Say it that way.

**Reconciliation.** A nightly job compares your ledger against the processor's
settlement file and raises discrepancies. Assume drift will happen; detect it.

**Webhook security.** Verify the signature, treat delivery as at-least-once
(dedupe on event ID), and tolerate out-of-order arrival.

**Currency.** Store amount and currency together; never aggregate across
currencies without an explicit, recorded FX rate.

---

# 4. Autocomplete / typeahead

## Requirements

Return the top 5–10 completions for a prefix, **ranked by popularity**, in
**under 100 ms end-to-end** — so the service budget is more like 20 ms.
Suggestions may be minutes stale. Read volume is enormous: several queries per
search, one per keystroke.

## Estimation

```
10 M searches/day, ~20 chars each with debouncing → ~4 keystroke queries each
40 M prefix queries/day ≈ 500/s average, ~2,000/s peak

Vocabulary: 100 M distinct queries, but the useful head is far smaller
```

## The data structure: trie with cached top-K

A plain trie finds the prefix node quickly but then needs a subtree traversal
to rank completions — too slow. **Store the top K completions at each node.**

```
          (root)
            │ t
            t ── top: [twitter, twitch, target]
            │ w
           tw ── top: [twitter, twitch, twins]
            │ i
          twi ── top: [twitter, twitch, twilight]
```

Lookup becomes: walk `len(prefix)` nodes → **return the precomputed list**.
O(prefix length), no traversal, microseconds.

```python
class Node:
    __slots__ = ("children", "top")      # top: list[(score, term)], length K

def suggest(root, prefix, k=10):
    node = root
    for ch in prefix:
        node = node.children.get(ch)
        if node is None:
            return []
    return node.top[:k]
```

> **Interview tip.** The insight is **precompute at write time so reads are
> trivial** — the same trade as fan-out-on-write in
> [chapter 16](16-hld-case-studies-core.md). If you only say "use a trie", you
> have given half the answer; the cached top-K per node is the other half.

## Design

```
                      ┌─────────── Query logs (Kafka)
                      │
              Aggregation job (hourly/daily)
                      │  count, filter, rank
                      ▼
              Trie builder ──► serialised trie snapshot (blob store)
                      │
                      ▼
Client ──► CDN ──► Suggestion service (trie held in memory) ──► Redis fallback
```

**Build offline, serve from memory.** The trie is rebuilt periodically and
shipped to the serving fleet as an immutable snapshot — so serving nodes never
do writes, and you can roll back a bad build.

## Deep dives

**Memory.** 100 M terms is too much; keep the head (terms above a frequency
threshold) in memory, which covers the overwhelming majority of queries.
Shard the trie by first 1–2 characters across nodes if it still does not fit.

**Freshness vs cost.** Hourly rebuilds are usually fine. For trending terms,
layer a real-time stream that boosts a small overlay on top of the snapshot.

**Client-side cost control.** Debounce ~50 ms, cache results per prefix in the
browser, and cancel in-flight requests when the user types again. A large
share of load disappears without touching the backend.

**Personalisation.** Blend the global top-K with a small per-user history list
at request time. Keep personalisation out of the shared trie — it would
destroy cacheability.

**Safety.** Filter profanity and sensitive terms at **build** time, not query
time, and maintain a blocklist. Autocomplete is a reputational surface.

**Typo tolerance.** Exact-prefix only is the baseline. Fuzzy matching needs
edit-distance search (BK-tree, or an n-gram index in a search engine) — say it
is a separate, more expensive path.

---

# 5. Metrics / monitoring system

## Requirements

Ingest metrics from hundreds of thousands of hosts; store them; serve dashboard
queries over ranges from minutes to a year; evaluate alert rules continuously.
Ingest is write-dominated and **relentless**; a lost data point is tolerable, a
multi-hour outage is not.

## Estimation

```
100,000 hosts × 1,000 metrics each, every 10 s
→ 100,000 × 1,000 / 10 = 10 M data points/second

Naive row ≈ 100 bytes → 1 GB/s → 86 TB/day     ← impossible
Compressed time-series ≈ 1–2 bytes/point → ~1 TB/day
```

> **Interview tip.** That compression ratio is the heart of the question.
> Time-series data compresses extraordinarily well because timestamps are
> **regular** (delta-of-delta encoding) and values **change slowly**
> (XOR encoding, as in Facebook's Gorilla). Naming this is what separates a
> real answer from "use a database".

## Data model

```
metric: cpu.usage{host="web-01", region="eu-west", service="api"}
series = metric name + sorted label set  →  a unique series ID
points = (timestamp, value) appended to that series
```

Storage is **columnar per series**: all timestamps together, all values
together, in time-bucketed blocks.

> **Trap — cardinality.** Adding a label like `user_id` or `request_id`
> multiplies the number of series into the millions and destroys the system.
> **Cardinality, not data volume, is what kills time-series databases.** Say
> this out loud; it is the most common real-world failure and interviewers
> love it.

## Design

```
Agents (push)  ─┐
                ├─► Ingest gateway ──► Kafka ──► Writers ──► TSDB
Scrape (pull)  ─┘        (validate,              (batch,     (hot: SSD
                          cardinality             compress)   warm: disk
                          limits)                             cold: object store)
                                                        │
                            Query service ◄─────────────┤
                            │   downsampled rollups ────┘
                            ▼
                     Dashboards        Alert evaluator ──► Notifier
                                       (rules every 30 s)   (dedupe, group,
                                                             silence, escalate)
```

**Push vs pull:** pull (Prometheus-style) gives you free liveness detection and
central control of scrape intervals; push (StatsD-style) handles short-lived
jobs, NAT and serverless. Large systems do both.

## Deep dives

**Retention tiering.** Full resolution for ~2 weeks, 1-minute rollups for 3
months, 1-hour rollups for 2 years. **Downsample on write**, so a year-long
dashboard query reads thousands of points, not billions.

**Query path.** A dashboard asking for a year at 10-second resolution is the
query that kills you. Automatically select the rollup that matches the
requested pixel width — nobody can see more points than their screen has.

**Sharding.** Hash by **series ID** so all points of a series live together
(so a range query is one sequential read). Time-based partitioning on top makes
retention a cheap drop of old blocks.

**Alerting.** Rules evaluated on a schedule over recent data; an alert must be
`for: 5m` (sustained) to avoid flapping. Deduplicate and group so one incident
sends one page, not two hundred ([chapter 11](11-observability-and-slos.md)).

**The monitoring system must not depend on itself.** Run it in a separate
failure domain, and have an external dead-man's-switch that alerts if the
monitoring system goes quiet.

---

## Cross-cutting lessons

| Lesson | Where it showed up |
|--------|--------------------|
| Find the dominant cost first | Video: egress, not storage |
| Precompute for read-heavy paths | Autocomplete top-K; feed fan-out |
| Separate hot ephemeral from durable transactional | Ride sharing |
| Idempotency is the real "exactly-once" | Payments |
| Compression and downsampling beat scaling up | Metrics |
| Cardinality/fan-out is the hidden scaling limit | Metrics, feed |
| Clients do work too | ABR selection, debouncing, resumable upload |
| Build offline, serve immutable snapshots | Autocomplete trie |

## Practice

1. Video: a new release will be watched by 10 M people in the first hour. What
   do you do before launch, and why does lazy CDN filling fail here?
2. Ride sharing: two riders request the same driver in the same instant. Give
   two mechanisms that prevent double assignment, and say which you prefer.
3. Payments: a client retries a charge with the same idempotency key but a
   different amount. What happens, and why?
4. Payments: your service crashes after calling the processor but before
   recording the result. How do you recover without double-charging?
5. Autocomplete: how do you serve a trending term that did not exist at the
   last trie build?
6. Metrics: someone adds a `request_id` label to a high-volume metric. Describe
   the failure and the guardrail that prevents it.
7. Which of these five systems can tolerate losing a write, and which cannot?

<details><summary>Answers</summary>

1. **Pre-warm the edge caches** by pushing the renditions out before release.
   Lazy filling means the first requests in every region all miss
   simultaneously and stampede the origin — a thundering herd at terabit
   scale. Also pre-generate all renditions rather than transcoding on demand.
2. (a) A **distributed lock** on the driver with a short TTL held during the
   offer; (b) a **conditional update** —
   `UPDATE drivers SET state='OFFERED' WHERE id=? AND state='AVAILABLE'` —
   which succeeds for exactly one writer. Prefer the conditional update: it
   needs no extra system, cannot be lost by lock-service failure, and the
   database is already the source of truth.
3. Reject with **409 Conflict**. The stored request hash does not match, so
   this is not a retry — it is a different payment reusing a key, almost
   certainly a client bug. Replaying the original response would silently
   charge the wrong amount.
4. The idempotency record was written as `IN_PROGRESS` **before** the call, so
   on restart you find it and **query the processor by the same idempotency
   key** to discover the real outcome, then complete the record. Never blindly
   retry an unknown-state charge.
5. A **real-time overlay**: a small, frequently updated top-K list from the
   streaming aggregation, merged with the snapshot result at query time. The
   snapshot stays immutable and cheap; freshness comes from the thin layer on
   top.
6. Each distinct request ID creates a new **series**, so cardinality explodes
   from thousands to millions; the index no longer fits in memory, ingestion
   backs up and queries time out. Guardrail: enforce a **per-metric
   cardinality limit at the ingest gateway**, reject or drop offending labels,
   and alert the owning team. High-cardinality identifiers belong in traces or
   logs, not metrics.
7. **Tolerant:** metrics (a dropped point is invisible at dashboard
   resolution), autocomplete (a lost query log barely moves ranking), video
   view counts, driver location (another arrives in 4 seconds).
   **Not tolerant:** payments (ledger entries are absolute), trip creation,
   and video upload metadata. The rule: anything representing **money or a
   committed real-world commitment** must not lose writes; telemetry and
   ephemeral state may.
</details>

---

Previous: [16 — Case studies: core](16-hld-case-studies-core.md) ·
Next: [18 — OOP foundations and UML](18-oop-and-uml.md)
