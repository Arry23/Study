# 02 — IAM and identity

The largest exam domain. Learn the **policy evaluation logic** properly — more
questions turn on it than on any other single topic.

## The four building blocks

| | What it is | Has credentials? |
|---|---|---|
| **User** | A person or application identity | Yes — password and/or access keys |
| **Group** | A collection of users | No |
| **Role** | A set of permissions something *assumes* | No — issues **temporary** credentials |
| **Policy** | A JSON document granting or denying | n/a |

> **Exam tip.** Whenever an answer offers "create an IAM user and store the
> access key", look for the option using a **role** instead. Roles are almost
> always the correct answer: no long-lived credentials, automatic rotation.

## Policy anatomy

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "ReadStateBucket",
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::my-bucket/envs/*",
    "Condition": {
      "StringEquals": { "aws:PrincipalTag/team": "platform" }
    }
  }]
}
```

| Element | Meaning |
|---------|---------|
| `Effect` | `Allow` or `Deny` |
| `Action` | API operations (`s3:GetObject`, `s3:*`) |
| `Resource` | ARNs the statement applies to |
| `Principal` | **Who** — only in resource-based policies |
| `Condition` | Extra constraints |

Also `NotAction`, `NotResource`, `NotPrincipal` — "everything except". Used in
this workspace to deny reading every S3 object *except* the state path:

```hcl
# arifa-infra/bootstrap/github_oidc.tf
statement {
  sid           = "DenyDataReads"
  effect        = "Deny"
  actions       = ["s3:GetObject", "s3:GetObjectVersion"]
  not_resources = ["${aws_s3_bucket.state.arn}/envs/*"]
}
```

## Identity-based vs resource-based

| | Attached to | Has `Principal`? | Example |
|---|---|---|---|
| **Identity-based** | User, group, role | No | "This role may read that bucket" |
| **Resource-based** | The resource itself | **Yes** | Bucket policy, SQS queue policy, KMS key policy |

Resource-based policies exist on: S3, SQS, SNS, KMS, Secrets Manager, Lambda,
API Gateway, ECR, EFS, Glue, and a few others. **Not** on EC2, RDS or DynamoDB
tables.

> **Exam tip.** For **cross-account** access you generally need **both**: a
> resource-based policy on the target granting the external principal, and an
> identity-based policy on the caller allowing the action. The exception is KMS,
> where the key policy alone can be sufficient.

## Policy evaluation logic

Memorise this. It decides many questions.

```
1. Explicit DENY anywhere?            -> DENY. Final. Nothing overrides it.
2. SCP allows it? (if in an Org)      -> else DENY
3. Resource-based policy allows?      -> ALLOW
4. Identity-based policy allows?      -> ALLOW
5. Permissions boundary allows?       -> else DENY
6. Session policy allows?             -> else DENY
7. Otherwise                          -> implicit DENY
```

Two rules carry most of the weight:

- **Default is deny.** No matching Allow means denied.
- **An explicit Deny always wins**, regardless of any Allow.

This workspace uses that second rule deliberately. The CI role has
`ReadOnlyAccess` attached — which grants `secretsmanager:GetSecretValue` — and
then overrides it:

```hcl
statement {
  sid       = "DenySecretValues"
  effect    = "Deny"
  actions   = ["secretsmanager:GetSecretValue", "secretsmanager:BatchGetSecretValue"]
  resources = ["*"]
}
```

A plan reads *configuration*, never secret values, so the Deny is safe and the
broad managed policy stays convenient.

## Roles and STS

A role is assumed, producing temporary credentials from **STS**.

Two policies define a role:

1. **Trust policy** (resource-based) — *who may assume it*
2. **Permissions policy** (identity-based) — *what it may do once assumed*

```hcl
# Trust policy: who
data "aws_iam_policy_document" "github_plan_trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github[0].arn]
    }
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:${local.github_repository}:pull_request"]
    }
  }
}
```

> **Exam tip.** "Role is assumed but the caller gets AccessDenied" → check the
> **trust policy** first. "Caller assumed the role but an API call fails" →
> check the **permissions policy**.

### STS operations

| API | Use |
|-----|-----|
| `AssumeRole` | Cross-account, or privilege escalation within an account |
| `AssumeRoleWithWebIdentity` | OIDC federation — GitHub Actions, Cognito, Google |
| `AssumeRoleWithSAML` | SAML federation — AD FS, Okta |
| `GetSessionToken` | MFA-protected temporary credentials |
| `GetCallerIdentity` | Who am I? Never denied |

Both federation forms appear in this workspace: GitHub Actions uses
`AssumeRoleWithWebIdentity`, and `GetCallerIdentity` is used to discover the
account ID:

```python
# common/code/backend/common_utilities/dae_utils.py
sts_client = boto3.client("sts", region_name=aws_region)
account_id = sts_client.get_caller_identity()["Account"]
```

### Roles for AWS services

| Scenario | Mechanism |
|----------|-----------|
| EC2 needs S3 access | **Instance profile** — role attached to the instance |
| Lambda needs DynamoDB | **Execution role** |
| ECS task needs S3 | **Task role** (distinct from the task *execution* role) |
| **EKS pod** needs S3 | **IRSA** — IAM Roles for Service Accounts, via OIDC |

IRSA is exactly what `common/code/backend/common_utilities/dae_utils.py` sets
up: it reads the cluster's OIDC issuer and builds a role a Kubernetes service
account can assume.

```python
eks_client = boto3.client("eks", region_name=aws_region)
response = eks_client.describe_cluster(name=eks_cluster_name)
oidc_issuer = response["cluster"]["identity"]["oidc"]["issuer"]
```

> **Exam tip.** "Grant a pod in EKS access to S3 without storing credentials" →
> **IRSA** (or EKS Pod Identity on newer exams). Not instance profiles — those
> grant every pod on the node the same access.

## Permissions boundaries

A boundary is a **maximum**, not a grant. Effective permissions are the
intersection of the identity policy and the boundary.

```hcl
resource "aws_iam_role" "github_plan" {
  permissions_boundary = local.permissions_boundary_arn
}
```

> **Exam tip.** "Let developers create roles, but ensure those roles can never
> exceed X" → permissions boundary. It is the standard answer to safe
> delegation of IAM.

## Organizations and SCPs

**Service Control Policies** set the maximum permissions for accounts in an OU
or organization. An SCP **never grants** — it only filters.

```
Effective permissions = SCP ∩ identity policy ∩ boundary ∩ session policy
```

> **Exam tip.** "Prevent anyone, including administrators, from disabling
> CloudTrail / leaving a region / using a service" → **SCP**. Root in a member
> account is also restricted by SCPs.

Other Organizations features: consolidated billing, volume discounts shared
across accounts, and delegated administration for services like GuardDuty.

## Federation and Identity Center

| Need | Service |
|------|---------|
| Workforce SSO into AWS accounts | **IAM Identity Center** (formerly AWS SSO) |
| Corporate AD / SAML IdP | Identity Center or SAML federation |
| Mobile/web app end-users | **Cognito** user pools + identity pools |
| CI system, Kubernetes, GitHub | **OIDC federation** via `AssumeRoleWithWebIdentity` |

> **Trap.** Cognito is for **application end users**. Identity Center is for
> **employees**. The exam offers them as alternatives and they are not
> interchangeable.

## Access keys and rotation

Long-lived access keys are the thing the exam wants you to avoid.

| Instead of | Use |
|------------|-----|
| Key in application config | Instance profile / task role / IRSA |
| Key in CI | **OIDC federation** |
| Key for a human | Identity Center with temporary credentials |
| Key that must exist | Rotate it, and audit with Credential Report |

## Useful condition keys

```json
"Condition": {
  "Bool":           { "aws:MultiFactorAuthPresent": "true" },
  "IpAddress":      { "aws:SourceIp": "203.0.113.0/24" },
  "StringEquals":   { "aws:PrincipalOrgID": "o-abc123" },
  "ArnLike":        { "aws:PrincipalArn": "arn:aws:iam::*:role/admin-*" },
  "Bool":           { "aws:SecureTransport": "true" },
  "StringEquals":   { "s3:x-amz-server-side-encryption": "aws:kms" }
}
```

`aws:SecureTransport` is how this workspace's state bucket refuses plain HTTP:

```hcl
statement {
  sid     = "DenyInsecureTransport"
  effect  = "Deny"
  actions = ["s3:*"]
  # ... condition on aws:SecureTransport = false
}
```

> **Exam tip.** "Allow access only from the corporate network" → `aws:SourceIp`.
> "Only from accounts in my organization" → `aws:PrincipalOrgID`.
> "Require MFA" → `aws:MultiFactorAuthPresent`.

## Auditing tools

| Tool | Answers |
|------|---------|
| **IAM Access Analyzer** | Which resources are shared outside my account/org? |
| **Credential Report** | Account-wide CSV of users, keys, MFA, last use |
| **Access Advisor** | Which services has this principal actually used? |
| **CloudTrail** | Who called what, when, from where |
| **Policy Simulator** | Would this request be allowed? |

> **Exam tip.** "Identify unused permissions to right-size a policy" →
> **Access Advisor** (last accessed data). "Find an S3 bucket shared publicly" →
> **Access Analyzer**.

## Exam decision table

| Requirement | Answer |
|-------------|--------|
| App on EC2 needs S3 | Instance profile (role) |
| Pod on EKS needs S3 | IRSA |
| Lambda needs DynamoDB | Execution role |
| CI needs AWS | OIDC federation + role |
| Cross-account access | Role + trust policy, or resource-based policy |
| Cap what delegated admins can grant | Permissions boundary |
| Guardrail across all accounts | SCP |
| Employees sign in with corporate IdP | IAM Identity Center |
| App end users sign in | Cognito |
| Temporary elevated access with MFA | `AssumeRole` + MFA condition |

## Check yourself

1. Recite the policy evaluation order. What always wins?
2. Which policy type has a `Principal` element?
3. Trust policy vs permissions policy — which error points at which?
4. What does a permissions boundary do that an identity policy cannot?
5. Can an SCP grant permissions?
6. IRSA vs instance profile for an EKS pod — why does it matter?
7. In this workspace, `ReadOnlyAccess` is attached yet secret values are
   unreadable. Why?
8. Cognito vs IAM Identity Center?

---

Previous: [01 — Fundamentals](01-fundamentals.md) ·
Next: [03 — VPC networking](03-vpc-networking.md)
