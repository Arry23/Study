# 12 — Security services

Part of the 30% Security domain. Chapter 02 covered identity; this covers
encryption, secrets, and the detection and protection services.

## KMS

Managed keys for encrypting data across AWS.

### Key types

| Type | Rotation | Policy control | Cost |
|------|----------|----------------|------|
| **AWS owned** | AWS | None | Free |
| **AWS managed** (`aws/s3`) | **Automatic, yearly** | None | Free |
| **Customer managed (CMK)** | **Optional**, configurable | **Full key policy** | $1/month + requests |

> **Exam tip.** "Must control who can use the key / need an audit trail of key
> usage / need to rotate on our own schedule" → **customer managed key**. If
> the requirement is just "encrypted at rest", an AWS managed key suffices and
> is free.

### Key policies

A KMS key policy is a **resource-based policy** and is the primary access
control. Unlike most services, an IAM policy alone is not enough — the key
policy must also allow it.

> **Trap.** Locking yourself out of a CMK is permanent. Always leave the
> account root principal in the key policy.

### Envelope encryption

```
KMS key (never leaves KMS)
  └─ encrypts → data key
                 └─ encrypts → your data (locally, any size)
```

KMS encrypts up to **4 KB** directly. Anything larger uses `GenerateDataKey`
and encrypts locally. This is exactly what the workspace's IAM policy grants:

```python
# common/code/backend/common_utilities/dae_utils.py — policy actions
"kms:Decrypt", "kms:GenerateDataKey"
```

Those two actions together are the signature of envelope encryption; seeing
them tells you S3 objects are encrypted with SSE-KMS.

### Multi-Region keys

Related keys in several regions sharing key material — so data encrypted in
one region can be decrypted in another. Needed for cross-region replication of
encrypted data and multi-region DR.

### CloudHSM

Dedicated, single-tenant, **FIPS 140-2 Level 3** hardware. **You** control the
keys; AWS has no access.

> **Exam tip.** "Must control the HSM / regulatory requirement for a dedicated
> HSM / AWS must have no access to keys" → **CloudHSM**. Otherwise **KMS**.

## Secrets Manager vs Parameter Store

| | **Secrets Manager** | **Parameter Store** |
|---|---|---|
| Purpose | Secrets | Configuration **and** secrets |
| **Automatic rotation** | **Yes**, built-in with Lambda | No (manual/custom) |
| Cross-account | Yes | Advanced tier |
| Cost | ~$0.40/secret/month | **Standard tier free** |
| Size | 64 KB | 4 KB standard, 8 KB advanced |
| RDS integration | **Native** | No |

> **Exam tip.** "Rotate database credentials automatically" → **Secrets
> Manager**. "Store configuration values cheaply" → **Parameter Store**.
> Cost-optimization questions often prefer Parameter Store when rotation is
> not required.

This workspace uses Secrets Manager for API keys and database credentials:

```python
# arifa/Code/src/arifa_agent/secrets.py
boto3.client("secretsmanager", region_name=region_name)
```

and manages the **containers** in Terraform while keeping values out of state:

```hcl
# arifa-infra/modules/secrets/main.tf
# It never manages the secret value (no aws_secretsmanager_secret_version),
# so values stay out of code and state.
resource "aws_secretsmanager_secret" "this" {
  for_each   = var.secrets
  name       = each.key
  kms_key_id = each.value.kms_key_id
}
```

That separation — infrastructure manages the container, humans or a rotation
Lambda manage the value — is exactly the pattern the exam considers correct.

## ACM

Free public TLS certificates, auto-renewed.

| Use | Where the certificate must live |
|-----|-------------------------------|
| CloudFront | **us-east-1** |
| ALB / NLB / API Gateway | The **same region** as the resource |

> **Trap.** ACM public certificates **cannot be exported**. For EC2 or
> on-premises use, you need **ACM Private CA** or a third-party certificate.

## Edge protection

### WAF

Layer 7 firewall for CloudFront, ALB, API Gateway, AppSync, Cognito.

Rules match on IP, country, headers, body, URI, rate, SQL injection, XSS.
**Managed rule groups** cover OWASP Top 10, bad bots, known-bad inputs.

> **Exam tip.** "Block SQL injection / XSS / specific URI patterns" → **WAF**.
> "Rate-limit a single IP" → WAF **rate-based rule**.

### Shield

| | Shield Standard | Shield Advanced |
|---|---|---|
| Cost | **Free, automatic** | $3,000/month |
| Protects | Layer 3/4 | L3/4/7, plus WAF included |
| Extras | — | 24/7 DDoS Response Team, **cost protection** |

> **Exam tip.** "Sophisticated DDoS protection with expert support and billing
> protection against scaling costs during an attack" → **Shield Advanced**.

### Firewall Manager

Centrally manage WAF rules, Shield Advanced, security groups and Network
Firewall **across an Organization**.

> **Exam tip.** "Apply the same WAF rules to all accounts, including new ones"
> → **Firewall Manager**.

### Network Firewall

Stateful VPC-level firewall with IPS/IDS, domain filtering, deep packet
inspection — beyond what security groups and NACLs can do.

## Detection services

| Service | Detects | Source |
|---------|---------|--------|
| **GuardDuty** | Threats: crypto mining, compromised credentials, recon | CloudTrail, VPC Flow Logs, DNS logs, EKS audit, S3 |
| **Macie** | **Sensitive data (PII)** in **S3** | S3 objects, ML classification |
| **Inspector** | **Vulnerabilities / CVEs** | EC2, **ECR images**, Lambda |
| **Detective** | Root-cause analysis of findings | Graph of log data |
| **Security Hub** | **Aggregates** findings, scores against standards | All of the above + partners |

> **Exam tip — memorise these four:**
> - "Unusual API calls, compromised instance, crypto mining" → **GuardDuty**
> - "Find credit card numbers / PII in S3" → **Macie**
> - "Unpatched CVEs in instances or container images" → **Inspector**
> - "Single pane of glass across security findings and compliance standards" →
>   **Security Hub**

GuardDuty requires **no agent** — it reads logs AWS already has.

## Compliance and audit

| Service | Purpose |
|---------|---------|
| **AWS Config** | Resource configuration history and **compliance rules** |
| **CloudTrail** | API call audit log |
| **Audit Manager** | Collect evidence for audits continuously |
| **Artifact** | Download AWS compliance reports (SOC, ISO, PCI) |

Covered further in [chapter 13](13-monitoring-and-governance.md).

> **Exam tip — the trio that gets confused:**
> - **CloudTrail** = *who did what* (API calls)
> - **Config** = *what does it look like now, and did it change* (configuration)
> - **CloudWatch** = *how is it performing* (metrics and logs)

## Defence in depth — a worked example

The state bucket in this workspace layers controls, which is what the exam
means by defence in depth:

| Layer | Control |
|-------|---------|
| Network | Private access; bucket policy denies non-TLS |
| Identity | IAM role scoped to `envs/*`, explicit Deny on secret values |
| Resource | Block Public Access, ownership enforced |
| Data | SSE at rest, versioning for recovery |
| Detection | CloudTrail; Access Analyzer would flag external sharing |
| Recovery | Versioning + lifecycle, `prevent_destroy` |

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| Control who can decrypt, with an audit trail | **Customer managed KMS key** |
| Dedicated single-tenant HSM, AWS no access | **CloudHSM** |
| Rotate RDS credentials automatically | **Secrets Manager** |
| Cheap config storage | **Parameter Store** |
| Free auto-renewing TLS for an ALB | **ACM** (same region) |
| TLS for CloudFront | ACM in **us-east-1** |
| Certificate for EC2 / on-premises | **ACM Private CA** or third-party |
| Block SQLi and XSS | **WAF** |
| Rate-limit abusive IPs | **WAF rate-based rule** |
| DDoS with expert support and cost protection | **Shield Advanced** |
| Same WAF rules across all accounts | **Firewall Manager** |
| Detect compromised credentials | **GuardDuty** |
| Find PII in S3 | **Macie** |
| Find CVEs in container images | **Inspector** (or ECR scanning) |
| Aggregate all security findings | **Security Hub** |
| Prove a resource never became public | **AWS Config** rule + history |
| Who deleted the bucket? | **CloudTrail** |

## Check yourself

1. AWS managed vs customer managed key — when do you need the latter?
2. What is envelope encryption, and which two KMS actions signal it?
3. When is CloudHSM required rather than KMS?
4. Secrets Manager vs Parameter Store — the deciding feature?
5. Why can't you export an ACM public certificate?
6. Where must a CloudFront certificate be issued?
7. GuardDuty vs Macie vs Inspector — one scenario each.
8. CloudTrail vs Config vs CloudWatch?

---

Previous: [11 — DNS, CDN and global reach](11-dns-cdn-and-global.md) ·
Next: [13 — Monitoring and governance](13-monitoring-and-governance.md)
