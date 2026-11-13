# 05 — Load balancing and auto scaling

Together these deliver "highly available and scalable", which is the single
most common phrase in exam questions.

## The three load balancers

| | **ALB** | **NLB** | **GWLB** |
|---|---|---|---|
| OSI layer | **7** (HTTP/HTTPS) | **4** (TCP/UDP/TLS) | **3** (IP) |
| Routing on | Path, host, header, query, method | Protocol and port | All traffic |
| Performance | High | **Millions of req/s, ultra-low latency** | — |
| Static IP | No (use DNS) | **Yes, one Elastic IP per AZ** | — |
| Targets | Instance, IP, **Lambda** | Instance, IP, **ALB** | Appliances |
| TLS termination | Yes | Yes | No |
| WebSocket / HTTP/2 | Yes | Pass-through | — |
| Preserves source IP | No (use `X-Forwarded-For`) | **Yes** | Yes |

> **Exam tip.** The three deciding keywords:
> - **"path-based" / "host-based" / "microservices" / "HTTP"** → **ALB**
> - **"static IP" / "extreme performance" / "TCP or UDP" / "millions"** → **NLB**
> - **"third-party firewall / IDS / IPS appliance"** → **GWLB**

**Classic Load Balancer** is legacy. If it appears as an option it is almost
certainly wrong.

### ALB routing

```
example.com/api/*    → target group: api
example.com/img/*    → target group: images
api.example.com      → target group: api
Header X-Version: v2 → target group: canary
```

ALB can also return fixed responses, redirect HTTP→HTTPS, and authenticate
users via Cognito or OIDC before traffic reaches targets.

> **Exam tip.** "Authenticate users before the request reaches the application"
> → **ALB with Cognito/OIDC authentication**, not code changes.

### Target groups and health checks

A target group holds targets plus a health check:

```
Protocol/port, path (/health), interval 30s,
timeout 5s, healthy threshold 2, unhealthy threshold 2
```

Unhealthy targets are removed from rotation. With an ASG, failing an **ELB
health check** can trigger replacement — but only if the ASG health check type
is set to `ELB`.

> **Trap.** The default ASG health check is **EC2** only, which just checks the
> instance is running. An app that has crashed but whose instance is up will
> never be replaced. Set `health_check_type = ELB`.

### Cross-zone load balancing

| | Default |
|---|---|
| ALB | **On**, free |
| NLB | **Off**, charged for cross-AZ data when enabled |

> **Exam tip.** "Uneven load across AZs behind an NLB" → enable cross-zone
> load balancing. Note the data transfer cost trade-off.

### Sticky sessions

ALB and NLB support stickiness via cookies (ALB) or source IP (NLB).

> **Exam tip.** Stickiness is usually the *wrong* answer architecturally —
> the better one is to externalise session state to **ElastiCache** or
> **DynamoDB**, making the tier stateless. The exam rewards statelessness.

## Auto Scaling Groups

```
ASG
├── Launch template (AMI, instance type, SG, IAM profile, user data)
├── Min / Desired / Max
├── Subnets across multiple AZs       ← the availability mechanism
├── Health check type: EC2 | ELB
└── Scaling policies
```

**Launch templates** supersede launch configurations — versioned, support
mixed instance types and Spot. If a question offers a launch configuration,
prefer the template.

### Scaling policies

| Policy | How it works | Use |
|--------|--------------|-----|
| **Target tracking** | Keep a metric at a target (e.g. CPU 50%) | **Default choice** — simplest |
| **Step scaling** | Different adjustments per alarm breach size | Needs graduated response |
| **Simple scaling** | One adjustment per alarm, with cooldown | Legacy |
| **Scheduled** | At a time | Known daily/weekly patterns |
| **Predictive** | ML forecast, scales ahead | Recurring, predictable cycles |

> **Exam tip.** "Simplest" or "least configuration" → **target tracking**.
> "Traffic spikes every weekday at 09:00" → **scheduled** (or predictive).
> "Scale before the load arrives" → **predictive**.

Useful target-tracking metrics: `ASGAverageCPUUtilization`,
`ALBRequestCountPerTarget`.

> **Exam tip.** For a web tier behind an ALB, **ALBRequestCountPerTarget** is
> often a better signal than CPU, because it tracks demand directly.

### Warm-up, cooldown, lifecycle hooks

- **Cooldown** — pause after a scaling activity so metrics settle
- **Warm-up** — new instances excluded from metrics until ready
- **Lifecycle hooks** — pause an instance at launch or terminate to run work
  (install software, drain connections, copy logs off)

> **Exam tip.** "Run a script before an instance is terminated" / "finish
> in-flight work before shutdown" → **lifecycle hook** (`Terminating:Wait`).

### Termination policy

Default order: largest AZ imbalance first, then oldest launch
template/configuration, then closest to the next billing hour.

**Instance protection** shields specific instances from scale-in.

### Warm pools

Pre-initialised, stopped instances ready to join quickly.

> **Exam tip.** "Application takes a long time to bootstrap; scaling is too
> slow" → **warm pool** (or a golden AMI).

## Availability patterns

**Multi-AZ is the baseline.** An ASG spanning three AZs with an ALB survives
an AZ failure with no intervention.

```
Route 53
   │
  ALB (public subnets, 3 AZs)
   │
  ASG (private subnets, 3 AZs)  ── min 3, so one per AZ survives
   │
  RDS Multi-AZ / Aurora
```

> **Exam tip.** If a question says the application must survive the loss of an
> AZ, check that **min capacity ≥ number of AZs** so capacity remains after a
> failure. A min of 2 across 3 AZs can leave you short.

## Scaling other services

| Service | Mechanism |
|---------|-----------|
| EC2 | Auto Scaling Group |
| ECS | Service Auto Scaling (+ Capacity Providers) |
| DynamoDB | Auto scaling on RCU/WCU, or on-demand mode |
| Aurora | Auto Scaling for read replicas; Serverless v2 for capacity |
| Lambda | Automatic; control with reserved/provisioned concurrency |

**AWS Auto Scaling** is the unified console across these.

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| Route `/api` and `/web` to different services | **ALB** path-based routing |
| Static IP for a whitelist | **NLB** |
| Millions of TCP connections, lowest latency | **NLB** |
| Inspect traffic with a third-party appliance | **GWLB** |
| Invoke Lambda from a load balancer | **ALB** |
| Authenticate before reaching the app | **ALB** + Cognito/OIDC |
| App hangs but instance is healthy | ASG health check type = **ELB** |
| Keep CPU at 60% | **Target tracking** |
| Scale up before Monday 09:00 | **Scheduled** or **predictive** |
| Drain connections before termination | **Lifecycle hook** |
| Slow bootstrap hurting scale-out | **Warm pool** / golden AMI |
| Make the tier stateless | Session state in **ElastiCache** / DynamoDB |

## Check yourself

1. Which load balancer gives a static IP, and which can target Lambda?
2. Why is the default ASG health check insufficient for a crashed app?
3. Cross-zone load balancing — default on which LB, and what does it cost?
4. Target tracking vs step scaling — when each?
5. What does a lifecycle hook let you do?
6. Why is stickiness usually the wrong architectural answer?
7. Three AZs, min capacity 2 — what is the risk?

---

Previous: [04 — EC2 and compute](04-ec2-and-compute.md) ·
Next: [06 — S3](06-s3.md)
