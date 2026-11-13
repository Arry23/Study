# 14 — Deployment and operations

A design that cannot be deployed safely is not a design.

## Containers

```
Code + dependencies + runtime  →  image  →  container
```

| | VM | Container |
|---|---|---|
| Isolation | Hardware-level, strong | Kernel namespaces, weaker |
| Boot | Seconds to minutes | **Milliseconds** |
| Size | GBs | MBs |
| Density | Tens per host | Hundreds |

**Image layering** matters: order a Dockerfile from least- to most-frequently
changing, so a code change does not invalidate the dependency layer.

```dockerfile
FROM python:3.12-slim
RUN apt-get update && apt-get install -y --no-install-recommends libpq5 \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .                 # changes rarely
RUN pip install --no-cache-dir -r requirements.txt
COPY src/ /app/src/                     # changes every commit
USER 1000
```

> **Interview tip.** Run as a **non-root** user, pin the base image, use
> multi-stage builds to keep build tools out of the runtime image, and scan
> images for CVEs.

## Orchestration

Kubernetes concepts you should be able to name:

| Object | Purpose |
|--------|---------|
| **Pod** | One or more containers sharing network and storage |
| **Deployment** | Declarative replicas + rolling updates |
| **Service** | Stable virtual IP and DNS for a set of pods |
| **Ingress** | L7 routing into the cluster |
| **ConfigMap / Secret** | Configuration and credentials |
| **HPA** | Horizontal Pod Autoscaler |
| **StatefulSet** | Stable identities and storage for stateful workloads |
| **PodDisruptionBudget** | Minimum availability during voluntary disruption |

> **Interview tip.** Readiness and liveness probes
> ([chapter 03](03-load-balancing-and-gateways.md)) are the mechanism that
> makes rolling updates safe. Without a readiness probe, Kubernetes sends
> traffic to a pod that has not finished starting.

## Deployment strategies

| Strategy | How | Risk | Cost |
|----------|-----|------|------|
| **Recreate** | Stop all, start new | **Downtime** | Low |
| **Rolling** | Replace a few at a time | Mixed versions live | Low |
| **Blue-green** | Two full environments, switch traffic | **Instant rollback** | 2× infra |
| **Canary** | Small % to new version, ramp up | Lowest | Medium |
| **Shadow** | Mirror traffic to new version, discard responses | None | Medium |
| **Feature flag** | Deploy dark, enable per cohort | Lowest | Low |

> **Interview tip.** **Canary with automated rollback on SLO breach** is the
> gold-standard answer. Mention what you watch (error rate, p99 latency,
> business metric) and the rollback trigger.

**Rolling updates imply version coexistence.** Both versions run at once, which
constrains schema and API changes.

## Expand–contract (the schema migration answer)

Never make a breaking change in one step.

```
1. EXPAND   add the new column/field, nullable; deploy code that writes BOTH
2. MIGRATE  backfill existing rows
3. SWITCH   deploy code that reads the new field
4. CONTRACT remove the old column and the dual-write
```

> **Interview tip.** This is the single most useful operational pattern to
> name. "How do you rename a column with zero downtime?" → expand–contract,
> four deploys. Candidates who say "take a maintenance window" lose points.

The same shape applies to API versioning
([chapter 21](21-api-design.md)) and event schema evolution.

## Configuration

**12-factor principle:** configuration lives in the environment, not the
image. One artefact promoted through environments.

| Config kind | Where |
|-------------|-------|
| Non-secret, per environment | Environment variables / ConfigMap |
| Secrets | Secrets manager, injected at runtime |
| Feature flags | Flag service, changeable without deploy |
| Tuning knobs | Config service with hot reload |

> **Trap.** Baking environment config into the image means you cannot promote
> the exact artefact you tested. Build once, deploy everywhere.

## Infrastructure as code

Declarative, version-controlled, reviewed infrastructure. Properties that
matter:

- **Idempotent** — applying twice changes nothing the second time
- **Plan before apply** — the diff is the review artefact
- **State** — the mapping from code to real resources
- **Immutable infrastructure** — replace rather than mutate servers

> **Interview tip.** "Pets vs cattle" — do not SSH in and fix a server; rebuild
> it from code. Configuration drift is what makes incidents unreproducible.

## Multi-region

Covered in [chapter 06](06-scaling-data.md) for data. Operationally:

| Pattern | RTO | Complexity |
|---------|-----|------------|
| Backup and restore | Hours | Low |
| Pilot light | Tens of minutes | Medium |
| Warm standby | Minutes | High |
| Active-active | Near zero | **Very high** |

**Failover must be tested.** An untested runbook is a hypothesis.

> **Interview tip.** Distinguish **multi-AZ** (high availability, usually a
> configuration flag) from **multi-region** (disaster recovery, a genuine
> architecture decision). Interviewers use these terms precisely.

## Capacity and autoscaling

| Scale on | When |
|----------|------|
| CPU | General compute-bound work |
| **Request rate per instance** | Web tiers — tracks demand directly |
| **Queue depth / age** | Worker tiers |
| Custom business metric | Known load drivers |
| Schedule | Predictable daily peaks |

> **Trap.** Scaling on CPU for an I/O-bound service never triggers — the
> service is slow while CPU sits at 15%. Scale on concurrency, queue depth or
> latency instead.

**Pre-scale for known events.** Autoscaling takes minutes; a flash sale takes
seconds.

## Operational readiness

Before a service goes live:

- [ ] Dashboards for the golden signals
- [ ] Alerts on symptoms, tied to SLOs
- [ ] Runbook for every alert
- [ ] Rollback procedure, **tested**
- [ ] Load tested to expected peak × 3
- [ ] Failure modes exercised (dependency down, DB slow)
- [ ] On-call ownership assigned
- [ ] Data backup **and restore** verified

> **Interview tip.** "Backups are not verified until you have restored one" is
> a line worth having.

## Incident response

```
Detect → Triage → Mitigate → Resolve → Review
```

**Mitigate before you diagnose.** Roll back, shift traffic, disable the
feature flag — restore service first, understand later.

**Blameless postmortems** with concrete action items. The question is "what
about the system allowed this?", not "who did it?".

## Cost

| Lever | Typical saving |
|-------|----------------|
| Right-size instances | Large — most fleets are over-provisioned |
| Autoscale down off-peak | Large for non-24/7 workloads |
| Commitment discounts | Up to ~70% |
| Spot for interruptible work | Up to ~90% |
| Tier cold storage | Large on data-heavy systems |
| **Fix data transfer paths** | Often the hidden line item |
| Cache | Reduces compute and database load |

> **Interview tip.** In a cost question, ask what the bill actually consists
> of before optimising. Compute, storage and **data transfer** have different
> answers, and people usually guess compute when it is transfer.

## Decision table

| Requirement | Answer |
|-------------|--------|
| Zero-downtime deploy | **Rolling** or blue-green |
| Instant rollback | **Blue-green** |
| Lowest-risk release | **Canary** + automated rollback |
| Test with real traffic, no risk | **Shadow** traffic |
| Rename a column, no downtime | **Expand–contract** |
| Turn a feature off without deploying | **Feature flag** |
| Worker tier autoscaling | **Queue depth/age** |
| I/O-bound service autoscaling | Concurrency or latency, **not CPU** |
| Survive region loss, RTO minutes | **Warm standby** |
| Reproducible environments | **IaC + immutable infrastructure** |
| Known traffic spike tomorrow | **Pre-scale**, do not rely on autoscaling |

## Practice

1. Rename a database column used by a running service, with no downtime.
2. Why do rolling deployments constrain API changes?
3. Blue-green vs canary — when each?
4. A service is slow at 15% CPU and never autoscales. Diagnose and fix.
5. Why "build once, deploy everywhere"?
6. What is the first action in an incident, and why is it not diagnosis?
7. What does a PodDisruptionBudget protect against?
8. How would you validate a multi-region failover claim?

<details><summary>Answers</summary>

1. **Expand–contract.** Add the new column; deploy code writing both; backfill
   old rows; deploy code reading the new column; verify; then drop the old
   column and the dual-write.
2. Both versions run simultaneously, so the new version must tolerate old data
   and the old version must tolerate new requests — i.e. changes must be
   **backwards and forwards compatible** for at least one release.
3. Blue-green when you need **instant, complete rollback** and can afford
   double infrastructure. Canary when you want to **limit blast radius** and
   validate against real traffic gradually — it is safer but rollback is
   partial and slower.
4. It is **I/O-bound** — waiting on a dependency, not computing. CPU-based
   autoscaling will never fire. Scale on **concurrency, request rate or
   latency**; also check thread/connection pool saturation (Little's Law).
5. So the artefact you tested is the artefact you ship. Rebuilding per
   environment means the production binary was never tested, and
   environment-specific bugs become possible.
6. **Mitigate** — roll back, shift traffic, disable the flag. Users care about
   service restoration; root cause can wait for the postmortem, and diagnosing
   under pressure extends the outage.
7. It prevents **voluntary** disruptions (node drains, cluster upgrades) from
   taking too many replicas down at once, so maintenance cannot cause an
   outage.
8. Actually fail over, in a game day — take the primary region offline on a
   schedule, measure real RTO/RPO, confirm DNS/traffic shifting, data
   freshness and that the runbook matches reality. Anything less is a claim,
   not a capability.
</details>

---

Previous: [13 — Architecture styles](13-architecture-styles.md) ·
Next: [15 — The HLD interview framework](15-hld-framework.md)
