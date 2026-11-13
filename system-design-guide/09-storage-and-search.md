# 09 — Storage and search

Two topics that appear in almost every design: where do large files live, and
how do users find things.

## Storage types

| | **Block** | **File** | **Object** |
|---|---|---|---|
| Unit | Fixed-size blocks | Files in a hierarchy | Objects with metadata |
| Access | Raw device | POSIX, NFS/SMB | **HTTP API** |
| Mutable | Yes, in place | Yes | **Usually replace-whole-object** |
| Scale | TBs | TBs–PBs | **Effectively unlimited** |
| Latency | Lowest | Low | Higher (network, HTTP) |
| Use | Databases, boot volumes | Shared app data, home dirs | Media, backups, data lake |

> **Interview tip.** Never put large binaries in a relational database. Store
> the object in blob storage and keep the **key and metadata** in the database.
> Proposing BLOB columns for images is a visible mistake.

## Designing a blob storage layer

```
Client ──(1) request upload URL──► API ──► generates presigned URL
Client ──(2) PUT directly────────────────► Object store
Client ──(3) notify complete────► API ──► store metadata in DB
                                            │
                                     (4) event → thumbnailer, virus scan
```

Key decisions:

**Direct upload with presigned URLs.** The file never transits your servers,
so you do not pay for bandwidth twice and your API stays small. Mention this —
it is the standard answer for "users upload photos".

**Metadata in a database, bytes in object storage.** The database row holds
`id, owner, key, size, content_type, created_at, status`.

**Content-addressed keys.** Naming an object by the hash of its content gives
free deduplication and makes caching trivial (`immutable` forever).

**Multipart upload** for large files — parallel parts, resumable, required
above a few GB.

**Lifecycle policies** — move to colder tiers, expire old versions, abort
incomplete uploads. See the cost chapter of any cloud guide; the same idea
applies everywhere.

### Serving

```
Client → CDN → (signed URL check) → Object store (private)
```

Keep the bucket private and serve through a CDN with **signed URLs** or
**signed cookies** for access control. Set long cache lifetimes on
content-hashed keys.

## Deduplication and chunking

For backup or sync systems (Dropbox-style):

```
File → split into chunks (fixed or content-defined) → hash each chunk
     → upload only chunks the server does not have
```

- **Fixed-size chunking** is simple but one inserted byte shifts everything
- **Content-defined chunking (rolling hash)** finds boundaries by content, so
  an insertion only changes one chunk

> **Interview tip.** Content-defined chunking is the right answer for a file
> sync service. It is why editing the middle of a 1 GB file uploads kilobytes,
> not gigabytes.

## Search

### Why a database `LIKE` is not search

```sql
SELECT * FROM posts WHERE body LIKE '%distributed%';
```

Full table scan, no ranking, no stemming, no typo tolerance, no phrase or
proximity matching.

### Inverted index

The core data structure.

```
Documents:
  d1: "the quick brown fox"
  d2: "the quick blue car"

Inverted index:
  the    → [d1, d2]
  quick  → [d1, d2]
  brown  → [d1]
  fox    → [d1]
  blue   → [d2]
  car    → [d2]
```

Query `quick AND fox` → intersect `[d1,d2]` with `[d1]` → **d1**.

**The pipeline:**

```
Document → tokenise → lowercase → remove stopwords → stem/lemmatise
        → index (term → posting list with positions and frequencies)
```

- **Stemming** — "running" → "run" (fast, crude)
- **Lemmatisation** — "better" → "good" (slower, accurate)
- **Positions** enable phrase queries
- **n-grams** enable substring and typo-tolerant matching

### Ranking

**TF-IDF** — a term matters more if it is frequent in this document and rare
across the corpus.

$$ \text{tf-idf}(t,d) = \text{tf}(t,d) \times \log\frac{N}{\text{df}(t)} $$

**BM25** is the modern refinement (saturating term frequency, length
normalisation) and is the default in Elasticsearch.

**Beyond text relevance**, real ranking blends: freshness, popularity,
personalisation, click-through rate, and business rules. Usually a learned
model (learning to rank).

> **Interview tip.** Separate **matching** (which documents qualify) from
> **ranking** (what order). Candidates who conflate them struggle with
> follow-ups like "how do you make recent posts rank higher?"

### Elasticsearch / OpenSearch essentials

| Concept | Meaning |
|---------|---------|
| **Index** | Collection of documents (like a table) |
| **Shard** | A Lucene index; unit of scale. **Fixed at creation** |
| **Replica** | Copy of a shard — availability and read throughput |
| **Mapping** | Field types and analysers |
| **Analyser** | The tokenise→filter pipeline |
| **Refresh interval** | How soon writes become searchable (default 1 s) |

> **Trap.** Elasticsearch is **near real-time**, not real-time — documents are
> searchable after a refresh. And it is not a primary datastore: keep the
> source of truth elsewhere and index into it.

**Search is a derived view.** The canonical pattern:

```
Postgres (source of truth) ──CDC/outbox──► Kafka ──► indexer ──► Elasticsearch
```

This reuses the outbox pattern from
[chapter 08](08-messaging-and-streaming.md) and is the standard answer for
"how do you keep search in sync?"

## Autocomplete

A frequent interview question of its own.

| Approach | Notes |
|----------|-------|
| **Trie / prefix tree** | Classic; store top-K completions at each node |
| **Finite state transducer** | Compressed trie — what Lucene actually uses |
| **n-gram index** | Handles mid-word matches |
| Precomputed top-K per prefix in Redis | Simple, fast, easy to update |

```
Trie node "ca"
 └── top-K cached here: ["cat", "car", "cake", "cash", "cast"]
```

> **Interview tip.** Precompute top-K completions **at each node** so a lookup
> is one traversal plus a read, not a subtree scan. Update them from a batch
> job over query logs — autocomplete does not need to be real-time.

## Geospatial indexing

For "find things near me":

| Method | How |
|--------|-----|
| **Geohash** | Encode lat/long into a string; shared prefix = nearby |
| **Quadtree** | Recursive subdivision, adapts to density |
| **S2 / H3** | Sphere-aware cells (Google S2, Uber H3) |
| **R-tree** | Bounding boxes; what PostGIS uses |

> **Trap — the geohash edge problem.** Two points either side of a cell
> boundary can be physically adjacent but share no prefix. Always query the
> cell **and its 8 neighbours**.

## Time series storage

Characteristics: append-heavy, time-ordered, rarely updated, queried by range,
and downsampled as it ages.

Optimisations: columnar layout, delta-of-delta timestamp encoding,
compression, automatic **rollups** (1 s → 1 m → 1 h), and retention tiers.

## Decision table

| Requirement | Answer |
|-------------|--------|
| User-uploaded photos/videos | **Object storage** + CDN, presigned uploads |
| Shared filesystem across servers | **File storage** (NFS/EFS) |
| Database storage | **Block storage** |
| Full-text search with ranking | **Elasticsearch** (inverted index + BM25) |
| Keep search in sync with the DB | **CDC/outbox → indexer** |
| Search-as-you-type | **Trie with cached top-K** |
| "Restaurants within 2 km" | **Geohash / S2 / PostGIS** |
| Metrics for a year | **Time series DB** with rollups |
| Dedup a backup system | **Content-defined chunking** |
| Serve private media | CDN + **signed URLs** |

## Practice

1. Why is storing images as database BLOBs a bad idea? Give three reasons.
2. Design the upload path for a photo app, minimising bandwidth through your
   servers.
3. Build the inverted index for: d1 "big red car", d2 "red car fast", d3 "big
   truck". Which documents match `big AND car`?
4. How do you keep Elasticsearch in sync with PostgreSQL? What breaks with the
   naive approach?
5. Why is fixed-size chunking bad for a file sync service?
6. Design autocomplete for 10 M queries/day with p99 under 50 ms.
7. What is the geohash edge problem and how do you handle it?
8. Search results must favour recent posts. Where in the pipeline do you do
   that?

<details><summary>Answers</summary>

1. It bloats the database and its backups; it wastes expensive, replicated
   database storage on bytes needing none of it; it cannot be served by a CDN;
   and it hurts the buffer pool/cache hit rate for real query data.
2. API issues a **presigned URL**; client PUTs directly to object storage;
   client notifies the API, which writes metadata; an **event** triggers
   thumbnailing and scanning. Bytes never pass through your servers.
3. `big → [d1,d3]`, `red → [d1,d2]`, `car → [d1,d2]`, `fast → [d2]`,
   `truck → [d3]`. `big AND car` = intersect `[d1,d3]` and `[d1,d2]` → **d1**.
4. Use **CDC or the outbox pattern** to publish changes to Kafka and let an
   indexer apply them. The naive approach — writing to both in application
   code — is the **dual-write problem**: a crash between them silently
   desynchronises search from truth.
5. Inserting one byte near the start shifts every subsequent boundary, so
   every chunk hashes differently and the whole file re-uploads.
   **Content-defined chunking** sets boundaries by content, so only the
   affected chunk changes.
6. Trie (or FST) with **precomputed top-K at each node**, served from memory
   or Redis; update the structure from an offline job over query logs; put it
   behind a CDN/edge cache for popular prefixes. 10 M/day ≈ 120 QPS average —
   trivially served; the latency comes from keeping it in memory.
7. Adjacent points can fall in different cells and share no prefix, so a
   prefix-only query misses them. Query the target cell **plus its eight
   neighbours**, then filter by true distance.
8. In **ranking**, not matching. Combine the BM25 relevance score with a
   recency decay (and popularity) into a final score — keep the matching stage
   unchanged so you do not lose results.
</details>

---

Previous: [08 — Messaging and streaming](08-messaging-and-streaming.md) ·
Next: [10 — Reliability patterns](10-reliability-patterns.md)
