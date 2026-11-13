# 19 — Service cheat sheet

One page. Review this the morning of the exam.

## Compute

| Service | One line |
|---------|----------|
| **EC2** | Virtual machines. Families: T burst, M general, C compute, R memory, I storage, P/G GPU |
| **Lambda** | Serverless functions. **15 min max**, 10 GB memory, CPU scales with memory |
| **ECS** | AWS-native containers. Task role (app) ≠ task execution role (agent) |
| **EKS** | Managed Kubernetes. **IRSA** for pod-level IAM |
| **Fargate** | Serverless compute for ECS/EKS |
| **Batch** | Managed batch, Spot-friendly |
| **App Runner** | Container/repo → HTTPS service |
| **Lightsail** | Simple VPS, fixed price |

**Purchasing:** On-Demand · Savings Plans (flexible, covers Fargate+Lambda) ·
RI (fixed shape; zonal reserves capacity) · **Spot (90% off, 2-min warning)** ·
Dedicated Host (BYOL)

**Placement groups:** cluster (low latency) · spread (7/AZ, availability) ·
partition (racks)

## Storage

| Service | One line |
|---------|----------|
| **S3** | Object storage. 11 nines. Strongly consistent |
| **EBS** | Block, **one AZ**. gp3 default, io2 BX for 256k IOPS |
| **Instance store** | Ephemeral NVMe, max IOPS, **lost on stop** |
| **EFS** | Shared **NFS**, Linux, multi-AZ |
| **FSx Windows** | **SMB** + Active Directory |
| **FSx Lustre** | **HPC/ML**, S3-linked |
| **Storage Gateway** | Hybrid: File / Volume (cached, stored) / **Tape** |
| **AWS Backup** | Central backup; **Vault Lock** = WORM |

**S3 classes:** Standard · **Intelligent-Tiering (unknown patterns)** ·
Standard-IA · **One Zone-IA (reproducible)** · Glacier Instant / Flexible /
**Deep Archive (cheapest, 12–48 h)**

**S3 security:** Block Public Access · bucket policy · **OAC** for CloudFront ·
presigned URLs · Object Lock (**Compliance mode = nobody deletes**) ·
SSE-S3 / **SSE-KMS** / SSE-C · Bucket Keys for KMS cost

## Networking

| Item | Key fact |
|------|----------|
| **Subnet is public** | Only if its route table has an **IGW** route |
| **Reserved IPs** | **5 per subnet** — /24 gives 251 usable |
| **NAT gateway** | **Public subnet, one per AZ**, outbound only, charged per GB |
| **Security group** | **Stateful**, allow only, instance level |
| **NACL** | **Stateless**, allow+deny, subnet level, **ephemeral ports** |
| **Gateway endpoint** | **S3 and DynamoDB only. Free** |
| **Interface endpoint** | Everything else, ENI, hourly + per GB |
| **VPC peering** | **Not transitive**, no overlapping CIDRs |
| **Transit Gateway** | Hub-and-spoke, transitive |
| **Direct Connect** | Dedicated, **weeks to provision**, not encrypted by default |
| **Site-to-Site VPN** | IPsec over internet, minutes to set up |
| **Flow Logs** | **Metadata** only; Traffic Mirroring for packets |

**Load balancers:** **ALB** L7 path/host, Lambda targets · **NLB** L4, static
IP, millions req/s · **GWLB** appliances

**ASG:** target tracking (default) · step · scheduled · predictive ·
**health check type = ELB** · lifecycle hooks · warm pools

## Databases

| Service | One line |
|---------|----------|
| **RDS** | Managed relational. **Multi-AZ = HA**, **read replica = read scaling** |
| **Aurora** | 6 copies/3 AZs, 15 replicas, **Global DB < 1 s RPO**, Serverless v2 |
| **DynamoDB** | NoSQL, single-digit ms. **DAX = microseconds**, Streams, Global Tables |
| **ElastiCache** | **Redis** (persistence, HA) vs **Memcached** (simple, multi-threaded) |
| **Redshift** | OLAP warehouse. Spectrum queries S3. **Never OLTP** |
| **Neptune** | **Graph** — relationships, fraud rings, knowledge graphs |
| **DocumentDB** | MongoDB-compatible |
| **QLDB** | Immutable ledger |
| **RDS Proxy** | Connection pooling — fixes **Lambda connection exhaustion** |

**LSI** = at creation only, same PK. **GSI** = any time, any keys.

## Integration

| Service | One line |
|---------|----------|
| **SQS** | Queue. **Visibility timeout**, DLQ, long polling. FIFO = order + dedup |
| **SNS** | Pub/sub fan-out. No retention — **fan out to SQS** for durability |
| **EventBridge** | Event bus, content routing, AWS sources, archive/replay, **Scheduler** |
| **Kinesis Data Streams** | Ordered, **replayable**, shards, multiple consumers |
| **Kinesis Firehose** | **Managed delivery** to S3/Redshift/OpenSearch, ~60 s |
| **MSK** | Managed Kafka |
| **Amazon MQ** | **JMS/AMQP/MQTT** — migrating existing brokers |
| **Step Functions** | Orchestration. Standard (1 yr) / Express (5 min) |

## Edge and DNS

| Item | Key fact |
|------|----------|
| **Route 53** | **100% SLA**. Alias for the **zone apex** (CNAME not allowed) |
| Routing policies | simple · **weighted (canary)** · **latency** · **failover (DR)** · geolocation (compliance) · geoproximity · multivalue |
| **CloudFront** | CDN, **cert must be in us-east-1**, OAC, signed URLs/cookies |
| **Global Accelerator** | **2 static anycast IPs**, TCP/UDP, **no caching**, seconds to fail over |

## Security

| Service | One line |
|---------|----------|
| **IAM** | Explicit **Deny always wins**; default deny; roles over keys |
| **STS** | `AssumeRole`, `AssumeRoleWithWebIdentity` (OIDC), `GetCallerIdentity` |
| **SCP** | Org guardrail — **filters, never grants** |
| **Permissions boundary** | **Maximum** permissions for an identity |
| **KMS** | Envelope encryption (`GenerateDataKey` + `Decrypt`), key policies |
| **CloudHSM** | Dedicated FIPS 140-2 L3; **AWS has no access** |
| **Secrets Manager** | **Automatic rotation**, RDS integration, ~$0.40/secret |
| **Parameter Store** | Config + secrets, **standard tier free**, no auto-rotation |
| **ACM** | Free public certs, auto-renew, **cannot export** |
| **WAF** | L7 — SQLi, XSS, rate limiting, geo-match |
| **Shield** | Standard free; **Advanced = DRT + cost protection** |
| **GuardDuty** | **Threat detection** from logs, no agent |
| **Macie** | **PII in S3** |
| **Inspector** | **CVEs** in EC2, ECR, Lambda |
| **Security Hub** | **Aggregates** findings |
| **Firewall Manager** | WAF/Shield/SG policy **across an org** |

## Management

| Service | One line |
|---------|----------|
| **CloudWatch** | **Performance** — metrics, logs, alarms. **Memory/disk need the agent** |
| **CloudTrail** | **Who called the API**. Data events **off by default** |
| **AWS Config** | **Configuration state + compliance**. Detects, does not prevent |
| **X-Ray** | Distributed tracing — **find the slow service** |
| **Systems Manager** | **Session Manager (no bastion)**, Patch Manager, Parameter Store |
| **Trusted Advisor** | Full checks need **Business+** support |
| **Compute Optimizer** | ML right-sizing |
| **Control Tower** | Multi-account landing zone |
| **Service Catalog** | Approved products only |

## DR

| Strategy | RTO | Cost |
|----------|-----|------|
| **Backup & Restore** | Hours | $ |
| **Pilot Light** | Tens of minutes | $$ |
| **Warm Standby** | Minutes | $$$ |
| **Multi-Site Active/Active** | Near zero | $$$$ |

**RPO** = data loss (replication frequency). **RTO** = downtime
(pre-provisioning).

**Multi-AZ = HA. Multi-region = DR.**

## Migration

| Need | Service |
|------|---------|
| Database, minimal downtime | **DMS** (+ **SCT** if heterogeneous) |
| Lift-and-shift servers | **MGN** |
| File/object transfer, ongoing | **DataSync** |
| Huge data, poor bandwidth | **Snowball / Snowmobile** |
| SFTP for partners | **Transfer Family** |
| On-prem keeps local access | **Storage Gateway** |

**Rule of thumb:** more than ~1 week over the network → **Snow**.

## Analytics

| Need | Service |
|------|---------|
| SQL over S3, serverless | **Athena** (Parquet + partitioning to cut cost) |
| Schema discovery, ETL | **Glue** |
| Spark/Hadoop | **EMR** |
| BI warehouse | **Redshift** |
| Search and log dashboards | **OpenSearch** |
| Dashboards for business users | **QuickSight** |
| Lake permissions, column-level | **Lake Formation** |

## Numbers worth memorising

| Thing | Value |
|-------|-------|
| Lambda timeout | **15 minutes** |
| Lambda default concurrency | 1,000/region |
| SQS message retention | 1 min – **14 days** |
| SQS max message size | 256 KB |
| SQS long polling max | 20 s |
| S3 max object | **5 TB** |
| S3 single PUT max | 5 GB |
| S3 per-prefix | 3,500 write / 5,500 read per second |
| DynamoDB item | 400 KB |
| Reserved IPs per subnet | **5** |
| VPC CIDR range | /16 to /28 |
| Aurora replicas | 15 |
| Aurora storage | 128 TB |
| RDS backup retention | 0–35 days |
| Spot warning | **2 minutes** |
| Kinesis retention | 1–365 days |
| Kinesis shard | 1 MB/s in, 2 MB/s out |
| EBS io2 Block Express | 256,000 IOPS |
| Exam | 65 Q, 130 min, 720/1000 |

---

Previous: [18 — Exam strategy](18-exam-strategy.md) ·
Next: [20 — Practice scenarios](20-practice-scenarios.md)
