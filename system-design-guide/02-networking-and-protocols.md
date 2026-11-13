# 02 — Networking and protocols

You rarely design a protocol, but you constantly choose one. Interviewers ask
"why WebSockets and not polling?" and expect a real answer.

## The layers that matter

```
Application   HTTP, gRPC, WebSocket, MQTT, DNS
Transport     TCP, UDP, QUIC
Network       IP
Link          Ethernet, Wi-Fi
```

## TCP vs UDP

| | **TCP** | **UDP** |
|---|---|---|
| Connection | Handshake (3-way) | None |
| Delivery | **Guaranteed, ordered** | Best effort, unordered |
| Congestion control | Yes | No |
| Overhead | Higher | Minimal |
| Head-of-line blocking | **Yes** | No |
| Use | Web, APIs, databases, file transfer | Video/voice, gaming, DNS, metrics |

> **Interview tip.** "Why UDP for video calls?" Because a dropped frame is
> better than a late frame. Retransmitting a 200 ms-old audio packet is
> useless; you would rather have a small glitch than growing delay.

**Head-of-line blocking** — TCP delivers in order, so one lost packet stalls
everything behind it. This is why HTTP/2 multiplexing over TCP still stalls,
and why HTTP/3 moved to QUIC over UDP.

## HTTP versions

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | **QUIC (UDP)** |
| Concurrency | 1 request per connection (6 per origin) | **Multiplexed streams** | Multiplexed, no HOL blocking |
| Headers | Plain text | **HPACK compressed** | QPACK compressed |
| Server push | No | Yes (deprecated) | No |
| Connection setup | TCP + TLS (2–3 RTT) | Same | **0–1 RTT** |

> **Interview tip.** HTTP/2 fixed application-layer head-of-line blocking but
> not transport-layer. HTTP/3 fixes both by abandoning TCP. Mention this and
> you have shown you understand the *why* of protocol evolution.

### Useful HTTP semantics

**Idempotency** — the same request repeated has the same effect.

| Method | Safe | Idempotent |
|--------|------|------------|
| GET, HEAD | yes | yes |
| PUT, DELETE | no | **yes** |
| **POST** | no | **no** |
| PATCH | no | usually not |

> **Interview tip.** This matters because **retries are only safe for
> idempotent operations**. "POST /payments" retried after a timeout can charge
> twice — hence idempotency keys ([chapter 21](21-api-design.md)).

**Status codes worth knowing:** 200, 201, 202 (accepted, async), 204, 301 vs
302, 304 (not modified — caching), 400, 401 vs 403, 404, 409 (conflict), 412
(precondition failed — optimistic concurrency), 429 (rate limited), 500, 502,
503, 504.

**Caching headers:** `Cache-Control`, `ETag` + `If-None-Match`,
`Last-Modified` + `If-Modified-Since`. See [chapter 04](04-caching.md).

## TLS

```
Client                                Server
  │── ClientHello (ciphers, SNI) ────►│
  │◄── ServerHello + certificate ─────│
  │── key exchange ──────────────────►│
  │◄──────── encrypted ──────────────►│
```

- TLS 1.3 is **1 RTT** (0-RTT on resumption); TLS 1.2 is 2 RTT
- **Termination** usually happens at the load balancer or CDN edge
- **mTLS** — both sides present certificates; the basis of zero-trust service
  meshes

> **Trade-off — terminate at the edge or end to end?** Terminating at the LB
> lets it inspect and route on L7, and reduces CPU on backends. Terminating
> end-to-end keeps traffic encrypted inside your network. Most designs
> terminate at the edge and re-encrypt internally (or use mTLS).

## Real-time communication

The classic interview comparison.

| Approach | How | Latency | Server cost | Use |
|----------|-----|---------|-------------|-----|
| **Short polling** | Client asks every N seconds | Up to N | High (wasted requests) | Simple, low-frequency |
| **Long polling** | Server holds request until data | Low | Medium (held connections) | Legacy-compatible push |
| **SSE** | Server-sent events, one-way stream over HTTP | Low | Low | **Server→client only**: feeds, notifications |
| **WebSocket** | Full-duplex, persistent | **Lowest** | Medium | **Bidirectional**: chat, games, collaboration |
| **WebRTC** | Peer-to-peer, UDP | Lowest | Lowest (P2P) | Video/voice calls |

> **Interview tip.** Do not reach for WebSockets reflexively. If the data flows
> **one way** (notifications, live scores, progress bars), **SSE** is simpler:
> it is plain HTTP, works through proxies, and reconnects automatically.

**Scaling WebSockets** is the follow-up question. Connections are **stateful**,
so:

- A load balancer must support sticky routing or connection-aware balancing
- Servers hold memory per connection (~10–50 KB) → one box handles ~50k–100k
- Broadcasting requires a **pub/sub backplane** (Redis, Kafka) because the
  recipient may be connected to a different server
- Clients need reconnect with exponential backoff and a resume token

## gRPC

HTTP/2 + Protocol Buffers.

| | REST/JSON | gRPC |
|---|---|---|
| Format | Text, human-readable | **Binary, compact** |
| Schema | Optional (OpenAPI) | **Required (.proto)**, generates clients |
| Streaming | Limited | **Bidirectional streaming** |
| Browser support | Native | Needs grpc-web proxy |
| Debuggability | curl works | Needs tooling |

> **Interview tip.** gRPC for **internal service-to-service** (fast, typed,
> streaming); REST/JSON for **public APIs** (universal, debuggable, cacheable).
> This split is the standard answer.

## DNS

```
Browser cache → OS cache → Resolver → Root → TLD (.com) → Authoritative
```

- **TTL** controls caching and therefore how fast changes propagate
- Record types: A, AAAA, CNAME, ALIAS, MX, TXT, NS, SRV
- **CNAME cannot exist at a zone apex** — use ALIAS/ANAME

**DNS as a load balancer** — return multiple A records, or geo/latency-based
responses.

> **Trap.** DNS-based failover is **slow and unreliable** for fast recovery:
> clients and intermediate resolvers cache beyond TTL. For sub-second failover
> use anycast IPs or an L4 load balancer, not DNS.

## Connection management

**Connection pooling** — establishing a TCP+TLS connection costs 1–3 RTT. Pool
and reuse. Sizing comes from Little's Law
([chapter 01](01-fundamentals-and-estimation.md)).

**Keep-alive** — reuse a connection for multiple requests.

**Backpressure** — when a consumer cannot keep up, the producer must slow down
rather than buffer unboundedly. TCP does this natively; your application layer
usually does not, which is why queues and bounded buffers matter
([chapter 10](10-reliability-patterns.md)).

## Network reality

> **The fallacies of distributed computing** — things that are false but
> systems are written as if true:
>
> 1. The network is reliable
> 2. Latency is zero
> 3. Bandwidth is infinite
> 4. The network is secure
> 5. Topology does not change
> 6. There is one administrator
> 7. Transport cost is zero
> 8. The network is homogeneous

Every reliability pattern in [chapter 10](10-reliability-patterns.md) exists
because one of these is false.

## Decision table

| Requirement | Choose |
|-------------|--------|
| Public API | REST/JSON over HTTP/2 |
| Internal RPC, low latency, typed | **gRPC** |
| Server pushes updates, one direction | **SSE** |
| Bidirectional real-time | **WebSocket** |
| Peer-to-peer audio/video | **WebRTC** |
| Loss-tolerant, latency-critical | **UDP** |
| Must not lose data, ordered | **TCP** |
| Mobile clients on poor networks | **HTTP/3 (QUIC)** |
| Sub-second failover | **Anycast / L4 LB**, not DNS |

## Practice

1. Why does a lost packet hurt more under HTTP/2 than HTTP/1.1 with six
   connections?
2. A dashboard shows live metrics, server→client only. Polling, SSE or
   WebSocket? Justify.
3. You run 2 M concurrent WebSocket connections. How many servers, and what
   infrastructure do you need for broadcast?
4. A payment POST times out. Is retrying safe? What makes it safe?
5. Why is DNS a poor failover mechanism, and what do you use instead?
6. Internal services currently use REST/JSON and are CPU-bound on
   serialisation. What do you propose, and what do you lose?

<details><summary>Answers</summary>

1. HTTP/2 multiplexes all streams over **one** TCP connection, so one lost
   packet blocks every stream (transport-level HOL blocking). With six
   connections, only one-sixth of traffic stalls. HTTP/3 solves this.
2. **SSE.** One-directional, plain HTTP, auto-reconnect, far simpler than
   WebSockets. Polling wastes requests and adds latency.
3. At ~50k connections/server, **~40 servers** plus headroom. You need a
   connection-aware LB, a **pub/sub backplane** (Redis/Kafka) so any server can
   reach any user, a presence/registry mapping user→server, and client
   reconnect with backoff.
4. Not safe — POST is not idempotent. Make it safe with an **idempotency key**
   the server records, so a repeat returns the original result instead of
   charging again.
5. Resolvers and clients cache beyond TTL, so propagation is minutes, not
   seconds. Use **anycast** (Global Accelerator-style) or an L4 load balancer
   with health checks.
6. **gRPC** — binary protobuf cuts CPU and bytes, gives typed clients and
   streaming. You lose browser-native support, curl-ability, and easy human
   inspection; you gain a schema you must version carefully.
</details>

---

Previous: [01 — Fundamentals and estimation](01-fundamentals-and-estimation.md) ·
Next: [03 — Load balancing and gateways](03-load-balancing-and-gateways.md)
