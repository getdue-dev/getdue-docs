# 09 · Security Standard — Bank-Grade Rules for All Services

> **Status:** Mandatory baseline. **Every** GetDue service, gateway, client, and shared repo MUST comply before it
> ships to production. This is the authoritative control catalogue; [06 · Security](./06-security.md) is the design
> narrative, this document is the **enforceable ruleset**.
>
> **Conformance language (RFC 2119):** **MUST** = hard requirement, release-blocking. **SHOULD** = strongly
> expected; deviations require a documented, time-boxed risk acceptance signed by the security owner. **MAY** = optional.

## 0. Guiding principles

1. **Zero trust** — no implicit trust by network location; authenticate and authorize every call, including
   service-to-service.
2. **Least privilege** — every identity (human, service, pipeline) gets the minimum access, time-boxed.
3. **Defense in depth** — gateway auth does not excuse a service from re-validating.
4. **Secure by default** — deny-by-default network, encryption on by default, secrets never in code.
5. **Separation of duties** — no single person can author, approve, and deploy to production unreviewed.
6. **Assume breach** — log everything security-relevant, immutably; design for fast detection and containment.
7. **Money never moves** (Phase 0) — and the controls that will guard money movement are built now.

---

## 1. Governance, ownership & change control

- **SEC-GOV-01 (MUST)** Every repo has a `CODEOWNERS` and a `SECURITY.md` referencing this standard.
- **SEC-GOV-02 (MUST)** Default branch protected: PR-only, **1 review from CODEOWNERS**, all CI gates green,
  **no merge on red**, no force-push, linear history.
- **SEC-GOV-03 (MUST)** **Signed commits** (GPG/Sigstore) required on protected branches.
- **SEC-GOV-04 (MUST)** **Separation of duties:** the person who approves a production deploy MUST NOT be its sole author.
- **SEC-GOV-05 (MUST)** Production deploys require **manual approval** in a protected GitHub `production` environment;
  staging may auto-deploy.
- **SEC-GOV-06 (MUST)** Every production change is traceable to a PR, an approver, and a ticket (auditable change record).
- **SEC-GOV-07 (SHOULD)** Quarterly access review of all repo, cloud, and database grants.

## 2. Identity & access management (IAM)

- **SEC-IAM-01 (MUST)** **MFA** for all human access to GitHub, cloud, CI, and production data stores.
- **SEC-IAM-02 (MUST)** **No long-lived cloud credentials in CI** — pipelines authenticate via **OIDC/workload
  identity federation** to short-lived tokens.
- **SEC-IAM-03 (MUST)** Each service runs under its **own workload identity / K8s ServiceAccount**; no shared identities.
- **SEC-IAM-04 (MUST)** Database credentials are **per-service**, least-privilege (a service can reach only its own DB),
  and rotated automatically.
- **SEC-IAM-05 (MUST)** **No shared human logins**; no standing admin. Privileged access is **just-in-time** and time-boxed.
- **SEC-IAM-06 (MUST)** Offboarding revokes all access within 24h; automated where possible.

## 3. Authentication & session security (per service)

- **SEC-AUTH-01 (MUST)** End-user auth issues **short-lived JWT access tokens (≤15 min)** + **rotating refresh tokens**;
  refresh tokens are revocable and stored server-side (Redis), never in `localStorage`.
- **SEC-AUTH-02 (MUST)** **Every service validates the JWT locally** (signature, `iss`, `aud`, `exp`, scopes) — the
  gateway's check does not exempt it (defense in depth, zero trust).
- **SEC-AUTH-03 (MUST)** Passwords hashed with **Argon2id** (or PBKDF2 ≥ OWASP iteration guidance); breached-password
  check at set-time; account lockout + exponential backoff.
- **SEC-AUTH-04 (MUST)** **MFA (TOTP/WebAuthn)** available to all users; **on by default for account owners**.
- **SEC-AUTH-05 (MUST)** JWT signing keys are asymmetric (RS/ES), stored in the vault/HSM, and **rotated** with a
  published JWKS; tokens are validated against current keys.
- **SEC-AUTH-06 (MUST)** Mobile stores refresh token in the **Keychain**; web uses **HttpOnly, Secure, SameSite** cookies.

## 4. Authorization & tenant isolation

- **SEC-AUTHZ-01 (MUST)** Authorization checked at the **service/handler layer**, not only the gateway.
- **SEC-AUTHZ-02 (MUST)** **Tenant isolation:** every data access is scoped by `householdId` via an EF Core **global
  query filter**; a CI architecture test asserts the filter on every entity. Cross-household access is impossible by
  construction.
- **SEC-AUTHZ-03 (MUST)** Roles enforced server-side (`OWNER`, `MEMBER`); never trust client-supplied role/identity.
- **SEC-AUTHZ-04 (MUST)** Object-level checks on every `{id}` route (no IDOR): the resource MUST belong to the caller's household.

## 5. Service-to-service & network security (zero trust)

- **SEC-NET-01 (MUST)** **mTLS** for all service-to-service traffic (service mesh or equivalent); plaintext intra-cluster
  traffic is prohibited.
- **SEC-NET-02 (MUST)** **Default-deny `NetworkPolicy`**; a service may reach only the dependencies it declares
  (its DB, Redis, broker, named peers).
- **SEC-NET-03 (MUST)** Public ingress only through **Cloudflare/edge → ingress-nginx → gateway**, behind a **WAF +
  TLS 1.2+ (prefer 1.3)**; HSTS enabled; no service is directly internet-exposed. TLS terminates at the ingress and,
  if a CDN/edge is used, the **edge→origin hop MUST also be encrypted with origin-cert validation** (Cloudflare
  **Full (strict)**, never "Flexible"). The origin LB MUST be **locked to the edge** (allow-list edge IP ranges or
  authenticated origin pull) so the WAF cannot be bypassed. See [01 §7.1](./01-architecture.md#71-edge-ingress--https).
- **SEC-NET-04 (MUST)** **Per-user + per-IP rate limiting** and request-size caps at the gateway/ingress;
  abuse/anomaly alerting. When fronted by a CDN/edge, the **real client IP** MUST be propagated (e.g. `CF-Connecting-IP`)
  so rate limits and audit logs key on the true client, not the edge.
- **SEC-NET-05 (MUST)** **Broker security:** RabbitMQ requires TLS + per-service credentials; events carry signed
  envelopes; consumers are **idempotent** (dedup by event id) and validate `schemaVersion`.
- **SEC-NET-06 (SHOULD)** Egress is default-deny; the Phase 0 guardrail (no outbound financial-institution calls) is
  enforced both by network policy and by architecture tests.

## 6. Secrets & key management

- **SEC-SEC-01 (MUST)** **No secrets in source, history, images, or logs.** Secret scanning + **push protection** ON
  on every repo; a leaked secret triggers immediate rotation.
- **SEC-SEC-02 (MUST)** Secrets live in a **managed vault** (Azure Key Vault / equivalent); in-cluster via Sealed
  Secrets/CSI driver, never plain K8s `Secret` in git.
- **SEC-SEC-03 (MUST)** Encryption keys are vault/**HSM-backed**; key rotation policy defined; access audited.
- **SEC-SEC-04 (MUST)** All secrets/keys have an owner and a **rotation schedule**; credentials auto-rotated where supported.

## 7. Data protection & privacy

- **SEC-DATA-01 (MUST)** **Encryption in transit** (TLS everywhere, mTLS internally) and **at rest** (DB, backups,
  object storage, queues).
- **SEC-DATA-02 (MUST)** **Data classification** applied: `RESTRICTED` (financial values, auth secrets), `CONFIDENTIAL`
  (PII), `INTERNAL`, `PUBLIC`. Controls scale with class.
- **SEC-DATA-03 (MUST)** **Field-level encryption** for the most sensitive identifier and free-text RESTRICTED columns
  (e.g., property address, any future account/government identifiers) with vault-managed keys. Monetary RESTRICTED
  columns (balances, principals, FX rates) rely on DB-level encryption at rest (SEC-DATA-01) plus strict access
  scoping and telemetry exclusion (SEC-DATA-05) — field-level encryption is impractical for columns participating in
  net-worth aggregation.
- **SEC-DATA-04 (MUST)** **Money as `decimal`/`numeric(19,4)`** — never float — to prevent rounding/integrity errors.
- **SEC-DATA-05 (MUST)** **No `RESTRICTED`/`CONFIDENTIAL` data in logs, metrics, traces, or error messages**; identifiers
  are hashed; **no monetary amounts in telemetry** (see [05 §9](./05-monitoring.md#9-data-privacy-in-telemetry)).
- **SEC-DATA-06 (MUST)** **Encrypted, tested backups**; PITR on Postgres; documented restore drills; defined retention.
- **SEC-DATA-07 (MUST)** **GDPR rights** built in: data export (`/me/export`) and erasure with a defined SLA; data
  residency pinned to one region.
- **SEC-DATA-08 (SHOULD)** Non-production environments use **synthetic/anonymized** data only; production data MUST NOT
  be copied to lower environments.

## 8. Secure SDLC & supply chain

Every service pipeline MUST run these gates, and **MUST fail the build** on threshold breach:

| Gate | Tool (example) | Blocking threshold |
|---|---|---|
| **SAST** | CodeQL | any High/Critical |
| **Secret scanning** | GitHub + gitleaks | any verified secret |
| **SCA (dependencies)** | Dependabot / OWASP DC | Critical, or High past SLA |
| **IaC scanning** | tfsec / Checkov | any High misconfig |
| **Container scanning** | Trivy/Grype | Critical OS/lib CVE |
| **License compliance** | SCA | disallowed license |
| **Architecture tests** | NetArchTest | any violation (incl. money-movement / outbound-finance guard) |
| **Code coverage** | Coverlet / Vitest / xccov | **< 100% line OR branch** ([12 · Testing](./12-testing-standard.md)) |
| **Mutation testing** | Stryker | mutation score < 85% (no surviving mutants in money/authz/tenant code) |

- **SEC-SDLC-01 (MUST)** Container images are **minimal** (distroless/chiseled), run as **non-root**, read-only
  filesystem, no shell where avoidable.
- **SEC-SDLC-02 (MUST)** Images are **signed (cosign)** and ship an **SBOM**; deploy admission **rejects unsigned images**.
- **SEC-SDLC-03 (MUST)** **Provenance attestation (SLSA)** for build artifacts; dependencies **pinned** (no floating ranges).
- **SEC-SDLC-04 (MUST)** **DAST** against staging before promotion to production for any externally-reachable surface.
- **SEC-SDLC-05 (SHOULD)** **Threat model** maintained per service; reviewed when the trust boundary changes.

## 9. Logging, audit & detection

- **SEC-LOG-01 (MUST)** **Immutable, append-only audit log** of security events: authn (success/failure), authz denials,
  role/permission changes, data export, deletion, admin/privileged actions, secret access.
- **SEC-LOG-02 (MUST)** Logs are **structured**, carry `traceId` + **hashed** subject ids, and **exclude** secrets,
  tokens, PII, and amounts.
- **SEC-LOG-03 (MUST)** Centralized aggregation (Loki/SIEM) with **tamper-evident retention** ≥ 1 year for audit events.
- **SEC-LOG-04 (MUST)** **Alerting** on: auth-failure spikes, privilege changes, anomalous access patterns, WAF blocks,
  a service dropping below 2 healthy pods, broker/consumer lag (see [05 §6](./05-monitoring.md#6-alerting-alertmanager--grafana-alerts)).
- **SEC-LOG-05 (SHOULD)** Time synchronization (NTP) across all nodes for forensically reliable timestamps.

## 10. Vulnerability & patch management (remediation SLAs)

| Severity (CVSS) | Remediate within | Action if exceeded |
|---|---|---|
| **Critical (9.0–10)** | **24–48h** | Block deploys; emergency patch/mitigation |
| **High (7.0–8.9)** | **7 days** | Build warns; escalates to blocking at SLA |
| **Medium (4.0–6.9)** | **30 days** | Tracked; reviewed in sprint |
| **Low (< 4.0)** | **90 days** | Backlog |

- **SEC-VULN-01 (MUST)** Base images and dependencies patched continuously; nightly rescan of running images.
- **SEC-VULN-02 (SHOULD)** Annual third-party **penetration test**; findings tracked to SLA.
- **SEC-VULN-03 (SHOULD)** A **coordinated disclosure** path (`security@`) and `SECURITY.md` in every repo.

## 11. Resilience & abuse prevention

- **SEC-RES-01 (MUST)** **2–3 stateless pods per service**, liveness/readiness probes, `PodDisruptionBudget` (minAvailable 1),
  rolling zero-downtime deploys.
- **SEC-RES-02 (MUST)** Resilience policies (Polly: timeout, retry-with-jitter, circuit breaker) on all sync inter-service calls.
- **SEC-RES-03 (MUST)** **Idempotency keys required** on all creating/state-changing endpoints, enforced via
  shared storage (so retries across the 2–3 pods are safe) with exactly-once side effects and a **7d key TTL**
  (mobile background retries routinely exceed 24h); **event consumers dedupe by event id**. Full mechanism:
  [04 · API Design §5](./04-api-design.md#5-idempotency-keys).
- **SEC-RES-04 (SHOULD)** Bot/abuse protection at the edge; graceful degradation under load.

## 12. Incident response

- **SEC-IR-01 (MUST)** Documented IR plan with severity levels, on-call, and escalation.
- **SEC-IR-02 (MUST)** Each alert links to a **runbook** (`docs/runbooks/`) with diagnosis + containment steps.
- **SEC-IR-03 (MUST)** Defined **breach-notification** procedure aligned to GDPR (72h regulator window) and user comms.
- **SEC-IR-04 (SHOULD)** Blameless post-incident review with tracked corrective actions; periodic tabletop exercises.

---

## 13. Per-service security acceptance gate (release checklist)

A service **MUST NOT** reach production until **all** are true:

- [ ] Branch protection, CODEOWNERS, signed commits, 1-CODEOWNER-review rule active (§1)
- [ ] Own workload identity + own least-privilege DB credentials (§2, §6)
- [ ] Local JWT validation + tenant `householdId` filter with passing arch test (§3, §4)
- [ ] mTLS + default-deny NetworkPolicy + declared dependencies only (§5)
- [ ] No secrets in repo/image; vault-sourced config; push protection on (§6)
- [ ] Encryption in transit + at rest; no RESTRICTED data in telemetry (§7)
- [ ] All SDLC gates green; signed image + SBOM; non-root, read-only fs (§8)
- [ ] **100% line + branch coverage** + mutation score ≥ 85% (no surviving mutants in money/authz/tenant code) — [12 · Testing](./12-testing-standard.md)
- [ ] Audit logging of security events wired to SIEM with alerting (§9)
- [ ] No open Critical/High vulns past SLA (§10)
- [ ] 2–3 pods, probes, PDB, resilience policies (§11)
- [ ] Runbooks + IR/breach procedure linked (§12)
- [ ] Architecture tests pass: **no money movement, no outbound financial-institution calls** (Phase 0 guardrail)

## 14. Compliance mapping (build now, certify later)

Phase 0 holds no real money/cards/KYC, so it is **out of scope for PCI-DSS/PSD2 licensing today** — but the controls
above are deliberately chosen to map to the frameworks that apply once Phase 1 connects real systems:

| Framework | What these rules pre-satisfy |
|---|---|
| **PCI-DSS** (future card issuance) | Network segmentation, encryption, least privilege, logging, vuln mgmt, change control |
| **SOC 2 (Security/Availability/Confidentiality)** | Access control, change mgmt, monitoring, IR, separation of duties |
| **ISO 27001** | ISMS controls: asset classification, access, crypto, ops security, supplier/supply chain |
| **GDPR** | Data classification, encryption, minimization, subject rights, breach notification, residency |
| **NIST CSF / SSDF** | Identify-Protect-Detect-Respond-Recover; secure SDLC + supply-chain integrity |

> **Principle:** every control that would be *expensive to retrofit* into a regulated platform is established in
> Phase 0 while the regulatory surface is near-zero. Phase 1 then turns certification into evidence-gathering, not redesign.
