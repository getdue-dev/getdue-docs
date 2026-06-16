# 06 · Security & Compliance

Phase 0 holds **highly sensitive personal financial data** even though it moves no money. The bar is "treat it like a
real bank's data, minus the regulated payment/KYC obligations."

> **This document is the security *design narrative*.** The **enforceable, release-blocking rules** every service
> must satisfy — with conformance levels (MUST/SHOULD), remediation SLAs, and a per-service acceptance gate — live in
> **[09 · Security Standard](./09-security-standard.md)**. Where the two overlap, **doc 09 is authoritative**.

## 1. Threat model (Phase 0, abridged)

| Asset | Threat | Mitigation |
|---|---|---|
| User credentials | Credential stuffing, theft | Argon2id hashing, rate limits, breach-password check, optional MFA |
| Financial data | Cross-tenant leakage | Mandatory `household_id` query filter; authz tests in CI |
| Session tokens | XSS / theft | Short-lived access JWT in memory; refresh token in HTTP-only, `Secure`, `SameSite` cookie |
| Data at rest | DB compromise | Encryption at rest (managed DB), column encryption for sensitive fields, Key Vault keys |
| Data in transit (clients) | MITM | TLS 1.2+, HSTS, cert pinning on mobile |
| **Service-to-service traffic** | Spoofing / lateral movement inside the cluster | **mTLS** (service mesh or cluster network policy), JWT propagated and re-validated at each service, default-deny `NetworkPolicy` |
| **Message broker** | Tampered/forged events | Broker auth (per-service creds), TLS to RabbitMQ, signed event envelopes, idempotent consumers |
| Telemetry | PII/money leakage | Hashed identifiers, no amounts in logs (see [05 §9](./05-monitoring.md#9-data-privacy-in-telemetry)) |
| Supply chain | Compromised dependency | Pinned versions, SBOM, automated dependency updates, signed images |

This platform-level table is **abridged**; a **per-service threat model** (from a shared template/stub) is maintained
for each service and reviewed when its trust boundary changes — see
[engineering/04 SDLC-05](../engineering/04-secure-sdlc.md#2-secure-sdlc--supply-chain).

## 2. Authentication

- **ASP.NET Core Identity** for user store; **Argon2id** (or PBKDF2 at high iterations) for password hashing. Argon2id
  uses the OWASP-minimum parameters **m=19456 KiB (19 MiB), t=2, p=1**, tuned upward as hardware allows.
- **JWT access token** (~15 min) + **rotating refresh token** (HTTP-only cookie, ~30 days, revocable in Redis).
- **Logout & token revocation:** logout **revokes the refresh token immediately** in Redis; the **stateless access JWT
  (≤15 min) stays valid until its `exp`**. For sensitive/critical operations a **Redis `jti` deny-list** gives
  **immediate** access-token revocation (see [09 SEC-AUTH-07](./09-security-standard.md#3-authentication--session-security-per-service)).
- **MFA (TOTP or WebAuthn)** — available on every account; **on by default for ALL users (OWNER and MEMBER)** (release-blocking per [09 SEC-AUTH-04](./09-security-standard.md#3-authentication--session-security-per-service)).
- Email verification required to reach `ACTIVE` status.
- Lockout + exponential backoff on repeated failures.

## 3. Authorization

- **Stateless validation:** the Identity service issues JWTs; **every microservice validates the token locally**
  (signature + claims) with no central session lookup — this is what keeps services stateless and independently
  scalable. The gateway authenticates the edge; each service re-validates defense-in-depth.
- **Tenant isolation:** every request resolves a `householdId` from the principal; each service applies an EF Core
  **global query filter** so no row outside the household is ever read or written — enforced independently in every
  service's own database.
- **Roles (Phase 0):** `OWNER`, `MEMBER` (read or co-edit). Enforced at the handler layer in each service.
- **Architecture test:** a CI test (per service) asserts every entity DbSet has the household filter applied.

## 4. Data protection

| Layer | Control |
|---|---|
| In transit | TLS everywhere; HSTS; mobile cert pinning |
| At rest | Managed-DB transparent encryption; backups encrypted |
| Field-level | Encrypt notably sensitive free-text and identifier RESTRICTED columns (e.g., property address, any future account/government IDs) with a Key Vault-managed key. Monetary RESTRICTED columns rely on DB-level encryption at rest plus telemetry exclusion — field-level encryption is impractical for aggregation columns; see [09 SEC-DATA-03](./09-security-standard.md#7-data-protection--privacy) |
| Keys/secrets | **Azure Key Vault** + managed identity; **zero secrets in code/CI/repo** |
| Backups | Automated, encrypted, tested restore; PITR on Postgres |

## 5. Application security baseline

- Input validation everywhere (FluentValidation + Zod mirror on web); reject unknown fields on writes.
- Output encoding; CSP, X-Content-Type-Options, Referrer-Policy headers on web.
- A **WAF at the edge is mandatory** regardless of provider (Cloudflare / Azure Front Door / CloudFront) — not optional ([09 SEC-NET-03](./09-security-standard.md#5-service-to-service--network-security-zero-trust)).
- Rate limiting (per-user + per-IP) via ASP.NET Core rate limiter, counters in Redis.
- Idempotency keys on creates to prevent duplicate entities.
- Audit log of security-relevant events (login, role change, data export, deletion). **Audit-class logs go to WORM
  storage (object-lock blob/S3) or a SIEM, retained ≥1 year — not Loki** (Loki carries ops logs only); see
  [09 SEC-LOG-03](./09-security-standard.md#9-logging-audit--detection).
- OWASP ASVS L1 as the checklist; OWASP Top 10 reviewed per release.

## 6. Privacy & data rights

Even pre-regulation, design for **GDPR-style** rights from day one:

- **Right to access/export:** `GET /v1/me/export` produces a full JSON dump of the household's data.
- **Right to erasure:** account deletion hard-deletes or anonymizes within a defined window.
- **Data minimization:** collect only what the features need; no SSNs/government IDs in Phase 0 (no KYC).
- **Residency:** pick a single region (e.g., EU) and keep all data + backups there.

## 7. Secure SDLC

The build-and-ship security pipeline — secret scanning, SAST, dependency/SCA review, IaC and container scanning,
SBOM, image signing, DAST — is an **engineering-process** concern and is specified in
**[engineering/04 · Secure SDLC](../engineering/04-secure-sdlc.md)**. The repo workflow itself is plain GitHub Flow on
public repos + a protected `main` ([engineering/01 §7](../engineering/01-repositories.md#7-branch-protection-all-repos)).

At **runtime** the app keeps least-privilege managed identities, a **mandatory WAF at the edge** (regardless of provider),
anomaly alerts, and an audit log (§3–§5, [05 · Monitoring](./05-monitoring.md)).

## 8. Compliance posture (Phase 0)

Because Phase 0 has **no money movement, no card issuance, no KYC/AML, and no real-institution data**, it is **out of
scope for PCI-DSS, PSD2, and most banking licensing** — by design. This is the strategic value of Phase 0: build and
validate the full product surface while the regulatory surface is near-zero.

**What changes in Phase 1+ (flagged now so the architecture is ready):**

| Future capability | Triggers |
|---|---|
| Open Banking / account aggregation | PSD2/Open Banking licensing or regulated aggregator partner |
| Real card issuance | PCI-DSS, BIN sponsor / issuer-processor, KYC/AML |
| Holding/moving real funds | E-money / banking license, AML program, sanctions screening |
| Live market data | Market-data licensing agreements |

The microservices architecture ([01](./01-architecture.md)) means these arrive as **new, independently deployed services**,
not a rewrite — but each is a deliberate, compliance-gated decision, never an accident.
