# 11 · Versioning System (Cloud-Wide)

A microservices platform has **many things that version independently** — APIs, services, contracts, events,
databases, infra. This document is the **single authoritative versioning standard** for all of GetDue. It unifies the
rules already referenced in [04 · API Design](./04-api-design.md), [08 · Repositories & Contracts](./08-repositories.md),
and [09 · Security Standard](./09-security-standard.md).

## 0. Principles

1. **SemVer everywhere** (`MAJOR.MINOR.PATCH`) — MAJOR = breaking, MINOR = additive/back-compat, PATCH = fix.
2. **Backward compatibility is the default**; breaking changes are deliberate, announced, and versioned.
3. **Immutable artifacts** — a built image / published package / git tag is never re-pushed; a change = a new version.
4. **Independent cadence** — each service versions and ships on its own clock; the contract is the coupling point, not a shared release train.
5. **Tolerant reader** — consumers ignore unknown fields, so additive producer changes never break them.
6. **Traceability** — every running thing can report exactly which version it is.

## 1. The layers that version

| Layer | Versioning scheme | Lives in | Authority |
|---|---|---|---|
| **Public API** | URL major `/v1` + deprecation headers | gateway + each service | this doc §2 |
| **Service / release** | SemVer git tag → container image tag | each service repo | §3 |
| **Contracts (DTOs/OpenAPI)** | SemVer package | `getdue-contracts` | §4 + [08 §3](./08-repositories.md#3-the-contracts-repo-getdue-contracts) |
| **Integration events** | `schemaVersion` + SemVer package | `getdue-contracts` | §5 |
| **Shared library** | SemVer NuGet | `getdue-buildingblocks` | §6 |
| **Database schema** | ordered migrations (expand/contract) | each service repo | §7 |
| **Infra / config** | chart + module versions, GitOps revision | `getdue-platform` / `getdue-deploy` | §8 |

## 2. API versioning

- **Major version in the URL path:** `https://api.getdue.com/v1/...`. A new MAJOR (`/v2`) is introduced **only** for a
  breaking change; `v1` and `v2` run **side by side** during the deprecation window.
- **Additive (MINOR) changes do not bump the URL** — new optional fields, new endpoints, new enum values are
  back-compatible and ship within `/v1` (tolerant-reader clients absorb them).
- **Breaking changes** (removing/renaming a field, tightening validation, changing a type or semantics) require a new
  MAJOR.
- **Deprecation protocol** (RFC 8594): a deprecated endpoint/version returns
  `Deprecation: true`, `Sunset: <date>`, and a `Link: <docs>; rel="deprecation"` header; the date is also published in
  the changelog. **Minimum support window: 6 months** (or until all first-party clients migrate, whichever is longer).
- The **gateway** routes by major version and emits per-version usage metrics so you can see who still uses `/v1`
  before sunsetting it.

```
GET /v1/networth
200 OK
Deprecation: true
Sunset: 2027-01-01T00:00:00Z
Link: <https://docs.getdue.com/api/v2/networth>; rel="deprecation"
```

## 3. Service / release versioning

Each service repo is versioned independently with **SemVer**, materialized as an immutable container image:

```
git tag           v2.4.1
container image   registry/getdue-accounts:2.4.1
                  registry/getdue-accounts:2.4.1+sha.<gitsha>   # build metadata
```

- **Tag = source of truth.** CI builds, tests, scans, **signs (cosign)**, and attaches an **SBOM + SLSA provenance**
  to the image ([09 §8](./09-security-standard.md#8-secure-sdlc--supply-chain)). The same immutable image is promoted
  staging → production — **never rebuilt per environment**.
- **No `latest` in production.** Deployments pin an exact version; admission control rejects unsigned/untagged images.
- A service's **API major** and its **release version** are decoupled: `accounts` can go `2.x → 3.x` internally while
  still serving API `/v1`.
- **Rollback = redeploy the previous image tag** (GitOps revert of the tag in `getdue-deploy`). Because images and DB
  migrations are forward/back-compatible within a window (§7), rollback is safe.

## 4. Contract (DTO / OpenAPI) versioning

The `getdue-contracts` package is **SemVer**, and is the coupling point between services and clients
([08 §3](./08-repositories.md#3-the-contracts-repo-getdue-contracts)):

- **MINOR** = additive (new optional field/endpoint/enum) → consumers upgrade at will, nothing breaks.
- **MAJOR** = breaking → requires an ADR, a new API major (§2), and a migration window.
- **CI gate:** a **schema-diff check** fails the build if a change is breaking but the version bump isn't MAJOR — so
  the compatibility promise is mechanically enforced, not trusted.
- Services depend on a **pinned, published** contracts version — never a branch or local path.

## 5. Event (message) versioning

Integration events on RabbitMQ are versioned so producers and consumers evolve independently:

- Every event envelope carries an explicit **`schemaVersion`** and an **event id** (for idempotent consumers).
- **Additive evolution only within a major:** add optional fields; **never repurpose or remove** a field in place.
- **Tolerant reader:** consumers ignore unknown fields and default missing optional ones.
- A **breaking** event change publishes a **new event type/version** (e.g., `PropertyValued.v2`); producers emit both
  during the transition, consumers migrate, then `v1` is retired.
- Event contracts live in `getdue-contracts` and version with it (§4), so the schema-diff gate covers them too.

## 6. Shared library versioning

`getdue-buildingblocks` (Money, outbox, idempotency, OTel, auth) is **SemVer NuGet**. A breaking change is a MAJOR
bump; **Dependabot** opens upgrade PRs across all consuming repos, and each service's CI re-runs contract + arch tests
before merge ([08 §5](./08-repositories.md#5-versioning--dependency-flow)). Services pin exact versions — no floating ranges.

## 7. Database schema versioning (zero-downtime)

Each service owns its DB and its **ordered EF Core migrations** (timestamped, committed, CI-gated). Because services
run **2–3 pods** with **rolling deploys**, old and new code run **simultaneously** during a rollout — so every schema
change is **backward-compatible across one release** via **expand → migrate → contract**:

| Phase | Action | Compatible with |
|---|---|---|
| **Expand** | add the new column/table (nullable/defaulted); deploy code that writes both old + new | old readers |
| **Migrate** | backfill data; switch reads to the new shape | both shapes present |
| **Contract** | a *later* release drops the old column once no code uses it | new readers only |

- **Never** combine a destructive change with the code that needs it in the same release — that breaks rolling deploys
  and rollback.
- Migrations run as a **gated pipeline step** before the new image receives traffic; readiness probes hold traffic
  until the DB is at the expected version.
- Forward-only in production; "rollback" is a new compensating migration, not a down-migration against live data.

## 8. Infrastructure & configuration versioning

- **Helm charts** (`getdue-platform`) are SemVer; **Terraform modules** are tagged releases — infra changes are
  reviewed via `plan` and versioned like code.
- **GitOps (`getdue-deploy`)** is the **declarative version of the running system**: the desired image tag + chart
  version per service is committed; Argo CD reconciles to it. The git history of this repo **is** the deployment
  history — every prod state is a commit you can diff and revert.
- Config/secrets are versioned references (Key Vault secret versions), never inline literals.

## 9. Compatibility & support matrix

| Pair | Rule |
|---|---|
| Client ↔ API | A client built against `/v1` keeps working until `/v1` sunset (≥6 months after `/v2` GA) |
| Service ↔ contracts | A service runs against any **same-MAJOR** contracts version |
| Producer ↔ consumer (events) | Any consumer handles all **same-major** event versions; new majors run dual-emit during transition |
| Code ↔ DB | New code works against the **current and previous** schema (expand/contract, §7) |

## 10. Traceability — "what version is running?"

Every service exposes and emits its version so any environment is auditable:

- **`GET /health`** (and a `/version` probe) returns `{ service, version, gitSha, builtAt, apiVersions: ["v1"] }`.
- **OpenTelemetry resource attributes** stamp `service.version` on every span/metric/log → dashboards and traces are
  filterable by version, so you can compare error rates **across a rollout** ([05 · Monitoring](./05-monitoring.md)).
- The **gateway** aggregates each service's version into a platform **release manifest** (also the GitOps commit, §8).

## 11. Change-log & release process

- Each repo keeps a **`CHANGELOG.md`** (Keep-a-Changelog style); releases are cut from SemVer git tags.
- **Breaking changes require an ADR** (`docs/adr/`) and a deprecation entry with a sunset date.
- Releases flow **one immutable artifact** through environments: `build → staging (auto) → production (manual approval,
  [09 SEC-GOV-05](./09-security-standard.md#1-governance-ownership--change-control))` — the version that passed staging
  is the exact version promoted to prod.

## 12. ADR additions

| ID | Decision |
|---|---|
| ADR-008 | **SemVer across all layers**; URL-path major for the public API with a ≥6-month deprecation/sunset window |
| ADR-009 | **Immutable, signed artifacts** promoted across environments (never rebuilt per env); GitOps repo = deployment history |
| ADR-010 | **Expand/contract DB migrations** to guarantee zero-downtime rolling deploys and safe rollback |
