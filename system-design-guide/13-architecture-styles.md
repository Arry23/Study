# 13 — Architecture styles

How you carve a system into pieces. Interviewers probe whether you choose
deliberately or by fashion.

## Monolith

One deployable unit.

```
┌──────────────────────────────┐
│  Web │ Orders │ Users │ Pay  │
│        shared database        │
└──────────────────────────────┘
```

| Pros | Cons |
|------|------|
| Simple deploy, test, debug | Scales as one unit |
| **Transactions across the whole domain** | One bug can take everything down |
| **No network between modules** | Large codebase slows teams |
| Easy refactoring across boundaries | Technology lock-in |

> **Interview tip.** Say explicitly that **a monolith is the right default**.
> "Start with a well-modularised monolith; extract services when a specific
> pressure demands it." This is the current industry consensus and candidates
> who leap to microservices look inexperienced.

### Modular monolith

A monolith with enforced internal boundaries — separate modules, explicit
interfaces, no reaching into another module's tables.

> **Interview tip.** This gets most of the organisational benefit of
> microservices without distributed-systems cost, and it makes later
> extraction straightforward. Naming it is a strong move.

## Microservices

Independently deployable services, each owning its data.

```
┌────────┐  ┌────────┐  ┌────────┐
│ orders │  │ users  │  │ payment│
│  DB    │  │  DB    │  │  DB    │   ← database per service
└────────┘  └────────┘  └────────┘
```

| Pros | Cons |
|------|------|
| Independent deploy and scale | **Distributed systems problems, everywhere** |
| Team autonomy | No cross-service transactions |
| Fault isolation (if done right) | Network latency and partial failure |
| Technology choice per service | Operational complexity: tracing, discovery, CI/CD |

### The prerequisites

> **Interview tip.** Microservices require a platform. Listing these shows you
> understand the real cost:
>
> - Automated CI/CD per service
> - Centralised logging, metrics and **distributed tracing**
> - Service discovery and a gateway
> - Contract testing or schema registry
> - On-call ownership per service
> - Infrastructure as code

**Database per service** is the defining rule. Sharing a database recreates
coupling and prevents independent deployment.

> **Trap — the distributed monolith.** Services that must be deployed together,
> call each other synchronously in long chains, and share a database. You have
> paid every cost of microservices and gained nothing. If a design has six
> services in one request path, say so.

### Sizing

Not "micro". Size by **business capability** and team ownership — a service a
small team can own end to end. **Domain-Driven Design** terms worth using:

- **Bounded context** — a boundary within which terms have one meaning
- **Aggregate** — a consistency boundary; a transaction stays inside one
- **Ubiquitous language** — the domain vocabulary shared by code and people

> **Interview tip.** "I'd draw service boundaries along bounded contexts, so
> that a transaction never needs to span two services" is exactly the right
> framing.

## Service-oriented architecture

Older, coarser-grained, often with an **enterprise service bus** holding
routing and transformation logic. Microservices deliberately moved that logic
out of the bus and into the services ("smart endpoints, dumb pipes").

## Event-driven architecture

Services publish events; others react.

```
order-service ──"OrderPlaced"──► [ event bus ] ──► inventory
                                               ──► email
                                               ──► analytics
```

| Pros | Cons |
|------|------|
| Loose coupling — publisher knows no consumers | **Hard to trace** the overall flow |
| Easy to add consumers | Eventual consistency |
| Natural load levelling | Debugging requires good tooling |
| Temporal decoupling | Event schema versioning |

**Event notification** (thin event, consumer calls back for detail) vs
**event-carried state transfer** (fat event with the data) — the latter
removes the callback but duplicates data and ages.

> **Interview tip.** Pair this with the **outbox pattern**
> ([chapter 08](08-messaging-and-streaming.md)) so the state change and the
> event are atomic. Event-driven designs that ignore dual-write are incomplete.

## CQRS

Separate the **write model** from the **read model**.

```
Commands → write model → DB ──projection──► read model(s) → Queries
```

Use when read and write workloads differ sharply — complex normalised writes,
denormalised fast reads; or 100:1 read:write ratios.

> **Trade-off.** Two models means **eventual consistency** between them and
> more code. Do not apply CQRS to a CRUD screen. Apply it to the one or two
> aggregates that genuinely need it.

## Event sourcing

Store the **sequence of events** as the source of truth; current state is a
fold over them.

```
[AccountOpened] [Deposited 100] [Withdrew 30] → balance = 70
```

| Pros | Cons |
|------|------|
| **Complete audit trail** | Querying current state needs projections |
| Time travel / replay | Schema evolution of old events is painful |
| Natural fit with CQRS and EDA | **You can never delete** (GDPR tension) |
| Debug by replaying | Steep learning curve |

**Snapshots** avoid replaying millions of events from the beginning.

> **Interview tip.** Event sourcing is right for domains that are *naturally*
> event-shaped and audit-heavy: ledgers, order lifecycles, trading. It is
> wrong as a general-purpose default — say so.

## Serverless

Functions triggered by events; no servers to manage.

| Pros | Cons |
|------|------|
| No capacity management | **Cold starts** |
| Pay per use, scales to zero | Execution time limits |
| Fast for event-driven glue | Vendor lock-in |
| | Hard to run long or stateful work |

> **Interview tip.** Serverless is excellent for **spiky, event-driven,
> stateless** work (image processing on upload, webhooks, scheduled jobs) and
> poor for steady high-throughput services, where a container is cheaper and
> faster.

## Hexagonal / clean architecture

The internal structure of one service — and the bridge to LLD.

```
          ┌─────────────────────┐
  HTTP ──►│  adapter            │
  CLI  ──►│   ┌───────────────┐ │
          │   │  application  │ │
          │   │   ┌────────┐  │ │
          │   │   │ domain │  │ │
          │   │   └────────┘  │ │
          │   └───────────────┘ │
          │  adapter ──► DB     │
          └─────────────────────┘
```

**The dependency rule:** dependencies point **inwards**. The domain knows
nothing about HTTP, SQL or frameworks.

```python
# Domain defines the interface it needs (a port)
class OrderRepository(Protocol):
    def save(self, order: Order) -> None: ...
    def find(self, order_id: str) -> Order | None: ...

# Infrastructure implements it (an adapter)
class PostgresOrderRepository:
    def save(self, order: Order) -> None: ...
```

> **Interview tip.** This is **dependency inversion** (the D in SOLID,
> [chapter 19](19-solid-and-principles.md)) applied at architectural scale.
> The payoff: the domain is testable without a database, and you can swap
> Postgres for DynamoDB without touching business logic.

## Choosing

| Signal | Style |
|--------|-------|
| Small team, new product, unclear domain | **Modular monolith** |
| Independent scaling and deploy per domain, multiple teams | **Microservices** |
| Many consumers reacting to changes | **Event-driven** |
| Read and write workloads diverge sharply | **CQRS** |
| Audit trail is a hard requirement | **Event sourcing** |
| Spiky, event-driven, stateless | **Serverless** |
| Any service you want testable and long-lived | **Hexagonal** internally |

> **Interview tip.** The strongest answer is usually a **hybrid**: a
> modular monolith or a few coarse services, event-driven between them, CQRS
> only where needed, hexagonal inside each. Monocultures are a smell.

## Migration — strangler fig

Extracting a service without a rewrite:

```
       ┌──► new service (1 route)
Proxy ─┤
       └──► monolith (everything else)
```

Route one capability at a time to the new service, migrate its data, verify,
repeat. The monolith shrinks until it disappears — or stops shrinking because
the rest is fine.

> **Interview tip.** "Big bang rewrite" is the wrong answer to every
> modernisation question. Strangler fig plus **branch by abstraction** is the
> right one.

## Practice

1. Why is a monolith the right default?
2. What is a distributed monolith, and how do you spot one?
3. Why is database-per-service the defining microservices rule?
4. List five platform prerequisites before adopting microservices.
5. Event notification vs event-carried state transfer — trade-offs?
6. When is CQRS justified, and what does it cost?
7. Why does event sourcing conflict with GDPR, and what is the workaround?
8. How do you extract one capability from a monolith with no downtime?

<details><summary>Answers</summary>

1. Lowest operational cost, in-process calls, real transactions, and easy
   refactoring while the domain is still being discovered. You can modularise
   internally and extract later when a concrete pressure (scaling, team
   autonomy, deploy independence) justifies the cost.
2. Services that must deploy together, call each other synchronously in deep
   chains, and share a database. Signs: a release train, cross-service schema
   migrations, and one request touching six services.
3. Without it, two services are coupled through the schema — neither can change
   its tables or deploy independently, which removes the entire benefit.
4. CI/CD per service; centralised logs, metrics and **distributed tracing**;
   service discovery and a gateway; contract/schema testing; clear on-call
   ownership; infrastructure as code.
5. Notification keeps events small and avoids stale duplication but forces a
   callback (coupling, load, latency). State transfer removes the callback and
   enables offline consumers, but duplicates data that can go stale and makes
   schema versioning harder.
6. When reads and writes have genuinely different models or wildly different
   volumes. It costs a second model, a projection pipeline, and **eventual
   consistency** between write and read — plus the user-experience work to
   hide it.
7. Events are immutable and append-only, so "delete my data" cannot be done by
   deletion. Use **crypto-shredding** — encrypt personal data per user and
   destroy the key — or keep PII out of events entirely, storing only
   references.
8. **Strangler fig**: put a proxy in front, implement the capability in a new
   service, route a small percentage (or an internal cohort) to it, compare
   results, migrate its data with dual-write and backfill, shift traffic
   fully, then delete the old code path.
</details>

---

Previous: [12 — Security in design](12-security-design.md) ·
Next: [14 — Deployment and operations](14-deployment-and-operations.md)
