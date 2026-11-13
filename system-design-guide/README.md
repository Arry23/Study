# System Design, A to Z — HLD and LLD

A complete course covering both halves of system design:

- **HLD (High-Level Design)** — distributed systems. How do you serve a million
  users, keep data consistent across machines, and stay up when things fail?
- **LLD (Low-Level Design)** — object-oriented design. Given one service, how do
  you structure the classes, interfaces and concurrency so it stays
  maintainable?

Interviews test them separately, and good engineers need both. A beautiful
class hierarchy inside a system that cannot scale is useless; a perfectly
scalable architecture made of unmaintainable code is worse.

## Chapters

### Foundations

| # | Chapter | Covers |
|---|---------|--------|
| 01 | [Fundamentals and estimation](01-fundamentals-and-estimation.md) | Latency numbers, scalability vs performance, availability math, back-of-envelope |

### HLD — building blocks

| # | Chapter | Covers |
|---|---------|--------|
| 02 | [Networking and protocols](02-networking-and-protocols.md) | TCP/UDP, HTTP/1.1–3, TLS, WebSockets, gRPC, DNS |
| 03 | [Load balancing and gateways](03-load-balancing-and-gateways.md) | L4 vs L7, algorithms, reverse proxies, API gateways, service mesh |
| 04 | [Caching](04-caching.md) | Layers, eviction, invalidation, patterns, stampedes |
| 05 | [Databases](05-databases.md) | SQL vs NoSQL, indexes, B-trees vs LSM, transactions, isolation |
| 06 | [Scaling data](06-scaling-data.md) | Replication, sharding, consistent hashing, hot keys |
| 07 | [Consistency and consensus](07-consistency-and-consensus.md) | CAP, PACELC, consistency models, Raft, quorums, vector clocks |
| 08 | [Messaging and streaming](08-messaging-and-streaming.md) | Queues, pub/sub, Kafka, delivery semantics, outbox, saga |
| 09 | [Storage and search](09-storage-and-search.md) | Object/block/file, blob stores, inverted indexes, ranking |
| 10 | [Reliability patterns](10-reliability-patterns.md) | Timeouts, retries, circuit breakers, bulkheads, rate limiting |
| 11 | [Observability and SLOs](11-observability-and-slos.md) | Logs, metrics, traces, SLI/SLO/error budgets |
| 12 | [Security in design](12-security-design.md) | AuthN/AuthZ, OAuth2/OIDC/JWT, encryption, threat modelling |
| 13 | [Architecture styles](13-architecture-styles.md) | Monolith, microservices, event-driven, serverless, CQRS, hexagonal |
| 14 | [Deployment and operations](14-deployment-and-operations.md) | Containers, orchestration, canary, multi-region, DR |

### HLD — practice

| # | Chapter | Covers |
|---|---------|--------|
| 15 | [The HLD interview framework](15-hld-framework.md) | How to drive 45 minutes without freezing |
| 16 | [Case studies: core](16-hld-case-studies-core.md) | URL shortener, rate limiter, news feed, chat |
| 17 | [Case studies: advanced](17-hld-case-studies-advanced.md) | Video streaming, ride sharing, payments, autocomplete, metrics |

### LLD

| # | Chapter | Covers |
|---|---------|--------|
| 18 | [OOP foundations and UML](18-oop-and-uml.md) | Encapsulation, composition, coupling, class and sequence diagrams |
| 19 | [SOLID and design principles](19-solid-and-principles.md) | SOLID, DRY, KISS, YAGNI, Law of Demeter |
| 20 | [Design patterns](20-design-patterns.md) | Creational, structural, behavioural — with working code |
| 21 | [API and interface design](21-api-design.md) | REST, gRPC, GraphQL, versioning, idempotency, pagination |
| 22 | [Concurrency](22-concurrency.md) | Threads, locks, deadlock, producer-consumer, thread pools |
| 23 | [LLD worked problems](23-lld-worked-problems.md) | Parking lot, elevator, LRU cache, rate limiter, booking |

### Practice

| # | Chapter | Covers |
|---|---------|--------|
| 24 | [Practice problems and labs](24-practice-and-labs.md) | **40+ problems and hands-on builds** |
| 25 | [Cheat sheet](25-cheatsheet.md) | One page to keep open |

## How to use this

**If you are preparing for interviews**, the order that works:

| Week | Do |
|------|-----|
| 1 | 01–05. You cannot design anything without estimation and databases |
| 2 | 06–10. The distributed-systems core |
| 3 | 11–14, then 15. Context, then the framework |
| 4 | 16–17, out loud, on a whiteboard, timed |
| 5 | 18–22. LLD fundamentals |
| 6 | 23, then 24 — write real code |

**If you are designing a real system**, read 15 first, then dip into the
building-block chapters for the decisions you face.

**If you only have a weekend**, read 01, 15, 25, and work two case studies
from 16 out loud.

## The two things that matter most

**There is no correct answer — only defensible trade-offs.** Every chapter
frames decisions as trade-offs, because that is what an interviewer is
listening for. "It depends" is a bad answer; "it depends on X, and here is how
I would decide" is the right one.

**Numbers beat adjectives.** "A lot of traffic" tells nobody anything.
"50k writes/second, 2 KB each, so 100 MB/s, so ~8 TB/day" drives the entire
design. Chapter 01 makes this a reflex.

## Conventions

- **Trade-off** marks a decision with no universally right answer.
- **Interview tip** marks something interviewers specifically listen for.
- **Trap** marks a common mistake.
- Code is Python for readability; Java/C++ notes appear where idioms differ.
