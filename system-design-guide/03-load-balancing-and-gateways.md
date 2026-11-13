# 03 — Load balancing and gateways

Every design you draw will have a box in front of the servers. Know what is in
it.

## Why load balance

```
                  ┌─ app-1
Clients → LB ─────┼─ app-2
                  └─ app-3
```

Four jobs: **distribute load**, **remove failed instances**, **enable
horizontal scaling**, **allow deploys without downtime**.

## L4 vs L7

| | **L4 (transport)** | **L7 (application)** |
|---|---|---|
| Sees | IP, port, TCP/UDP | **URL, headers, cookies, body** |
| Routing on | Connection tuple | **Path, host, header, method** |
| TLS | Pass-through or terminate | **Terminates** |
| Performance | Very high, low latency | Lower (must parse) |
| Features | Minimal | Rewrites, auth, rate limiting, canary |

> **Interview tip.** "L4 when you need raw throughput, non-HTTP protocols, or
> a static IP. L7 when you need to route on content — microservices by path,
> canary by header, A/B by cookie."

## Algorithms

| Algorithm | How | Best for |
|-----------|-----|----------|
| **Round robin** | Next in order | Homogeneous, stateless |
| **Weighted round robin** | Proportional to capacity | Mixed instance sizes |
| **Least connections** | Fewest active | **Long-lived or variable requests** |
| **Least response time** | Fastest + fewest | Latency-sensitive |
| **IP hash** | Hash(client IP) | Crude stickiness |
| **Consistent hashing** | Hash ring | **Cache affinity**, minimal reshuffle |
| **Random (two choices)** | Pick 2, choose better | Cheap, surprisingly good |

> **Interview tip.** "Power of two choices" — sample two servers at random and
> pick the less loaded. Near-optimal distribution without global state. Worth
> naming; it shows depth.

> **Trap.** Round robin is bad when requests vary wildly in cost. One server
> gets three expensive requests and queues while others idle. Use **least
> connections**.

## Health checks

```
Active:  LB polls GET /health every N seconds
Passive: LB observes real traffic for errors/timeouts
```

Two kinds of endpoint — and the distinction is an interview favourite:

| Endpoint | Means | Used for |
|---|---|---|
| **Liveness** | The process is alive | Restart if failing |
| **Readiness** | Ready to serve traffic | Remove from LB if failing |

> **Trap.** A deep health check that queries the database turns a slow database
> into **every instance being marked unhealthy**, removing all capacity and
> making an outage total. Keep liveness shallow; make readiness meaningful but
> bounded.

## Stickiness

Routing a user consistently to one server.

| Method | Mechanism |
|--------|-----------|
| Cookie-based | LB sets a cookie naming the backend |
| IP hash | Hash the client IP |
| Consistent hashing | Hash a key onto a ring |

> **Interview tip.** Stickiness is almost always the **wrong** answer. It
> breaks even distribution, complicates deploys, and loses state on failure.
> The right answer is to **externalise session state** (Redis, JWT, database)
> and keep servers stateless. Say this unprompted.
>
> Legitimate uses: WebSocket connections (inherently stateful), and cache
> affinity to raise local hit rates.

## Load balancer topology

**Single LB** is a single point of failure. Real deployments use:

```
            DNS / Anycast
                 │
        ┌────────┴────────┐
    LB-active         LB-standby      (VRRP / floating IP)
        │
   ┌────┼────┐
  app  app  app
```

- **Active-passive** with a floating IP (keepalived/VRRP)
- **Active-active** with DNS round robin or anycast
- **GSLB** — global server load balancing across regions

> **Interview tip.** When asked "what if the load balancer fails?", the answer
> is redundant LBs plus an IP-level failover mechanism (anycast or VRRP) —
> **not** DNS, which is too slow ([chapter 02](02-networking-and-protocols.md)).

## Reverse proxy vs forward proxy

| | Reverse proxy | Forward proxy |
|---|---|---|
| Sits in front of | **Servers** | **Clients** |
| Client knows? | No | Yes |
| Use | LB, TLS, caching, compression, WAF | Corporate egress, filtering, anonymity |

Nginx, Envoy, HAProxy and Traefik are reverse proxies. A load balancer is one
kind of reverse proxy.

## API gateway

A single entry point for clients, doing the cross-cutting work so services do
not have to.

```
Clients → API Gateway → ┌─ users-service
                        ├─ orders-service
                        └─ payments-service
```

Responsibilities:

| Concern | Why at the gateway |
|---|---|
| **Authentication** | Verify once, pass identity downstream |
| **Rate limiting** | Protect everything behind it |
| **Routing** | Path/host → service |
| **Request aggregation** | One client call, several service calls |
| **Protocol translation** | REST outside, gRPC inside |
| **TLS termination** | One certificate to manage |
| **Observability** | Uniform logs, metrics, request IDs |
| **Versioning** | Route `/v1` and `/v2` |

> **Trade-off.** A gateway removes duplication but becomes a **single point of
> failure** and, if you are careless, a place where business logic accretes.
> Keep it to cross-cutting concerns; run it redundantly.

### BFF — Backend for Frontend

One gateway per client type (web, mobile, partner), each shaping responses for
that client.

> **Interview tip.** Mention BFF when a question involves mobile clients on
> poor networks needing fewer, smaller calls than the web app.

## Service mesh

Moves service-to-service concerns into a **sidecar proxy** next to each
instance.

```
┌─ service-a ─┐        ┌─ service-b ─┐
│  app        │        │  app        │
│  sidecar ───┼───────►│  sidecar    │
└─────────────┘        └─────────────┘
         control plane (Istio / Linkerd)
```

Gives you, without changing application code: mTLS everywhere, retries,
timeouts, circuit breaking, traffic splitting for canary, and uniform traces.

> **Trade-off.** Powerful and heavy. A mesh adds a proxy hop (latency), a
> control plane to operate, and real debugging complexity. Do not propose one
> for five services. Say so — recognising when **not** to use something scores
> well.

| | API Gateway | Service Mesh |
|---|---|---|
| Traffic | **North-south** (in/out) | **East-west** (service↔service) |
| Deploy | Centralised | Sidecar per pod |
| Primary concern | Client-facing | Inter-service reliability and security |

## Rate limiting at the edge

Covered in depth in [chapter 10](10-reliability-patterns.md), but the gateway
is where it usually lives. Algorithms: fixed window, sliding window log,
sliding window counter, token bucket, leaky bucket.

> **Interview tip.** **Token bucket** is the usual answer — it allows bursts
> while bounding the average rate, and it is cheap to implement in Redis.

## CDN

An edge cache in front of everything.

| Serves | Benefit |
|--------|---------|
| Static assets | Offload origin, cut latency |
| Dynamic content | TLS at the edge, backbone transit |
| Large media | Range requests, throughput |

**Pull CDN** fetches on first miss (simple, good for most). **Push CDN** you
upload to (good for large, predictable assets).

> **Interview tip.** Putting a CDN in front is one of the cheapest wins in any
> read-heavy design. Mention it early, with cache-key and invalidation
> strategy ([chapter 04](04-caching.md)).

## Putting it together

```
            Users
              │
          DNS / Anycast
              │
            CDN  ─────────── static assets
              │
       API Gateway (authN, rate limit, routing)
              │
        Load Balancer (L7)
        ┌─────┼─────┐
      app   app   app        ← stateless
        └─────┼─────┘
         Cache / DB / Queue
```

> **Interview tip.** Draw this skeleton first in almost any HLD question, then
> specialise. It takes 60 seconds and gives you a structure to hang
> everything on.

## Decision table

| Requirement | Answer |
|-------------|--------|
| Route `/api` and `/static` differently | **L7** load balancer |
| Non-HTTP protocol, extreme throughput | **L4** |
| Static IP for whitelisting | **L4** / anycast |
| Requests vary wildly in cost | **Least connections** |
| Maximise cache hit rate on backends | **Consistent hashing** |
| User must hit the same server | Externalise state; stickiness only if truly stateful |
| One entry point, auth and rate limits | **API gateway** |
| Mobile needs fewer, smaller calls | **BFF** |
| mTLS and retries across 200 services | **Service mesh** |
| Global static content | **CDN** |
| LB itself must not be a SPOF | Redundant LBs + **anycast/VRRP** |

## Practice

1. Why is a health check that queries the database dangerous?
2. Liveness vs readiness — what does each failure trigger?
3. When is stickiness defensible, and what do you prefer instead?
4. A gateway is a SPOF. How do you mitigate it?
5. Round robin is causing uneven load. Diagnose and fix.
6. You have 8 services. Should you adopt a service mesh? Justify either way.
7. Explain "power of two choices" and why it beats plain random.

<details><summary>Answers</summary>

1. A slow or down database makes **every** instance fail its check, so the LB
   removes all of them — converting a degraded dependency into a total outage.
   Keep liveness shallow; bound readiness checks and fail open where sensible.
2. Liveness failure → **restart the process**. Readiness failure → **remove
   from the load balancer** but leave it running (it may recover).
3. Defensible for **WebSockets** and for **cache affinity**. Otherwise
   externalise session state to Redis/JWT/database and keep servers stateless.
4. Run multiple gateway instances behind an L4 LB or anycast, across AZs;
   keep it stateless; autoscale; and ensure it fails open for non-critical
   concerns where acceptable.
5. Requests have very different costs, so equal *counts* give unequal *load*.
   Switch to **least connections** or least response time; also check for a
   slow dependency affecting one instance.
6. **Probably not.** Eight services can use libraries or a gateway for
   retries/timeouts. A mesh adds a control plane, a sidecar hop and real
   operational cost. Revisit at scale, or if mTLS everywhere is mandated.
7. Sample two servers at random, send to the less loaded. Plain random can
   pile onto an unlucky server; comparing two makes the maximum load grow
   **log log n** instead of **log n** — near-optimal without global state.
</details>

---

Previous: [02 — Networking and protocols](02-networking-and-protocols.md) ·
Next: [04 — Caching](04-caching.md)
