# 11 — Observability and SLOs

A design is not finished when it works; it is finished when you can tell
whether it is working.

## Monitoring vs observability

| | Monitoring | Observability |
|---|---|---|
| Answers | "Is the known thing broken?" | "Why is this unknown thing broken?" |
| Based on | Predefined dashboards and alerts | Rich, high-cardinality data you can query |

> **Interview tip.** Monitoring tells you *that* checkout latency rose.
> Observability lets you discover it is only for users in one region, on one
> API version, hitting one shard.

## The three pillars

| | **Logs** | **Metrics** | **Traces** |
|---|---|---|---|
| What | Discrete events | Numeric aggregates over time | One request across services |
| Cardinality | High | **Low** (labels explode cost) | High |
| Cost | High volume | Cheap | Medium (sampled) |
| Answers | What happened in this case? | Is the system healthy? | Where did the time go? |

### Logs

**Structured, not prose.**

```python
# Bad
log.info(f"User {uid} failed to checkout: {err}")

# Good
log.info("checkout_failed", extra={
    "user_id": uid, "order_id": oid, "error": str(err),
    "trace_id": trace_id, "duration_ms": 142,
})
```

Structured logs are queryable: *"all `checkout_failed` where
`duration_ms > 1000` grouped by region"*.

**Levels:** ERROR (needs action), WARN (suspicious), INFO (business events),
DEBUG (off in production).

> **Trap.** Logging every request at INFO with full bodies is how teams end up
> with a logging bill larger than their compute bill. Sample, and never log
> secrets or PII.

### Metrics

Four types:

| Type | Meaning | Example |
|------|---------|---------|
| **Counter** | Monotonically increasing | `http_requests_total` |
| **Gauge** | Goes up and down | `queue_depth`, `memory_bytes` |
| **Histogram** | Bucketed distribution | `request_duration_seconds` |
| **Summary** | Client-side quantiles | p99 latency |

> **Trap — cardinality explosion.** A label like `user_id` creates one time
> series per user. A million users means a million series, and your metrics
> system falls over. Labels must be **low cardinality**: status code, method,
> endpoint, region — never IDs. Put IDs in logs and traces instead.

**The four golden signals** (Google SRE):

| Signal | Question |
|--------|----------|
| **Latency** | How long do requests take? (split success vs error) |
| **Traffic** | How much demand? |
| **Errors** | What fraction fail? |
| **Saturation** | How full is the system? |

**RED** (Rate, Errors, Duration) for request-driven services;
**USE** (Utilization, Saturation, Errors) for resources.

> **Interview tip.** Naming the golden signals or RED/USE when asked "how
> would you monitor this?" is a quick, strong answer.

### Traces

```
Trace: checkout request (420 ms)
├── span: api-gateway            (420 ms)
│   ├── span: auth-service        (15 ms)
│   ├── span: cart-service        (80 ms)
│   │   └── span: redis GET       (2 ms)
│   └── span: payment-service    (310 ms)   ← the problem
│       └── span: stripe API     (295 ms)   ← actually here
```

Each span carries a **trace ID** (whole request) and **span ID** (this unit),
propagated via headers (W3C `traceparent`, OpenTelemetry).

> **Interview tip.** Traces are the only thing that answers "which of our 30
> services made this slow?" In a microservices design question, say that
> trace-context propagation must be built in from the start — retrofitting it
> is painful.

**Sampling:** head-based (decide at the start, cheap) vs tail-based (decide
after seeing the whole trace — keeps all the slow and failed ones, which is
what you actually want).

## Correlation

One identifier tying everything together:

```
Request → generate trace_id → propagate in headers
        → attach to every log line
        → attach to every span
        → return in the response header for support tickets
```

> **Interview tip.** "Every log line carries the trace ID" is a small detail
> that makes debugging a distributed system tractable, and interviewers notice
> it.

## SLI, SLO, SLA

| Term | Definition | Example |
|------|------------|---------|
| **SLI** | A measured indicator | Fraction of requests under 300 ms |
| **SLO** | Your internal target | 99.9% of requests under 300 ms over 30 days |
| **SLA** | Contractual promise + penalty | 99.5%, or credits |

> **Interview tip.** The SLA should always be **looser** than the SLO. You want
> internal alarms firing well before you owe anyone money.

### Choosing SLIs

Measure what **users** experience, not what is easy:

| Good SLI | Poor SLI |
|----------|----------|
| Successful requests / total | CPU utilisation |
| p99 latency at the edge | Average latency |
| Freshness of a pipeline | Jobs started |

### Error budgets

```
SLO 99.9% over 30 days
→ allowed downtime = 0.1% × 30 days ≈ 43 minutes
→ that is your error budget
```

**How it is used:**

- Budget remaining → ship features, take risks
- Budget exhausted → freeze features, fix reliability

> **Interview tip.** Error budgets turn "how reliable should it be?" from an
> argument into arithmetic, and they explicitly make 100% the **wrong** target
> — chasing it costs more than it returns. This is a strong thing to say.

## Alerting

> **Alert on symptoms, not causes.**

| Alert on | Not on |
|----------|--------|
| "Error rate > 1% for 5 min" | "CPU > 80%" |
| "p99 latency > 1 s" | "Disk 70% full" |
| "Queue age > 10 min" | "A pod restarted" |

High CPU is only a problem if users notice. Paging on causes produces alert
fatigue, and alert fatigue produces missed outages.

**Every alert must be actionable.** If the runbook is "look at it and do
nothing", delete the alert.

**Burn-rate alerting** — page when the error budget is being consumed too
fast:

```
Fast burn: 14.4× budget rate over 1 hour  → page
Slow burn: 6×    budget rate over 6 hours → ticket
```

> **Interview tip.** Multi-window burn-rate alerting is the modern SRE answer
> and distinguishes a thoughtful response from "alert at 90% CPU".

## Dashboards

Three tiers:

1. **Service health** — the golden signals, one screen, for on-call
2. **Business** — orders, sign-ups, revenue
3. **Deep dive** — per-dependency, per-shard, for debugging

> **Trap.** A dashboard with 60 graphs communicates nothing. The on-call
> dashboard should fit on one screen and answer "are users affected?" in five
> seconds.

## Beyond the three pillars

| Technique | Purpose |
|-----------|---------|
| **Profiling** | CPU/memory flame graphs in production |
| **Synthetic monitoring** | Probe from outside, continuously |
| **RUM** | Real user metrics from browsers/apps |
| **Change tracking** | Overlay deploys on graphs — most incidents follow a change |

> **Interview tip.** "Was anything deployed?" is the first question in most
> incidents. Annotating dashboards with deploys is cheap and valuable.

## Decision table

| Need | Use |
|------|-----|
| Is the system healthy? | **Metrics** — golden signals |
| Why did *this* request fail? | **Logs** with trace ID |
| Which service is slow? | **Traces** |
| Keep all slow traces | **Tail-based sampling** |
| Correlate everything | **Trace ID propagation** |
| Decide reliability targets | **SLOs + error budget** |
| Reduce pager noise | Alert on **symptoms**, burn rate |
| Detect outages users see | **Synthetic probes** + RUM |
| Debug high CPU in production | **Continuous profiling** |

## Practice

1. Why is `user_id` a dangerous metric label but a good log field?
2. Name the four golden signals, and what each catches that the others miss.
3. SLO is 99.95% monthly. What is the error budget in minutes?
4. Why alert on error rate rather than CPU?
5. A request is slow across 12 services. Which pillar finds it, and what must
   have been built in advance?
6. Head-based vs tail-based sampling — which keeps the traces you care about?
7. Your SLA is 99.9% and your SLO is 99.9%. What is wrong?
8. Design observability for a checkout flow spanning 5 services.

<details><summary>Answers</summary>

1. As a metric label it creates one time series per user — **cardinality
   explosion** that overwhelms the metrics backend. In logs it is just a
   field, and logs are already high-cardinality and queryable.
2. **Latency** (slow but succeeding), **traffic** (demand changes, including
   drops that signal an upstream outage), **errors** (failures), **saturation**
   (approaching a limit before it becomes latency or errors). Saturation is the
   leading indicator the others lack.
3. 0.05% × 30 days = 0.0005 × 43,200 min ≈ **21.6 minutes**.
4. Users do not experience CPU; they experience errors and latency. High CPU
   may be harmless, and low CPU does not mean healthy. Symptom alerts fire
   exactly when users suffer, which keeps the pager meaningful.
5. **Traces.** They require **trace-context propagation** through every
   service and client library, instrumented in advance — you cannot
   reconstruct it after the fact.
6. **Tail-based** — it decides after seeing the complete trace, so you can keep
   all errors and slow traces. Head-based samples randomly at the start and
   usually discards the interesting ones.
7. There is no margin. You will breach the contractual SLA at exactly the
   moment you breach your internal target, with no warning window. The SLO
   should be **stricter** than the SLA.
8. Trace ID generated at the gateway and propagated to all five services;
   structured logs carrying it; RED metrics per service plus business metrics
   (orders/min, payment success rate); SLOs on checkout success rate and p99
   latency with burn-rate alerts; synthetic checkouts every minute; deploy
   annotations; and a DLQ/alert on the payment queue.
</details>

---

Previous: [10 — Reliability patterns](10-reliability-patterns.md) ·
Next: [12 — Security in design](12-security-design.md)
