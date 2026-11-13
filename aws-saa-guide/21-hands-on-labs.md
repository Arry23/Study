# 21 — Hands-on labs

Reading about a VPC teaches you the words. Building one teaches you the exam.

## Read this first — safety and cost

This workspace points at a **shared Roche Cloud Platform account**
(`483075907105`) with platform guardrails. **Do not build lab infrastructure
there.** It is not yours, it is governed by SCPs, and a stray NAT gateway shows
up on someone's bill.

The labs are therefore tiered:

| Tier | Where | Cost | Needs |
|------|-------|------|-------|
| **0 — Paper** | Nowhere | Free | Pen and paper |
| **1 — Read-only** | The real account | Free | AWS CLI + your SSO access |
| **2 — LocalStack** | Your laptop | Free | Podman (you have it) |
| **3 — Build** | **Your own** free-tier account | Small, if you follow the rules | A personal AWS account |

### The things that cost money even when idle

> **Learn these before tier 3. They are the classic surprise bills.**

| Resource | Cost if left running | Free tier? |
|---|---|---|
| **NAT Gateway** | **~$33/month** + per GB | **No** |
| **Application Load Balancer** | **~$17/month** + LCU | **No** |
| **Unattached Elastic IP** | ~$4/month | No |
| **RDS Multi-AZ** | 2× instance | No (single-AZ db.t3.micro is) |
| **Provisioned IOPS (io1/io2)** | Per IOPS/month | No |
| **EKS control plane** | **~$73/month** | **No** |
| **Interface VPC endpoint** | ~$7/month each | No |

EC2 `t2.micro`/`t3.micro` (750 h/month), S3 (5 GB), Lambda (1M requests) and
DynamoDB (25 GB) **are** free tier for 12 months.

### Rules for tier 3

1. **Set a billing alarm first.** Lab 12 does this, and it is lab 12 for a
   reason — do it before anything else.
2. **One lab at a time.** Build, learn, **destroy**.
3. **Tag everything** `Purpose=lab` so you can find strays.
4. **Run the teardown checklist** at the end of every session.
5. Prefer **Terraform** for labs — `terraform destroy` is a reliable delete.

---

# Tier 0 — Paper labs

No account needed. These are the highest-value-per-minute labs for the exam.

## Lab 1 — Draw the standard VPC

**Covers:** [ch. 03](03-vpc-networking.md)

From memory, on paper, draw a VPC `10.0.0.0/16` across **three** AZs with:

- Public and private subnets in each AZ, with CIDRs that do not overlap
- Internet gateway, NAT gateways (how many?), route tables (how many?)
- An ALB, an ASG, and an RDS Multi-AZ instance — each in the right subnet tier
- A gateway endpoint for S3
- Security groups: `sg-alb`, `sg-app`, `sg-db`, with rules referencing each
  other

Then annotate:

1. Exactly what makes each public subnet public
2. How many usable IPs in each `/24`, and why not 256
3. Where traffic from a private instance to S3 goes — and what it costs
4. What breaks if there is only one NAT gateway, in two different ways

**Verify:** three NAT gateways, four route tables (one per public tier shared,
or one per subnet), 251 usable IPs, and your S3 answer mentions **free**.

Redraw it until you can do it in five minutes. This single diagram underpins
perhaps a fifth of the exam.

---

## Lab 2 — CIDR and bandwidth arithmetic

**Covers:** [ch. 03](03-vpc-networking.md), [ch. 16](16-migration-and-transfer.md)

**Subnetting.** Split `10.0.0.0/16` into six subnets — three public `/24`, three
private `/20`. Write the ranges. How many addresses remain unallocated?

**Capacity.** An EKS cluster assigns pod IPs from the VPC. You expect 2,000
pods. Is a `/24` per subnet enough? What would you choose?

**Transfer.** For each, decide network or Snow, showing the arithmetic:

| Data | Link | Deadline |
|---|---|---|
| 8 TB | 1 Gbps | 1 week |
| 60 TB | 500 Mbps | 3 weeks |
| 400 TB | 200 Mbps | 1 month |
| 2 PB | 10 Gbps | 2 months |

**Verify:** rows 1 and 4 are network; 2 and 3 are Snow. Show seconds, not
vibes.

---

## Lab 3 — Policy evaluation puzzles

**Covers:** [ch. 02](02-iam-and-identity.md)

For each, state **allowed or denied** and **which rule decided**:

1. Identity policy allows `s3:GetObject` on `*`. Bucket policy says nothing.
2. Identity policy allows it. An SCP on the OU does not include S3.
3. Identity policy allows it. A permissions boundary allows only DynamoDB.
4. Nothing allows it. Nothing denies it.
5. Identity policy allows `s3:*`. Bucket policy explicitly denies
   `s3:DeleteObject`. The call is `DeleteObject`.
6. `ReadOnlyAccess` is attached. An inline policy explicitly denies
   `secretsmanager:GetSecretValue`. The call is `GetSecretValue`.
7. A role in account B is assumed by a principal in account A. The trust policy
   permits it; the permissions policy does not include the action.

**Verify:** 6 is exactly what
[`arifa-infra/bootstrap/github_oidc.tf`](../arifa-infra/bootstrap/github_oidc.tf)
does. 7 distinguishes trust policy from permissions policy — a frequent exam
question.

---

## Lab 4 — Architecture from requirements

**Covers:** everything

Design and draw a solution for each. One paragraph plus a diagram.

**A.** A global media site. Static assets and a REST API. Users in Europe, Asia
and the Americas. Must survive an AZ failure. Minimise latency and egress cost.

**B.** A nightly batch job processing 500 GB from S3, tolerant of interruption,
taking 2–6 hours. Minimise cost.

**C.** A regulated workload: seven-year retention, records immutable by anyone,
retrieval within 12 hours acceptable, auditors query twice a year with SQL.

**D.** A relational workload that must survive the loss of a **region** with
under one second of data loss.

**E.** On-premises file server, 40 TB, must keep working locally while data is
backed up to AWS. Link is 1 Gbps.

**Verify** against the decision tables:
A → CloudFront + ALB + ASG multi-AZ + Route 53 latency.
B → Spot (or Batch on Spot).
C → Glacier Deep Archive + Object Lock Compliance + Athena.
D → Aurora Global Database.
E → Storage Gateway (volume, cached) — **not** DataSync, because local access
must continue.

---

# Tier 1 — Read-only exploration

**Safe in the real account.** Every command below only reads.

## Lab 5 — Install the AWS CLI and look around

**Covers:** [ch. 01](01-fundamentals.md), [ch. 13](13-monitoring-and-governance.md)

The CLI is not installed on your machine. Install **AWS CLI v2**, then:

```bash
aws configure sso --profile arifa     # per arifa-infra/README.md
aws sso login --profile arifa
export AWS_PROFILE=arifa
```

Now explore — **all read-only**:

```bash
aws sts get-caller-identity
aws s3 ls
aws ec2 describe-vpcs            --query 'Vpcs[].{Id:VpcId,Cidr:CidrBlock,Default:IsDefault}'
aws ec2 describe-subnets         --query 'Subnets[].{Id:SubnetId,AZ:AvailabilityZone,Cidr:CidrBlock}'
aws ec2 describe-route-tables    --query 'RouteTables[].{Id:RouteTableId,Routes:Routes[].GatewayId}'
aws ec2 describe-security-groups --query 'SecurityGroups[].{Id:GroupId,Name:GroupName}'
aws iam list-roles               --query 'Roles[].RoleName'
aws secretsmanager list-secrets  --query 'SecretList[].Name'
```

**Answer from the output:**

1. Which subnets are public? Prove it from the route tables, not the names.
2. How many AZs are in use?
3. Find the SageMaker notebook's execution role. What does its name suggest
   about the database behind this project?
4. List the secrets. Can you read a **value**? Try — and explain the result.

> Question 4 is the lesson. If `get-secret-value` is denied, that is
> least privilege working. Do not try to work around it.

**Verify:** your answer to 1 cites an `igw-` target.

---

## Lab 6 — Read the Terraform, predict the console

**Covers:** [ch. 06](06-s3.md), [ch. 12](12-security-services.md)

Read [`arifa-infra/bootstrap/state.tf`](../arifa-infra/bootstrap/state.tf) and
**before** looking at AWS, write down what you expect:

1. Versioning — on or off?
2. Encryption — which type, and why not KMS?
3. Public access — which of the four blocks are set?
4. Lifecycle — what expires, and after how long?
5. The bucket policy — what does it deny, and on which condition key?

Then verify (read-only):

```bash
B=arifa-infrastructure-tfstate-483075907105-eu-central-1
aws s3api get-bucket-versioning          --bucket $B
aws s3api get-bucket-encryption          --bucket $B
aws s3api get-public-access-block        --bucket $B
aws s3api get-bucket-lifecycle-configuration --bucket $B
aws s3api get-bucket-policy              --bucket $B --query Policy --output text | python -m json.tool
```

**Verify:** your five predictions match. Any mismatch is either drift or a
misreading — both worth understanding.

---

# Tier 2 — LocalStack

Free, offline, destroys nothing real. Good for IAM, S3, SQS, DynamoDB and
Lambda practice.

## Lab 7 — Stand up LocalStack

**Covers:** [ch. 06](06-s3.md), [ch. 10](10-application-integration.md)

```bash
podman run --rm -d --name localstack -p 4566:4566 localstack/localstack:3
export AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test AWS_DEFAULT_REGION=eu-central-1
alias awsl='aws --endpoint-url=http://localhost:4566'
awsl sts get-caller-identity
```

Now practise S3 for real:

```bash
awsl s3 mb s3://lab-bucket
awsl s3api put-bucket-versioning --bucket lab-bucket \
    --versioning-configuration Status=Enabled

echo "v1" > f.txt && awsl s3 cp f.txt s3://lab-bucket/f.txt
echo "v2" > f.txt && awsl s3 cp f.txt s3://lab-bucket/f.txt

awsl s3api list-object-versions --bucket lab-bucket
awsl s3 rm s3://lab-bucket/f.txt
awsl s3api list-object-versions --bucket lab-bucket     # look for the DeleteMarker
```

**Verify:** you can explain, from the output, what a delete marker is and how
to restore the object. Then restore it by deleting the marker.

**Then:** add a lifecycle rule expiring noncurrent versions after 1 day and
confirm it is stored.

---

## Lab 8 — SQS semantics

**Covers:** [ch. 10](10-application-integration.md)

```bash
Q=$(awsl sqs create-queue --queue-name lab-q --query QueueUrl --output text)
awsl sqs send-message --queue-url $Q --message-body "hello"

# Receive without deleting
awsl sqs receive-message --queue-url $Q

# Immediately receive again — what happens, and why?
awsl sqs receive-message --queue-url $Q
```

**Verify:** the second receive returns nothing. Explain which setting causes
that and what happens after it elapses.

**Then:**

1. Set the visibility timeout to 5 seconds, repeat, and observe redelivery.
2. Create a DLQ and a redrive policy with `maxReceiveCount=2`. Receive the same
   message three times without deleting. Where does it end up?
3. Create a FIFO queue. What must its name end with?

This lab makes "visibility timeout" and "DLQ" permanent knowledge rather than
flashcards.

---

## Lab 9 — IAM policy simulation

**Covers:** [ch. 02](02-iam-and-identity.md)

In LocalStack, create a role with `ReadOnlyAccess`-style permissions plus an
explicit Deny, mirroring the real CI role:

```bash
cat > deny.json <<'EOF'
{"Version":"2012-10-17","Statement":[
 {"Effect":"Allow","Action":"s3:*","Resource":"*"},
 {"Effect":"Deny","Action":"s3:DeleteObject","Resource":"*"}]}
EOF
awsl iam create-policy --policy-name lab-deny --policy-document file://deny.json
```

**Verify by reasoning, then by testing:** can this policy delete an object?
Which statement decided, and would adding another `Allow` change anything?

**Then:** write a policy that allows reading **only** objects under
`envs/*` and denies everything else — using `NotResource`, like
`github_oidc.tf` does.

---

# Tier 3 — Build it (your own account)

> **Only in a personal account. Set the billing alarm first (lab 12).**

## Lab 10 — VPC from scratch

**Covers:** [ch. 03](03-vpc-networking.md)

Build the lab 1 diagram by hand in the console — **once**, to feel it — then
destroy it and rebuild with Terraform.

Deliberately **omit the NAT gateway** at first:

1. Launch a `t3.micro` in a private subnet
2. Connect with **Session Manager** (needs the SSM role and endpoints, or a
   NAT). Why can't you SSH?
3. `curl https://example.com` — it fails. Why?
4. Add a NAT gateway. It works. **Note the hourly charge starting now.**
5. Add a **gateway endpoint for S3** and `aws s3 ls`. Compare the cost path.
6. **Delete the NAT gateway** before you stop for the day.

**Verify:** you can state what each of IGW, NAT, route table and endpoint
contributed, and roughly what the NAT cost you.

> **Teardown:** NAT gateway, EIP, endpoints, instance, then the VPC.

---

## Lab 11 — Highly available web tier

**Covers:** [ch. 05](05-load-balancing-and-autoscaling.md), [ch. 14](14-resilience-and-dr.md)

ALB + ASG across two AZs, serving a trivial page from user data.

1. ASG min 2, max 4, across two AZs
2. Target tracking on CPU at 50%
3. Health check type **EC2** initially
4. SSH in and `kill` the web server, leaving the instance running
5. Observe: the ALB marks the target unhealthy, but the ASG **does not replace
   it**. Why?
6. Switch the health check type to **ELB**. Repeat. Now it is replaced.
7. Terminate an instance manually and watch the ASG restore desired capacity

**Verify:** step 5 is the lesson — the default EC2 health check cannot see a
crashed application.

> **Teardown:** ASG (set desired/min to 0 first), ALB, target group, security
> groups. The ALB is the expensive part — delete it the same day.

---

## Lab 12 — Billing alarm (do this first)

**Covers:** [ch. 15](15-cost-optimization.md)

1. Enable billing alerts in Billing preferences
2. In **us-east-1** (billing metrics live only there), create a CloudWatch
   alarm on `EstimatedCharges` > $5
3. Send it to an SNS topic with your email; confirm the subscription
4. Create an **AWS Budget** at $10 with alerts at 50%, 80%, 100%
5. Open **Cost Explorer** and group by service

**Verify:** you receive the SNS confirmation email and the alarm shows `OK`.

**Then answer:** why must the billing alarm be in us-east-1?

---

## Lab 13 — S3 lifecycle and cost

**Covers:** [ch. 06](06-s3.md), [ch. 15](15-cost-optimization.md)

1. Create a bucket, enable versioning
2. Upload the same file five times
3. Start a multipart upload and **abort the client** mid-way
4. `aws s3api list-multipart-uploads` — see the orphan
5. `aws s3 ls --summarize --human-readable --recursive` vs
   `list-object-versions` — compare the totals
6. Add a lifecycle rule: noncurrent versions expire after 1 day, incomplete
   multipart uploads abort after 1 day
7. Add a transition to Standard-IA after 30 days

**Verify:** you can explain why step 5's two numbers differ — this is exactly
the surprise-bill scenario from practice question 10.

> **Teardown:** empty **all versions** (`aws s3 rm --recursive` is not enough —
> use a lifecycle rule or delete versions explicitly), then delete the bucket.

---

## Lab 14 — Serverless end to end

**Covers:** [ch. 09](09-serverless-and-containers.md), [ch. 10](10-application-integration.md)

Build: **S3 upload → event → Lambda → DynamoDB**, all free tier.

1. Bucket, Lambda (Python), DynamoDB table (on-demand)
2. Lambda execution role with **only** `dynamodb:PutItem` on that table and
   CloudWatch Logs
3. S3 event notification on `ObjectCreated`
4. Upload a file; confirm the item appears
5. **Remove** `dynamodb:PutItem` from the role. Upload again. Read the
   CloudWatch log
6. Restore it. Add a DLQ for the async invocation and force a failure

**Verify:** step 5's error names the exact denied action — the fastest way to
learn to read IAM failures.

> **Teardown:** Lambda, table, bucket, role, log group.

---

## Lab 15 — Break and diagnose

**Covers:** [ch. 03](03-vpc-networking.md), [ch. 18](18-exam-strategy.md)

Have someone (or a script) introduce one fault; you diagnose it with
**Reachability Analyzer** and **VPC Flow Logs**:

| Fault | Symptom |
|---|---|
| Remove the IGW route | |
| Security group missing port 443 inbound | |
| NACL missing ephemeral ports **outbound** | |
| ASG health check back to EC2 | |
| Subnet route to a deleted NAT | |
| Instance in the wrong subnet tier | |

**Verify:** for each, record which tool found it fastest. The NACL one is the
exam's favourite — note that traffic arrives but responses never return.

---

## Teardown checklist

Run this at the end of **every** tier 3 session:

```bash
aws ec2 describe-nat-gateways   --filter Name=state,Values=available
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'
aws ec2 describe-addresses      --query 'Addresses[?AssociationId==null].PublicIp'
aws ec2 describe-instances      --filters Name=instance-state-name,Values=running \
    --query 'Reservations[].Instances[].InstanceId'
aws rds describe-db-instances   --query 'DBInstances[].DBInstanceIdentifier'
aws ec2 describe-volumes        --filters Name=status,Values=available \
    --query 'Volumes[].VolumeId'
aws ec2 describe-vpc-endpoints  --query 'VpcEndpoints[?VpcEndpointType==`Interface`].VpcEndpointId'
```

**Anything returned is costing money.** Unattached EIPs, available (unattached)
volumes and idle NAT gateways are the usual stragglers.

Then check Cost Explorer the next morning — the only way to be sure.

---

## Suggested order

| If you have | Do |
|---|---|
| 30 minutes | Labs 1 and 3 — paper, highest value per minute |
| 2 hours | Labs 1–4 |
| An evening | Labs 5–9 (read-only + LocalStack, zero cost) |
| A weekend | Labs 12, 10, 11, 13, 14 — **billing alarm first** |

Tier 0 alone will move your practice-exam score more than any amount of video.
Tiers 2 and 3 make the knowledge stick.

---

Previous: [20 — Practice scenarios](20-practice-scenarios.md) ·
Back to the [index](README.md)
