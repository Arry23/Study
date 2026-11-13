# 12 — Security in design

Security is a design concern, not a review gate. Interviewers expect you to
raise it unprompted.

## Authentication vs authorization

| | **AuthN** | **AuthZ** |
|---|---|---|
| Question | **Who are you?** | **What may you do?** |
| Failure | **401** Unauthorized | **403** Forbidden |
| Mechanism | Password, token, certificate | Roles, policies, ACLs |

> **Trap.** HTTP names these badly — 401 actually means *unauthenticated*.
> Using 401 and 403 correctly is a small but noticeable signal.

## Session vs token

| | **Session (stateful)** | **Token/JWT (stateless)** |
|---|---|---|
| Server stores | Session data | **Nothing** |
| Scale | Needs shared store (Redis) | Scales freely |
| **Revocation** | **Immediate** — delete the session | **Hard** — valid until expiry |
| Size | Small cookie | Larger header |

> **Trade-off — the revocation problem.** A JWT cannot be invalidated before
> it expires. The standard answer: **short-lived access tokens** (5–15 min)
> plus a **long-lived refresh token** that *is* stored and revocable. Say this;
> it is the question interviewers follow up with.

### JWT

```
header.payload.signature
```

```json
{ "sub": "user-123", "exp": 1731350400, "scope": "read:orders", "iss": "auth.example.com" }
```

- **Signed, not encrypted** — anyone can read the payload. Never put secrets
  in it.
- Validate `exp`, `iss`, `aud`, and the signature **algorithm**.

> **Trap — `alg: none`.** A classic vulnerability: the library trusts the
> token's own algorithm claim, and an attacker sets it to `none`. Always pin
> the expected algorithm server-side.

## OAuth 2.0 and OIDC

| | Purpose |
|---|---|
| **OAuth 2.0** | **Authorization** — delegated access to resources |
| **OIDC** | **Authentication** layered on OAuth 2.0 (adds `id_token`) |

**Authorization Code flow with PKCE** — the correct flow for web and mobile:

```
User → App → Authorization Server (login) → code → App
App + code + verifier → Token endpoint → access_token (+ refresh_token)
```

> **Interview tip.** PKCE exists because a mobile app cannot keep a client
> secret. Implicit flow is **deprecated**. Knowing that is a current, concrete
> detail.

**Client Credentials** flow is for machine-to-machine (no user).

## Authorization models

| Model | Basis | Example |
|-------|-------|---------|
| **RBAC** | Roles | `admin`, `editor`, `viewer` |
| **ABAC** | Attributes | `department == resource.department` |
| **ReBAC** | Relationships | "user is owner of document" (Google Zanzibar) |
| **ACL** | Per-object list | File permissions |

> **Interview tip.** RBAC is simple and explodes into role proliferation at
> scale ("editor-but-only-for-EU-invoices"). ABAC/ReBAC handle that. For a
> document-sharing or collaboration product, **ReBAC/Zanzibar** is the modern
> answer.

**Always authorize on the server.** Hiding a button is UX, not security.

> **Trap — IDOR.** `GET /orders/1234` must check that the order belongs to the
> caller. Insecure Direct Object Reference is the most common real-world API
> vulnerability and a great thing to raise in an API design question.

## Encryption

| | In transit | At rest |
|---|---|---|
| Mechanism | **TLS 1.2+** | Disk/volume or field-level encryption |
| Internal traffic | **mTLS** in zero-trust | Managed keys (KMS) |

**Envelope encryption** — a key management service holds a master key that
encrypts **data keys**, which encrypt the data. Lets you rotate the master key
without re-encrypting everything.

**Field-level encryption** for the most sensitive columns (SSN, card data), so
a database dump is not sufficient to leak them.

> **Interview tip.** "Encrypted at rest" alone is weak — if the application
> can read everything, so can an attacker with application access. Field-level
> encryption and tokenisation limit the blast radius.

## Secrets management

| Bad | Good |
|-----|------|
| Hardcoded in source | Secrets manager (Vault, AWS Secrets Manager) |
| In environment variables committed to git | Injected at runtime |
| Shared across environments | Per-environment, least privilege |
| Rotated never | **Automatic rotation** |

> **Trap.** Environment variables are visible in process listings, crash dumps
> and some logging frameworks. They are acceptable, not ideal; mounted files
> or a sidecar-injected secret are better.

## Passwords

```
Never: plaintext, MD5, SHA-1, SHA-256 unsalted
Use:   bcrypt, scrypt, or Argon2id (with a per-user salt)
```

Password hashes must be **slow by design** — that is the entire point. Argon2id
is the current recommendation; bcrypt remains acceptable.

Add: rate limiting on login, account lockout or exponential delay, breached-
password checks, and **MFA**.

## The OWASP risks worth naming

| Risk | Design response |
|------|-----------------|
| **Broken access control** | Authorize every request server-side; default deny |
| **Injection** (SQL, command) | Parameterised queries, never string concatenation |
| **Cryptographic failures** | TLS everywhere, strong hashing, no home-made crypto |
| **SSRF** | Allowlist outbound destinations; block link-local/metadata |
| **Insecure design** | Threat model before building |
| **Security misconfiguration** | Hardened defaults, no debug in production |
| **Vulnerable dependencies** | SCA scanning, patching policy |
| **Identification failures** | MFA, session management, no credential stuffing |

```python
# Injection — the only rule that matters
cur.execute("SELECT * FROM users WHERE email = %s", (email,))   # parameterised
```

## Defence in depth

No single control is sufficient.

```
Edge:        WAF, DDoS protection, rate limiting, TLS
Network:     private subnets, security groups, mTLS, zero trust
Application: authN/authZ, input validation, output encoding
Data:        encryption at rest, field-level, least-privilege DB users
Operations:  audit logs, anomaly detection, secret rotation
```

**Zero trust** — never trust the network. Every call is authenticated and
authorized even inside the perimeter. mTLS plus service identity is how.

## Multi-tenancy

| Isolation | Model | Blast radius |
|-----------|-------|--------------|
| Shared everything | `tenant_id` column | **Largest** — one bug leaks across tenants |
| Schema per tenant | Separate schemas | Medium |
| Database per tenant | Separate DBs | Small |
| Infrastructure per tenant | Separate deployments | Smallest, most expensive |

> **Trap.** With a shared table, a **single missing `WHERE tenant_id = ?`**
> leaks every customer's data. Enforce it structurally — row-level security,
> a repository layer that cannot issue unscoped queries, or query review —
> not by developer discipline.

## Threat modelling — STRIDE

| Threat | Mitigation |
|--------|------------|
| **S**poofing | Strong authentication |
| **T**ampering | Signatures, integrity checks, TLS |
| **R**epudiation | Audit logs |
| **I**nformation disclosure | Encryption, access control |
| **D**enial of service | Rate limiting, autoscaling, WAF |
| **E**levation of privilege | Least privilege, authorization checks |

> **Interview tip.** Walking a design through STRIDE for 60 seconds is a
> structured, senior-sounding way to cover security without rambling.

## Privacy and compliance

| Requirement | Design implication |
|-------------|--------------------|
| **GDPR right to erasure** | You must be able to **delete** a user — including from backups, caches, logs and analytics |
| Data residency | Region-pinned storage and processing |
| PII minimisation | Do not collect it; tokenise what you must |
| Audit | Immutable, tamper-evident logs |

> **Interview tip.** "Delete my data" is genuinely hard once data is in event
> logs, backups and a warehouse. **Crypto-shredding** — encrypt per-user and
> destroy the key — is the practical answer. Raising this shows real-world
> awareness.

## Decision table

| Requirement | Answer |
|-------------|--------|
| Stateless auth across services | **JWT**, short TTL + refresh token |
| Must revoke access immediately | Server-side sessions, or a token denylist |
| Third-party app access to user data | **OAuth 2.0 authorization code + PKCE** |
| "Who can view this document?" | **ReBAC** (Zanzibar-style) |
| Protect passwords | **Argon2id** / bcrypt + MFA |
| Prevent SQL injection | **Parameterised queries** |
| Service-to-service identity | **mTLS** + service mesh |
| Rotate database credentials | **Secrets manager** with rotation |
| Tenant data isolation | Row-level security or DB-per-tenant |
| Honour a deletion request | **Crypto-shredding** + deletion pipeline |
| Protect the metadata endpoint | Block link-local; enforce IMDSv2-style session auth |

## Practice

1. 401 vs 403 — define each precisely.
2. Why can't a JWT be revoked, and what is the standard mitigation?
3. What is PKCE for, and which flow does it replace?
4. A user changes `GET /invoices/501` to `/invoices/502` and sees someone
   else's invoice. Name the vulnerability and the fix.
5. Why must password hashing be slow?
6. One missing clause leaks all tenants' rows. How do you prevent this
   structurally?
7. A GDPR deletion request arrives. List the places data may hide.
8. Walk a file-sharing service through STRIDE.

<details><summary>Answers</summary>

1. **401** — not authenticated (or credentials invalid); the client should
   authenticate. **403** — authenticated but not permitted; authenticating
   again will not help.
2. It is self-contained and validated by signature alone, so the server has no
   record to delete. Mitigate with **short expiry** (5–15 min) plus a
   revocable **refresh token**, and optionally a denylist for emergencies.
3. **PKCE** binds the authorization code to the client that requested it, so a
   stolen code is useless. It replaces the **implicit flow** and is required
   for mobile/SPA clients that cannot hold a secret.
4. **IDOR** (broken access control). Fix by authorizing on the server —
   verify ownership on every request. Opaque or random IDs help but are not a
   substitute.
5. To make brute-forcing a leaked hash dump expensive. A fast hash lets an
   attacker try billions per second; Argon2id/bcrypt with a tuned work factor
   reduces that to thousands.
6. Database **row-level security** scoped to the connection's tenant, a
   repository layer that physically cannot build an unscoped query, separate
   schemas/databases, and automated tests that assert cross-tenant access
   fails.
7. Primary database, read replicas, backups and snapshots, caches, search
   index, event log/Kafka, data warehouse, application logs, analytics tools,
   CDN, and third-party processors. **Crypto-shredding** handles the ones you
   cannot selectively delete.
8. **S** — account takeover → MFA, strong sessions. **T** — file modified in
   transit/at rest → TLS, checksums, versioning. **R** — user denies deleting a
   file → immutable audit log. **I** — public link leaks → signed URLs with
   expiry, authorization checks. **D** — upload flood → rate limits, quotas,
   size caps. **E** — viewer becomes owner → server-side permission checks,
   least privilege.
</details>

---

Previous: [11 — Observability and SLOs](11-observability-and-slos.md) ·
Next: [13 — Architecture styles](13-architecture-styles.md)
