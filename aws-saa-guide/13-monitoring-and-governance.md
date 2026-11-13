# 13 — Monitoring and governance

## The three that get confused

| Service | Answers | Data |
|---------|---------|------|
| **CloudWatch** | *How is it performing?* | Metrics, logs, alarms, dashboards |
| **CloudTrail** | *Who did what, when, from where?* | API call audit log |
| **AWS Config** | *What does it look like, and did it change?* | Resource configuration history |

> **Exam tip.** This distinction appears on nearly every exam. Learn it as a
> sentence each:
> - CloudWatch = **performance**
> - CloudTrail = **who called the API**
> - Config = **configuration state and compliance**

## CloudWatch

### Metrics

- Namespaces per service; dimensions identify the resource
- Standard resolution **1 minute**; **high resolution** 1 second
- Retention: 1-second data for 3 hours, 1-minute for 15 days, 5-minute for 63
  days, 1-hour for **15 months**

> **Trap — what is NOT a default EC2 metric.** CPU, network and disk I/O are
> free. **Memory usage and disk space used are not** — they require the
> **CloudWatch agent** publishing custom metrics. The exam asks this often.

**Custom metrics** via `PutMetricData`. **Metric filters** turn log patterns
into metrics.

### Alarms

```
Alarm: metric, threshold, evaluation periods
States: OK | ALARM | INSUFFICIENT_DATA
Actions: SNS, Auto Scaling, EC2 action, Systems Manager
```

> **Exam tip.** "Notify on a threshold" → alarm → **SNS**. "Recover a failed
> instance automatically" → alarm on `StatusCheckFailed_System` → **EC2
> recover** action.

**Composite alarms** combine alarms to reduce noise.

### Logs

- **Log groups** → **log streams** → events
- Retention is configurable per group; **default is never expire** (a cost
  trap)
- Sources: CloudWatch agent, Lambda, ECS, API Gateway, VPC Flow Logs, RDS
- Export to **S3**, stream to **OpenSearch** or **Firehose**
- **Logs Insights** — query language over logs

> **Exam tip.** "Centralise logs from many accounts for analysis" →
> CloudWatch Logs → **Firehose** → S3/OpenSearch, or a cross-account
> destination.

**Contributor Insights** finds top-N contributors. **Synthetics** canaries
monitor endpoints from outside. **RUM** captures real user performance.

## CloudTrail

Records API calls across the account.

| Event type | Default | Notes |
|------------|---------|-------|
| **Management events** | **Enabled**, 90 days in Event history | Control-plane operations |
| **Data events** | **Off** (chargeable) | S3 object-level, Lambda invokes, DynamoDB items |
| **Insights events** | Off | Detects unusual call-rate patterns |

> **Trap.** The 90-day **Event history** is not a trail. For long-term
> retention you must create a **trail** delivering to **S3**.

> **Exam tip.** "Who deleted the object in this S3 bucket?" → CloudTrail with
> **data events enabled** — management events alone do not record object-level
> operations.

**Organization trail** — one trail covering every account.

**Log file integrity validation** — digest files prove logs were not tampered
with. Combine with **S3 Object Lock** for an immutable audit trail.

> **Exam tip.** "Prove audit logs have not been altered" → **log file
> integrity validation** + Object Lock + a separate logging account.

## AWS Config

Records resource configuration over time and evaluates **rules**.

| Capability | Use |
|------------|-----|
| Configuration history | What did this security group look like last Tuesday? |
| **Config rules** | Managed or custom compliance checks |
| **Conformance packs** | Bundles of rules mapped to a standard |
| **Remediation** | Auto-fix via SSM Automation |
| Aggregators | Multi-account, multi-region view |

> **Exam tip.** "Ensure all EBS volumes are encrypted and fix any that are
> not" → **Config rule** + **remediation action**. "Detect configuration
> drift" → Config.

> **Trap.** Config **detects and can remediate**, but it does not **prevent**.
> To prevent an action outright, use an **SCP** or an IAM policy.

## Systems Manager

A large toolbox; the exam tests a handful.

| Capability | Purpose |
|------------|---------|
| **Session Manager** | **Shell access with no SSH, no bastion, no open ports** |
| **Patch Manager** | Patch compliance and automated patching |
| **Run Command** | Run commands across fleets |
| **Parameter Store** | Configuration and secrets |
| **Automation** | Runbooks; used by Config remediation |
| **State Manager** | Keep instances in a defined state |
| **Inventory** | Software and configuration inventory |
| **Fleet Manager** | Manage instances without logging in |

> **Exam tip — Session Manager.** "Access instances in a private subnet
> without a bastion host or inbound ports" → **Session Manager**. It needs the
> SSM agent, an instance role, and either internet access or VPC endpoints.
> Sessions are logged to CloudTrail/S3, which is why it beats SSH for audit.

> **Exam tip.** "Patch hundreds of instances on a schedule and report
> compliance" → **Patch Manager** with a maintenance window.

## X-Ray

Distributed tracing. Shows a request's path across services, where latency
accumulates, and which dependency failed.

> **Exam tip.** "Find the bottleneck in a microservices call chain" →
> **X-Ray**. Not CloudWatch — metrics show *that* something is slow, traces
> show *where*.

## Trusted Advisor

Checks across five pillars: cost, performance, security, fault tolerance,
service limits.

> **Trap.** **Basic and Developer** support get only a handful of core checks.
> **Business and Enterprise** unlock the full set and programmatic access.

> **Exam tip.** "Identify idle resources and underused reservations" →
> **Trusted Advisor** (or Cost Explorer / Compute Optimizer).

## Compute Optimizer

ML-based right-sizing recommendations for EC2, ASGs, EBS, Lambda and ECS on
Fargate.

> **Exam tip.** "Which instance type should we move to?" →
> **Compute Optimizer**.

## Organizations-wide governance

| Service | Purpose |
|---------|---------|
| **Organizations + SCPs** | Hard guardrails |
| **Control Tower** | Opinionated multi-account landing zone with guardrails |
| **Service Catalog** | Curated, approved products users may launch |
| **Config aggregator** | Compliance across accounts |
| **CloudTrail organization trail** | Audit across accounts |
| **Firewall Manager** | WAF/Shield/SG policy across accounts |

> **Exam tip.** "Set up a secure multi-account environment with best-practice
> guardrails quickly" → **Control Tower**. "Let teams deploy only approved
> architectures" → **Service Catalog**.

## A sensible baseline

What a well-governed account looks like — and most of this exists in this
workspace already:

- CloudTrail org trail → S3 in a **separate logging account**, Object Lock on
- Config enabled everywhere, with conformance packs
- GuardDuty and Security Hub enabled, delegated administrator
- CloudWatch alarms on billing and critical metrics
- SCPs preventing disabling of any of the above
- Session Manager instead of SSH
- IAM Access Analyzer enabled

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| Who deleted the security group? | **CloudTrail** |
| Who deleted an S3 **object**? | CloudTrail with **data events** |
| Was this bucket ever public? | **AWS Config** history |
| Enforce encryption on all volumes | Config rule + remediation, or **SCP** to prevent |
| Alert when CPU > 80% | **CloudWatch alarm** → SNS |
| Monitor memory usage on EC2 | **CloudWatch agent** (custom metric) |
| Retain logs 7 years | CloudWatch Logs → **export to S3** with lifecycle |
| Shell into a private instance, no bastion | **Session Manager** |
| Patch a fleet on a schedule | **Patch Manager** |
| Find the slow service in a chain | **X-Ray** |
| Right-size instances | **Compute Optimizer** |
| Tamper-proof audit trail | CloudTrail integrity validation + **Object Lock** |
| Multi-account landing zone | **Control Tower** |
| Approved self-service catalogue | **Service Catalog** |

## Check yourself

1. One sentence each: CloudWatch, CloudTrail, Config.
2. Which EC2 metrics require the CloudWatch agent?
3. Why is CloudTrail Event history not enough for compliance?
4. What must be enabled to see S3 object-level API calls?
5. Config detects — what does it *not* do, and what does instead?
6. What does Session Manager remove the need for?
7. CloudWatch vs X-Ray for a latency problem?
8. What is the default CloudWatch Logs retention, and why does it matter?

---

Previous: [12 — Security services](12-security-services.md) ·
Next: [14 — Resilience and DR](14-resilience-and-dr.md)
