# 11 — DNS, CDN and global reach

## Route 53

Managed DNS, plus health checks and domain registration. **100% availability
SLA** — the only AWS service with one.

### Record types

| Type | Purpose |
|------|---------|
| A / AAAA | Name → IPv4 / IPv6 |
| CNAME | Name → another **name**. **Not allowed at the zone apex** |
| **Alias** | Route 53 extension: apex → AWS resource, **free**, health-aware |
| MX, TXT, NS, SOA, PTR, SRV | Standard |

> **Exam tip — the apex problem.** You cannot create a CNAME for
> `example.com` (the zone apex). Pointing the apex at an ALB, CloudFront, S3
> website or API Gateway requires an **Alias** record.

Alias targets: ELB, CloudFront, S3 website endpoints, API Gateway, VPC
endpoints, Global Accelerator, another record in the same zone. **Not EC2
instances.**

### Routing policies

The most examined part of Route 53.

| Policy | Behaviour | Use |
|--------|-----------|-----|
| **Simple** | One record, no health checks | Single resource |
| **Weighted** | Split by assigned weights | **Canary / blue-green / A-B testing** |
| **Latency** | Lowest-latency region for the user | Multi-region performance |
| **Failover** | Primary, with secondary on health-check failure | **Active-passive DR** |
| **Geolocation** | By user's country/continent | Compliance, localisation, licensing |
| **Geoproximity** | By distance, with a bias to shift traffic | Fine-grained geographic shaping |
| **Multivalue answer** | Up to 8 healthy records, client picks | Poor-man's load balancing |
| **IP-based** | By client CIDR | ISP-specific routing |

> **Exam tip — the three giveaways:**
> - "Gradually shift 10% of traffic" → **weighted**
> - "Serve users from the nearest/fastest region" → **latency**
> - "Automatically fail over to a standby region" → **failover**
> - "Users in Germany must be served from Germany **for legal reasons**" →
>   **geolocation** (not latency — the requirement is legal, not performance)

### Health checks

Three kinds: endpoint, calculated (combine other checks), CloudWatch alarm
state.

> **Exam tip.** DNS failover needs **health checks**. Also note DNS **TTL**
> governs how quickly clients notice a change — a long TTL slows failover.

### Hosted zones

| | Public | Private |
|---|---|---|
| Resolves from | The internet | Associated **VPCs only** |

**Route 53 Resolver** endpoints bridge DNS between on-premises and AWS
(chapter 03).

## CloudFront

CDN with 600+ edge locations. Caches content close to users and shortens the
TLS path even for dynamic content.

```
User → Edge location → (Regional edge cache) → Origin (S3, ALB, EC2, any HTTP)
```

### Key concepts

| Concept | Meaning |
|---------|---------|
| **Origin** | Where content comes from |
| **Behaviour** | Path pattern → origin + caching rules |
| **TTL** | How long an object is cached |
| **Invalidation** | Force-expire cached objects (chargeable above a free tier) |
| **OAC** | Origin Access Control — only CloudFront may read the S3 bucket |
| **Signed URL / cookie** | Restrict access to specific users |
| **Field-level encryption** | Encrypt specific form fields end to end |

> **Exam tip.** "Serve private S3 content through a CDN" → **CloudFront +
> OAC**, with the bucket blocked from public access. OAC replaces the older
> OAI.

> **Exam tip.** "Restrict content to paying subscribers" → **signed URLs**
> (one file) or **signed cookies** (many files).

> **Exam tip.** "Block users from certain countries" → CloudFront
> **geo-restriction**, or **WAF geo-match** for finer control.

### HTTPS and certificates

> **Trap.** A certificate used with **CloudFront** must be in **us-east-1**,
> regardless of where your origin lives. For an ALB, the certificate goes in
> the ALB's region.

### What CloudFront is good for

- Static assets (obvious)
- **Dynamic content** — TLS terminates at the edge, and the origin fetch uses
  AWS's backbone
- **Large file downloads and video streaming**
- Absorbing traffic spikes; shields the origin
- **Free data transfer out to the internet** is cheaper through CloudFront
  than direct from S3/EC2

> **Exam tip.** "Reduce latency for a global audience and reduce load on the
> origin" → **CloudFront**. It is also a frequent cost-optimization answer
> because egress via CloudFront is cheaper.

## Global Accelerator

Two **static anycast IPs** that route over the AWS backbone to the nearest
healthy endpoint.

| | CloudFront | Global Accelerator |
|---|---|---|
| Layer | HTTP/HTTPS (7) | **TCP/UDP (4)** |
| Caches? | **Yes** | **No** |
| Entry | Edge location | Edge, then AWS backbone |
| IPs | DNS name | **2 static anycast IPs** |
| Best for | Web content, APIs | Non-HTTP, gaming, IoT, VoIP, fast regional failover |

> **Exam tip — the distinguishing keywords:**
> - "cache", "static content", "web" → **CloudFront**
> - "static IP addresses", "UDP", "gaming", "non-HTTP", "instant regional
>   failover" → **Global Accelerator**

Global Accelerator failover is **seconds** because it does not depend on DNS
TTL — a frequent exam point against Route 53 failover.

## Putting it together

A standard global web architecture:

```
Route 53 (alias, latency or failover)
   │
CloudFront (OAC, WAF, ACM cert in us-east-1)
   ├── /static/*  → S3 (private)
   └── /api/*     → ALB → ASG → RDS Multi-AZ
```

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| Point `example.com` at an ALB | **Alias** record |
| Shift 5% of traffic to a new version | **Weighted** routing |
| Serve from the nearest region | **Latency** routing |
| Automatic region failover | **Failover** routing + health checks |
| Data must stay in-country for compliance | **Geolocation** routing |
| Cache static assets globally | **CloudFront** |
| Private S3 behind a CDN | CloudFront + **OAC** |
| Restrict content to subscribers | **Signed URLs / cookies** |
| HTTPS for CloudFront | ACM certificate in **us-east-1** |
| Static IPs for a whitelist, global | **Global Accelerator** |
| UDP game traffic, lowest latency | **Global Accelerator** |
| Failover faster than DNS allows | **Global Accelerator** |
| Reduce egress cost | Serve via **CloudFront** |
| Block traffic by country | Geo-restriction or **WAF** |

## Check yourself

1. Why can't you use a CNAME at the zone apex, and what do you use instead?
2. Weighted vs latency vs geolocation — one scenario each.
3. What two things does DNS failover depend on?
4. Where must a CloudFront certificate live?
5. OAC — what problem does it solve?
6. CloudFront vs Global Accelerator — three deciding keywords.
7. Why is Global Accelerator failover faster than Route 53 failover?
8. How can a CDN reduce cost rather than add it?

---

Previous: [10 — Application integration](10-application-integration.md) ·
Next: [12 — Security services](12-security-services.md)
