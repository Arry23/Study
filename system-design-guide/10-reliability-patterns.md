# 10 — Reliability patterns

Every pattern here exists because one of the fallacies of distributed computing
is false. Interviewers listen specifically for these.

## Timeouts

**The most important and most neglected setting.**

```python
# Wrong — waits forever
requests.get(url)

# Right
requests.get(url, timeout=(3, 10))   # connect, read
```

> **Trap.** A request with no timeout holds a thread, a connection and memory
> indefinitely. Under load, all threads end up waiting on one slow dependency
> and the service dies — even though it is "only" a dependency that is slow.
> This is the single most common cause of cascading failure.

**Choosing a value:** base it on the dependency's **p99**, not its average.
A timeout below p99 fails healthy requests; far above it provides no
protection.

**Deadline propagation** — pass the remaining budget downstream:

```
Client: 1000 ms budget
  → Service A (took 200 ms) passes 800 ms
    → Service B (took 300 ms) passes 500 ms
      → Service C must finish within 500 ms
```

> **Interview tip.** gRPC supports deadlines natively. Mentioning deadline
> propagation shows you have thought past "set a timeout".

## Retries

Retries turn transient failures into success — and outages into disasters.

**Rules:**

1. **Only retry idempotent operations** (or use an idempotency key)
2. **Only retry transient errors** — 503, timeouts, connection resets. Never
   400, 401, 404
3. **Bound the attempts** — 2 or 3, not "until success"
4. **Exponential backoff with jitter**

```python
import random, time

def retry(fn, attempts=3, base=0.1, cap=2.0):
    for i in range(attempts):
        try:
            return fn()
        except Transient:
            if i == attempts - 1:
                raise
            sleep = min(cap, base * (2 ** i))
            time.sleep(random.uniform(0, sleep))   # full jitter
```

> **Trap — the retry storm.** A service degrades; every client retries three
> times; load **triples** exactly when the service is weakest, guaranteeing it
> never recovers. Jitter spreads the retries; a circuit breaker stops them
> entirely.

**Retry budgets** — cap retries as a percentage of total requests (say 10%),
so retries can never dominate traffic.

> **Trap — retry amplification.** In a chain A→B→C, if each layer retries 3×,
> a single client request can become **27** calls to C. Retry at **one** layer,
> usually the outermost or the one closest to the failure.

## Circuit breaker

Stop calling a dependency that is clearly broken.

```
        failures > threshold
CLOSED ──────────────────────► OPEN
  ▲                              │  (reject immediately, no call)
  │                              │
  │  success          timeout elapsed
  │                              ▼
  └──────────────────────── HALF-OPEN
            failure          (allow a few probes)
```

| State | Behaviour |
|-------|-----------|
| **Closed** | Normal; count failures |
| **Open** | **Fail fast** without calling. Return fallback or error |
| **Half-open** | Let a few requests through to test recovery |

> **Interview tip.** The benefit is twofold: the **caller** stops wasting
> threads on calls that will fail, and the **callee** gets breathing room to
> recover. Say both halves.

Pair it with a **fallback** — cached data, a default value, or degraded
functionality.

## Bulkhead

Isolate resources so one failure cannot consume everything.

```
Without:  [ shared pool of 100 threads ]  ← slow dependency eats all 100

With:     [ A: 40 ][ B: 40 ][ C: 20 ]     ← C's failure costs only 20
```

Implement with separate thread pools, connection pools, or separate service
instances per dependency or tenant.

> **Interview tip.** Named after ship compartments — one flooded compartment
> does not sink the ship. Mention bulkheads when a question involves a
> multi-tenant system or several downstream dependencies of differing
> reliability.

## Rate limiting

Protects you from clients; throttling protects dependencies from you.

| Algorithm | How | Bursts | Memory |
|-----------|-----|--------|--------|
| **Fixed window** | Count per interval | Allows 2× at boundaries | Tiny |
| **Sliding window log** | Timestamps of each request | Exact | **High** |
| **Sliding window counter** | Weighted blend of two windows | Good approximation | Low |
| **Token bucket** | Tokens refill at rate R, capacity B | **Allows bursts up to B** | Tiny |
| **Leaky bucket** | Fixed drain rate | **Smooths** — no bursts | Tiny |

> **Interview tip.** **Token bucket** is the default answer: it bounds the
> average rate while permitting legitimate bursts, and it is two numbers per
> key in Redis.

```python
# Token bucket, conceptually
def allow(key, rate, capacity):
    now = time.time()
    tokens, last = store.get(key, (capacity, now))
    tokens = min(capacity, tokens + (now - last) * rate)
    if tokens < 1:
        return False
    store.set(key, (tokens - 1, now))
    return True
```

**Fixed-window boundary problem:** with a limit of 100/minute, a client can
send 100 at 00:59 and 100 at 01:00 — 200 in two seconds. Sliding window fixes
this.

**Distributed rate limiting** — the counter must be shared. Redis with Lua for
atomicity is standard; for extreme scale, approximate locally and reconcile.

Always return **429** with a `Retry-After` header.

## Load shedding

When overloaded, **reject work fast** rather than queueing it.

> **Interview tip.** Shedding 20% of requests quickly is better than serving
> 100% of them 30 seconds late — by then the client has timed out and retried,
> so the work was wasted anyway. This is counter-intuitive and worth saying
> explicitly.

Shed by **priority**: drop analytics before checkout; drop anonymous before
authenticated.

## Graceful degradation

Design what to do when a dependency is unavailable.

| Dependency down | Degrade to |
|-----------------|------------|
| Recommendations | Show popular items |
| Personalisation | Generic content |
| Search | Browse categories |
| Image service | Placeholder |
| Analytics | Drop silently |

> **Interview tip.** Classify dependencies as **critical** or **non-critical**
> early in a design, and make non-critical calls fail open. A checkout flow
> should never fail because the recommendation service is down.

## Idempotency

The antidote to retries. Covered in
[chapter 07](07-consistency-and-consensus.md); in practice:

```
POST /payments
Idempotency-Key: 7f3a9c21-...
```

Server stores key → result. A repeat returns the stored result rather than
charging again. Keys should expire (24h is typical).

## Health checks

| Check | Means | On failure |
|-------|-------|------------|
| **Liveness** | Process is alive | Restart |
| **Readiness** | Can serve traffic | Remove from LB |
| **Startup** | Finished initialising | Delay the other checks |

> **Trap.** A readiness check that calls the database makes a slow database
> remove **every** instance, turning degradation into total outage. Keep checks
> shallow, bounded, and prefer failing open.

## Chaos engineering

Deliberately inject failure in production-like environments: kill instances,
add latency, drop packets, fill disks.

> **Interview tip.** "Untested failover does not work." If a design claims
> multi-region failover, say that it must be exercised regularly — ideally
> with game days — or it will fail when needed.

## Putting it together

A well-defended call to a dependency:

```
Request
  → rate limiter (protect ourselves)
  → bulkhead (bounded pool)
  → circuit breaker (fail fast if broken)
  → timeout (bounded wait)
  → retry with jittered backoff (transient only, budgeted)
  → fallback (cached/default/degraded)
```

> **Interview tip.** Being able to list this stack in order is a complete
> answer to "how do you make service calls reliable?"

## Decision table

| Problem | Answer |
|---------|--------|
| Slow dependency exhausts threads | **Timeout** + bulkhead |
| Transient network blips | **Retry with jittered backoff** |
| Retries making an outage worse | **Circuit breaker** + retry budget |
| One tenant starves others | **Bulkhead** + per-tenant rate limit |
| Abusive client | **Rate limit**, 429 + `Retry-After` |
| Overloaded, queue growing | **Load shedding** by priority |
| Non-critical dependency down | **Graceful degradation**, fail open |
| Duplicate side effects from retries | **Idempotency key** |
| Slow DB removing all instances | Shallow **liveness**, bounded readiness |
| Failover never tested | **Chaos engineering** / game days |

## Practice

1. Why is a missing timeout more dangerous than a wrong one?
2. A→B→C, each retrying 3×. How many calls can one request make to C? Fix it.
3. Describe the three circuit breaker states and the transitions.
4. 100 requests/minute fixed window — show how a client sends 200 in two
   seconds. Which algorithm fixes it?
5. Why shed load rather than queue it?
6. The payment service is down. Which parts of an e-commerce site should still
   work?
7. A readiness probe queries the database. The DB slows to 5 s. What happens?
8. Design the full defence stack for calling a third-party API with a 99.5%
   SLA.

<details><summary>Answers</summary>

1. With no timeout, a hung dependency holds threads and connections until
   exhaustion, taking your service down too — the failure **propagates**. A
   wrong timeout only causes premature failures or late detection, which is
   bounded.
2. 3 × 3 × 3 = **27**. Retry at a single layer — typically the outermost or
   the one nearest the failure — and apply a retry budget so retries stay a
   small fraction of traffic.
3. **Closed** (normal, counting failures) → **Open** when failures exceed the
   threshold (fail fast, no calls) → after a cooldown, **Half-open** (a few
   probe requests) → **Closed** on success, back to **Open** on failure.
4. Send 100 at 00:00:59 and 100 at 00:01:00 — both windows are satisfied but
   200 arrive within two seconds. **Sliding window** (log or counter) fixes it.
5. An overloaded system that queues keeps doing work whose clients have already
   timed out and retried — so the work is wasted and the queue grows. Fast
   rejection lets clients back off and keeps latency bounded for the requests
   you do serve.
6. Browsing, search, cart, account, wishlists — everything except completing
   payment. Queue the order as "pending payment" if possible and notify later,
   rather than losing the sale.
7. Every instance fails readiness, the load balancer removes them all, and
   100% of traffic fails — a degraded database becomes a total outage. Make
   readiness shallow or fail open after a bounded check.
8. Rate limit outbound calls; bulkhead with a dedicated pool; circuit breaker
   tuned to the SLA; timeout from their p99; 2 retries with full jitter, only
   on transient errors, under a retry budget; idempotency keys on writes;
   cached/degraded fallback; and metrics plus alerting on breaker state.
</details>

---

Previous: [09 — Storage and search](09-storage-and-search.md) ·
Next: [11 — Observability and SLOs](11-observability-and-slos.md)
