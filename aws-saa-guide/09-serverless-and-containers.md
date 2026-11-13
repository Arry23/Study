# 09 — Serverless and containers

"Reduce operational overhead" is the phrase that sends you here.

## Lambda

Run code without managing servers. Pay per request and per GB-second.

| Limit | Value |
|-------|-------|
| Timeout | **15 minutes** max |
| Memory | 128 MB – 10,240 MB |
| `/tmp` | 512 MB – 10 GB |
| Deployment package | 50 MB zipped, 250 MB unzipped, **10 GB as a container image** |
| Payload | 6 MB synchronous, 256 KB asynchronous |
| Concurrency | 1,000 per region by default (soft limit) |

> **Exam tip — the 15-minute limit.** Any question where processing might
> exceed 15 minutes rules Lambda out. Move to **Fargate**, **Batch**, or
> **Step Functions** orchestrating smaller Lambdas.

**CPU scales with memory.** Allocating more memory is often the way to make a
Lambda faster *and* cheaper, because duration drops.

### Concurrency

| | Purpose |
|---|---|
| **Reserved concurrency** | Cap a function's concurrency; also guarantees it |
| **Provisioned concurrency** | Pre-initialised environments — **eliminates cold starts** |

> **Exam tip.** "Latency spikes on first invocation" → **cold start** →
> **provisioned concurrency**. "One function is starving the others" →
> **reserved concurrency**.

### Invocation models

| Model | Examples | Retries |
|-------|----------|---------|
| **Synchronous** | API Gateway, ALB, direct invoke | Caller handles |
| **Asynchronous** | S3, SNS, EventBridge | **2 automatic retries**, then DLQ |
| **Poll-based** | SQS, Kinesis, DynamoDB Streams | Lambda polls; batch semantics |

> **Exam tip.** Always pair asynchronous Lambda with a **dead-letter queue**
> or **on-failure destination**, or failed events vanish.

### Lambda in a VPC

Needed to reach private resources (RDS, ElastiCache). Consequence: the function
loses direct internet access and needs a **NAT gateway** or VPC endpoints.

> **Trap.** "Lambda in a VPC cannot reach the internet / a public API" →
> NAT gateway in a public subnet, or an interface endpoint.

### Lambda@Edge vs CloudFront Functions

| | CloudFront Functions | Lambda@Edge |
|---|---|---|
| Runtime | JavaScript, sub-millisecond | Node/Python, up to 5–30 s |
| Use | Header manipulation, URL rewrite, simple auth | Heavier logic, origin requests |
| Cost | Much cheaper | Higher |

## API Gateway

Managed front door for APIs: throttling, auth, caching, transformation.

| Type | Use |
|------|-----|
| **HTTP API** | Lower cost, lower latency, modern. **Default choice** |
| **REST API** | Full features: API keys, usage plans, request validation, WAF |
| **WebSocket API** | Bidirectional, real-time |

> **Exam tip.** "Lowest cost / simplest proxy to Lambda" → **HTTP API**.
> "API keys, usage plans, request/response transformation, caching" →
> **REST API**.

Authorisation options: IAM, **Cognito user pools**, **Lambda authorizer**
(custom), or none.

> **Exam tip.** "Third-party token / custom auth logic" → **Lambda
> authorizer**. "Sign-up and sign-in for app users" → **Cognito**.

**Throttling** and **usage plans** protect backends. **Caching** reduces
backend calls.

## Containers

### ECS

AWS-native orchestration.

| Launch type | You manage |
|-------------|------------|
| **EC2** | The instances (patching, scaling, capacity) |
| **Fargate** | **Nothing** — serverless containers |

Concepts: **task definition** (the blueprint), **task** (a running instance of
it), **service** (keeps N tasks running behind a load balancer).

Two distinct roles — the exam tests this:

| Role | Used by |
|------|---------|
| **Task execution role** | The ECS **agent**: pull from ECR, write logs |
| **Task role** | **Your application**: call S3, DynamoDB, etc. |

> **Trap.** "Container cannot access S3" → the **task role**, not the
> execution role.

### EKS

Managed Kubernetes. Choose it when you already have Kubernetes expertise or
need portability.

| Mode | You manage |
|------|------------|
| Self-managed nodes | Everything above the control plane |
| **Managed node groups** | AWS handles node lifecycle |
| **Fargate** | No nodes at all |

**IRSA — IAM Roles for Service Accounts** maps a Kubernetes service account to
an IAM role via the cluster's OIDC provider. This workspace builds exactly that:

```python
# common/code/backend/common_utilities/dae_utils.py
eks_client = boto3.client("eks", region_name=aws_region)
response = eks_client.describe_cluster(name=eks_cluster_name)
oidc_issuer = response["cluster"]["identity"]["oidc"]["issuer"]
```

The role it then creates grants `s3:GetObject`, `s3:PutObject`, `kms:Decrypt`
and `kms:GenerateDataKey` — least privilege for one workload rather than for
the whole node.

> **Exam tip.** "Grant a pod access to AWS services without node-wide
> credentials" → **IRSA** (or EKS Pod Identity). An instance profile would
> grant every pod on the node the same access.

### ECR

Private container registry. Image scanning, lifecycle policies, cross-region
replication, IAM-based access.

> **Exam tip.** "Scan container images for vulnerabilities" → **ECR image
> scanning** (basic or enhanced via Inspector).

### ECS vs EKS vs Fargate vs Lambda

```
Event-driven, short, minimal ops         -> Lambda
Containers, no infrastructure            -> Fargate (on ECS or EKS)
Containers, need node control / GPUs     -> ECS or EKS on EC2
Already on Kubernetes / multi-cloud      -> EKS
Simplest AWS-native containers           -> ECS
Web app from a container or repo         -> App Runner
Batch jobs at scale                      -> AWS Batch
```

## Step Functions

Serverless orchestration as a state machine. Handles retries, branching,
parallelism, error handling and waits.

| Workflow | Duration | Semantics | Use |
|----------|----------|-----------|-----|
| **Standard** | Up to **1 year** | Exactly-once | Long-running, auditable |
| **Express** | Up to 5 minutes | At-least-once | High-volume event processing |

> **Exam tip.** "Coordinate multiple Lambdas with retries and error handling
> without writing orchestration code" → **Step Functions**. "Workflow must run
> for days" → **Standard**, which also sidesteps the 15-minute Lambda limit.

## Other serverless compute

| Service | Use |
|---------|-----|
| **AWS Batch** | Managed batch at scale, Spot-friendly |
| **App Runner** | Container or repo → HTTPS service, minimal config |
| **Lightsail** | Simple VPS with predictable pricing |
| **Fargate Spot** | Interruptible containers, big discount |

## Decoupling and statelessness

Serverless architectures assume stateless compute:

| State | Put it in |
|-------|-----------|
| Sessions | **ElastiCache** or **DynamoDB** |
| Files | **S3** or **EFS** (Lambda and Fargate can mount EFS) |
| Queues/buffers | **SQS** |
| Configuration | **Parameter Store** / **AppConfig** |
| Secrets | **Secrets Manager** |

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| Run 10 s of code on each S3 upload | **Lambda** |
| Job takes 45 minutes | **Fargate**, **Batch**, or Step Functions |
| Eliminate cold starts | **Provisioned concurrency** |
| Lambda needs RDS in a private subnet | Put Lambda **in the VPC** |
| Lambda in VPC needs internet | **NAT gateway** or VPC endpoint |
| Cheapest Lambda-backed API | **HTTP API** |
| API keys and usage plans | **REST API** |
| Custom token validation | **Lambda authorizer** |
| Containers without managing servers | **Fargate** |
| Pod needs S3 access, least privilege | **IRSA** |
| Container cannot reach S3 | Fix the **task role** |
| Orchestrate a multi-step workflow | **Step Functions** |
| Scan images for CVEs | **ECR scanning** |
| Interruptible container batch | **Fargate Spot** or Batch on Spot |

## Check yourself

1. What is Lambda's maximum timeout, and what do you use beyond it?
2. Why can increasing Lambda memory reduce cost?
3. Reserved vs provisioned concurrency?
4. What breaks when you put Lambda in a VPC, and how do you fix it?
5. ECS task role vs task execution role?
6. What problem does IRSA solve that an instance profile does not?
7. Step Functions Standard vs Express?
8. HTTP API vs REST API — two deciding features.

---

Previous: [08 — Databases](08-databases.md) ·
Next: [10 — Application integration](10-application-integration.md)
