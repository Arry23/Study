# 03 — VPC networking

Networking questions are everywhere on this exam, and they are the ones people
get wrong. Be able to draw the standard diagram from memory.

## The standard VPC

```
VPC 10.0.0.0/16
│
├── AZ-a
│   ├── Public subnet  10.0.1.0/24   → route 0.0.0.0/0 → IGW
│   │     └── NAT gateway, ALB node
│   └── Private subnet 10.0.11.0/24  → route 0.0.0.0/0 → NAT-a
│         └── app servers
│
├── AZ-b
│   ├── Public subnet  10.0.2.0/24   → IGW
│   │     └── NAT gateway, ALB node
│   └── Private subnet 10.0.12.0/24  → NAT-b
│         └── app servers
│
└── Internet Gateway (IGW)
```

**A subnet is public if and only if its route table has a route to an Internet
Gateway.** That is the whole definition. Not a name, not a flag.

## CIDR and subnets

VPC CIDR: `/16` to `/28`. Cannot be changed after creation, but you can add
secondary CIDR blocks.

**AWS reserves 5 IPs per subnet:**

| Address in 10.0.1.0/24 | Use |
|---|---|
| 10.0.1.0 | Network address |
| 10.0.1.1 | VPC router |
| 10.0.1.2 | DNS |
| 10.0.1.3 | Reserved for future |
| 10.0.1.255 | Broadcast (not supported, still reserved) |

So a `/24` gives **251** usable addresses, not 256.

> **Exam tip.** A question about running out of IPs in a subnet is usually
> about either these 5 reserved addresses or about EKS pods consuming VPC IPs.

Sizing quick reference:

| CIDR | Total | Usable |
|------|-------|--------|
| /28 | 16 | 11 |
| /24 | 256 | 251 |
| /20 | 4,096 | 4,091 |
| /16 | 65,536 | 65,531 |

## Internet access

| Component | Direction | Where it lives |
|-----------|-----------|----------------|
| **Internet Gateway** | In and out | VPC — one per VPC, horizontally scaled, no bottleneck |
| **NAT Gateway** | **Outbound only** | A **public subnet**, per AZ. Managed, scales to 45 Gbps |
| **NAT Instance** | Outbound only | An EC2 you manage. Legacy |
| **Egress-only IGW** | Outbound only, **IPv6** | VPC |

> **Exam tip — the classic.** A NAT gateway must live in a **public** subnet,
> and private subnets route to it. Put one **per AZ**, otherwise an AZ failure
> breaks outbound access for the surviving AZ *and* you pay cross-AZ data
> charges.

> **Trap.** NAT gateways are a common cost-optimization answer. They charge
> hourly **and** per GB processed. If the traffic is to S3 or DynamoDB, a
> **gateway VPC endpoint** removes it entirely and is free.

## Security groups vs NACLs

The single most tested comparison on the exam.

| | Security group | Network ACL |
|---|---|---|
| Operates at | **ENI / instance** | **Subnet** |
| **Stateful?** | **Yes** | **No** |
| Rules | **Allow only** | Allow **and Deny** |
| Evaluation | All rules together | **Numbered order**, first match wins |
| Default | Deny all in, allow all out | Default NACL allows all; custom denies all |

**Stateful** means: allow traffic in, and the response is automatically
allowed out. You never write a return rule.

**Stateless** means: you must write **both** directions, including
**ephemeral ports** (1024–65535) for return traffic.

> **Exam tip.** "Traffic reaches the instance but responses do not return" →
> a NACL missing the ephemeral port range on the outbound side.

> **Exam tip.** "Block a specific malicious IP" → **NACL**, because security
> groups cannot Deny. This is the main reason NACLs appear in answers.

Security groups can reference **other security groups** — the idiomatic way to
express "the app tier may talk to the database tier":

```
sg-db:  inbound 5432 from sg-app
sg-app: inbound 443  from sg-alb
sg-alb: inbound 443  from 0.0.0.0/0
```

## VPC endpoints

Private connectivity to AWS services without traversing the internet.

| Type | Services | How it works | Cost |
|------|----------|--------------|------|
| **Gateway endpoint** | **S3 and DynamoDB only** | A route table entry | **Free** |
| **Interface endpoint** (PrivateLink) | Most other services | An ENI with a private IP in your subnet | Hourly + per GB |

> **Exam tip.** Memorise that gateway endpoints support exactly **two**
> services: S3 and DynamoDB. Everything else is an interface endpoint.

> **Exam tip.** "Private subnet must reach S3 without internet access" and/or
> "reduce NAT gateway costs" → **gateway endpoint for S3**.

**PrivateLink** also exposes *your own* service to other VPCs or accounts
privately, via a Network Load Balancer plus an endpoint service.

## Connecting networks

| Option | Use | Transitive? |
|--------|-----|-------------|
| **VPC Peering** | Two VPCs, any account/region | **No** |
| **Transit Gateway** | Many VPCs, hub-and-spoke | **Yes** |
| **Site-to-Site VPN** | On-premises over internet, IPsec | — |
| **Direct Connect** | Dedicated private line to AWS | — |
| **PrivateLink** | Expose one service privately | — |

> **Exam tip — peering is not transitive.** A↔B and B↔C does **not** give
> A↔C. With more than a handful of VPCs, the answer is **Transit Gateway**.

> **Trap.** Peered VPCs must have **non-overlapping CIDRs**. A question where
> both VPCs are `10.0.0.0/16` is testing this.

### Direct Connect vs VPN

| | Direct Connect | Site-to-Site VPN |
|---|---|---|
| Medium | Dedicated private circuit | Public internet |
| Bandwidth | 50 Mbps – 100 Gbps | Up to ~1.25 Gbps per tunnel |
| Latency | Consistent | Variable |
| Setup | Weeks to months | Minutes |
| Encryption | **Not by default** (add MACsec or VPN over DX) | IPsec |

> **Exam tip.** "Consistent low latency / high bandwidth hybrid" → Direct
> Connect. "Quickly" or "backup connection" → VPN. Best practice for critical
> hybrid: **Direct Connect with a VPN as failover**.

## Routing

Each subnet associates with exactly one route table. Most specific prefix wins.

```
Destination        Target
10.0.0.0/16        local            ← always present, cannot be removed
0.0.0.0/0          igw-xxxx         ← makes the subnet public
pl-xxxx (S3)       vpce-xxxx        ← gateway endpoint
192.168.0.0/16     tgw-xxxx
```

> **Trap.** The `local` route cannot be deleted or overridden. Traffic within
> the VPC CIDR never leaves the VPC.

## DNS

Two VPC attributes:

- `enableDnsSupport` — the Amazon-provided resolver at `VPC CIDR base + 2`
- `enableDnsHostnames` — instances get public DNS names

**Route 53 Resolver** endpoints handle hybrid DNS:

- **Inbound endpoint** — on-premises resolves AWS private zones
- **Outbound endpoint** — AWS resolves on-premises zones

> **Exam tip.** "On-premises servers cannot resolve private hosted zone names"
> → Route 53 Resolver **inbound** endpoint.

## Monitoring

**VPC Flow Logs** — IP traffic metadata to CloudWatch Logs, S3 or Firehose.

> **Exam tip.** Flow Logs capture **metadata**, not packet contents. For
> payloads you need **VPC Traffic Mirroring**. "Why is this connection being
> rejected?" → Flow Logs, look for `REJECT`.

**Reachability Analyzer** — static analysis of whether a path is open, and
which component blocks it. **Network Access Analyzer** — find unintended paths.

## Elastic IPs and addresses

| Type | Behaviour |
|------|-----------|
| Private IP | Persists through stop/start |
| Public IP | **Changes** on stop/start |
| **Elastic IP** | Static, yours until released. Charged when **not** attached |

> **Exam tip.** "The IP changes after a reboot" → Elastic IP. But for most
> architectures the better answer is a **load balancer** or **Route 53**, not
> an EIP.

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| Private subnet needs outbound internet | NAT gateway in a public subnet, one per AZ |
| Private subnet needs S3, no internet | **Gateway VPC endpoint** |
| Private access to SQS/KMS/Secrets Manager | **Interface endpoint** (PrivateLink) |
| Block one malicious IP | **NACL** deny rule |
| App tier may reach DB tier only | Security group referencing a security group |
| 50 VPCs need to interconnect | **Transit Gateway** |
| Two VPCs, simple, cheap | **VPC peering** |
| Consistent hybrid bandwidth | **Direct Connect** |
| Hybrid quickly / as backup | **Site-to-Site VPN** |
| Diagnose rejected traffic | **VPC Flow Logs** |
| Expose my service to another VPC privately | **PrivateLink** |
| Reduce NAT data processing cost | Gateway endpoints; keep traffic in-AZ |

## Check yourself

1. What exactly makes a subnet public?
2. How many usable IPs in a `/24`? Why?
3. Stateful vs stateless — which needs ephemeral port rules?
4. Which two services support gateway endpoints?
5. Why does VPC peering not scale past a few VPCs?
6. Where must a NAT gateway live, and how many do you need?
7. Direct Connect — is it encrypted by default?
8. Flow Logs vs Traffic Mirroring?

---

Previous: [02 — IAM and identity](02-iam-and-identity.md) ·
Next: [04 — EC2 and compute](04-ec2-and-compute.md)
