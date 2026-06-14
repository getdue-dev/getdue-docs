# 07 · Roadmap & Phasing

## 1. Phase 0 milestones

| # | Milestone | Outcome | Exit criteria |
|---|---|---|---|
| **M0** | Walking skeleton | Repo, CI/CD, Docker Compose, deploy pipeline, OTel wired | Empty API deploys to staging; "hello" traced end-to-end |
| **M1** | Identity | Register, verify, login, household, MFA | A user can log into web + a stub mobile screen |
| **M2** | Core entities | Bank accounts, loans, real estate, mortgages, stocks CRUD | All six entities persist with validation + snapshots |
| **M2.5** | Loan schedules & payments | Import/generate amortization schedule; record SCHEDULED / PARTIAL_PREPAYMENT / FULL_PAYOFF / EXTRA payments with re-amortization | Schedule imports atomically; partial prepayment re-amortizes (new version); full payoff zeroes balance → `PAID_OFF` and updates net worth |
| **M3** | Net worth | Multi-currency aggregation + FX rates + history + breakdown | Net-worth correct across ≥2 currencies; per-currency breakdown + `unconverted` handling; trend chart matches fixtures |
| **M4** | Goals | Create/fund/track goals, projections | A goal shows progress %, on/off-track, required monthly |
| **M4.5** | Client dashboard & analytics | `/dashboard` read model + widgets (net worth, allocation, debt, goals, currency exposure, KPIs) | Single-call dashboard renders on web + mobile; values match net-worth fixtures; `unconverted` surfaced |
| **M5** | Monitoring | Ops dashboards + financial-health insights + alerts | Dashboards live; an alert fires; insights endpoint returns signals |
| **M6** | iPhone app | Thin client for view + key edits | App Store TestFlight build hitting staging API |
| **M7** | Hardening | Security review, load test, runbooks, backup/restore drill | OWASP ASVS L1 pass; **100% coverage + mutation gates green**; restore drill green; SLOs met |

Suggested sequence: **M0 → M1 → M2 → M3 → M4 → M5** in parallel-friendly order, with **M6 (mobile)** starting after
M3 (once the contract is stable) and **M7** as the closing gate.

## 2. Definition of Done for Phase 0

- [ ] All six entity types fully CRUD-able via API, web, and mobile (view at minimum on mobile).
- [ ] Net worth correct and reconstructable for any historical date, **across multiple currencies** (entities in native currency, rolled up to the household base + any display currency).
- [ ] Loans/mortgages support **schedule import/generate** and **payments (regular, partial prepayment, full payoff)** with re-amortization; balances and net worth update correctly.
- [ ] At least one financial goal type works end-to-end with projections.
- [ ] **Client dashboard** live on web + mobile: net worth, allocation, debt, goals, currency exposure, KPIs from a single `/dashboard` read model ([10](./10-dashboard-analytics.md)).
- [ ] Operational dashboards + financial-health insights live; alerting verified.
- [ ] **Every service passes the [§13 security acceptance gate](./09-security-standard.md#13-per-service-security-acceptance-gate-release-checklist)** — no service ships to prod otherwise.
- [ ] Multi-repo governance live: per-service repos + shared `contracts`/`buildingblocks`, org branch-protection rulesets ([08](./08-repositories.md)).
- [ ] CI runs unit + integration + **architecture/guardrail** tests + full security gate (SAST/SCA/secret/IaC/container scan, image signing) on every PR.
- [ ] **100% line + branch coverage** + mutation testing enforced as a blocking CI gate on every repo ([12 · Testing](./12-testing-standard.md)).
- [ ] Backup + restore drill executed and documented.
- [ ] OpenAPI published; both clients generate types from it.

## 3. Explicitly deferred to Phase 1+

| Capability | Phase | Note |
|---|---|---|
| Live market data for stocks | 1 | Replaces manual prices; needs data-license |
| Open Banking / account aggregation | 1–2 | Regulated; separate service subscribing to events |
| Real card issuance | 2+ | PCI-DSS, issuer-processor, KYC/AML |
| Real payments / transfers | 2+ | E-money/banking license |
| **Live FX rate feed** | 1 | Phase 0 already supports multi-currency with **user-sourced** rates; Phase 1 adds an automatic rate provider |
| Multi-household membership | 1 | A user in several households |
| Budgeting / cash-flow forecasting | 1 | Builds on the snapshot history |
| Advisory / recommendations | 2 | "What-if" planning on top of goals |

## 4. How Phase 0 makes Phase 1 cheap

- **Microservices** → each integration arrives as a *new service* (e.g. `market-data`, `open-banking`) subscribing to the same events, deployed independently — no change to existing services.
- **Valuation snapshots** → live pricing and aggregation feeds just become a *new source* (`source = OPEN_BANKING`,
  `source = MARKET_FEED`) writing the same snapshot table.
- **Contract-first API** → clients absorb new fields without breaking.
- **Observability + guardrails baked in** → new services inherit tracing, dashboards, and architecture tests for free.

## 5. Risks & mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Scope creep into real integrations | Pulls in regulation early | Architecture tests block outbound finance calls in Phase 0 |
| Money math errors | Wrong net worth | `decimal`/`numeric(19,4)` only; property-based tests on aggregation |
| Swift skills ramp-up (new language) | Slows M6 | Thin-client scope (UI only); contract-driven Swift client; start after M3 |
| Telemetry leaks PII/amounts | Privacy incident | Hashed IDs, no amounts in telemetry, reviewed in M7 |
| Single-region outage | Downtime | Phase 0 accepts it (99.5% SLO); multi-region is a later decision |
| Distributed-system complexity (broker, per-service DB, eventual consistency) | Slower start, harder debugging | Shared `BuildingBlocks` (outbox, events, OTel); idempotent consumers; distributed tracing from day one |

## 6. First two weeks (concrete starting point)

1. Create the **shared repos first** ([08](./08-repositories.md)): `getdue-contracts`, `getdue-buildingblocks`, `getdue-platform`, `getdue-deploy`, and the org `.github` (reusable workflows + security rulesets per [09](./09-security-standard.md)).
2. Apply org-wide **branch protection & security rulesets** (signed commits, PR-from-feature-branch, required CI gates, push protection) so every repo is governed from day one. The `CODEOWNERS` approval and two-person prod-deploy rules are checked in as deferred controls per [09 §0.1](./09-security-standard.md#01-solo-phase-scope), activating when a second engineer joins.
3. Stand up the `getdue-platform` Docker Compose mesh: Postgres, Redis, **RabbitMQ**, OTel Collector, Grafana stack.
4. Build **one reference service repo** (`getdue-identity`): 4-project shape, own DB, JWT validation, tenant filter, health checks, first trace — the template every other service repo copies.
5. Add the gateway repo (`getdue-gateway`) + each service's `deploy/k8s` (Deployment replicas: 2, HPA max 3, probes, PDB) on **Kubernetes (AKS)**; verify a rolling, zero-downtime deploy.
6. Wire the **reusable** GitHub Actions pipeline (build → test → archtest → SAST/secret/SCA/IaC/container scan → SBOM + sign → push image → GitOps deploy), consumed by every service repo.
7. Prove the event path: emit a domain event from Identity through the outbox → RabbitMQ → a stub consumer, traced end-to-end. Every subsequent service repo reuses this skeleton.
