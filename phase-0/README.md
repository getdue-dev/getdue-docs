# GetDue — Cloud Home Bank · Phase 0 Documentation

> **Phase 0 = Foundation Slice.** A working, end-to-end skeleton of a personal "home bank in the cloud" with
> **no integrations to real financial institutions** and **no issuance of real bank cards**. Everything is
> self-contained: the user manually records their financial reality (accounts, debts, property, mortgages,
> stocks, goals) and the platform aggregates, tracks, and monitors it.

> **Ownership model (Phase 0).** Solo-maintained. All repos live in the **public** GitHub org `getdue-dev`, and
> `main` is protected — plain **GitHub Flow** (feature branch → PR → green CI → merge; no direct pushes, no
> force-push). Engineering process, tooling, and the CI security pipeline live in the
> **[engineering handbook](../engineering/README.md)**, separate from these product docs.

## Product surface (3 clients, 1 backend)

| Surface | Description | Phase 0 status |
|---|---|---|
| **iPhone mobile app** | Primary day-to-day client | Thin client over the same API |
| **Personal client web page** | Full-feature personal cabinet | Primary build target |
| **Monitoring system** | Operational + financial-health dashboards | Operator + user views |

## What Phase 0 delivers

1. **Identity** — register/login a user, profile, household.
2. **Manual financial entities** — Bank Accounts, Loan Debts, Real Estate, Mortgage Loans, Stock Portfolio (each in its **native currency**).
3. **Loan schedules & payments** — import/generate an amortization schedule; record regular, **partial-prepayment**, and **full early-payoff** payments with automatic re-amortization (recorded ledger entries — no money moves).
4. **Financial Goals** — define, fund, and track goals against net worth.
5. **Multi-currency net-worth aggregation** — entities in native currencies, rolled up to the household base (and any display) currency via user-sourced FX rates.
6. **Client dashboard & money analytics** — net worth, asset allocation, debt breakdown, goal progress, currency exposure, and KPIs at a glance ([10](./10-dashboard-analytics.md)).
7. **Monitoring** — system health (ops) **and** financial health (product), with alerting.

## What Phase 0 explicitly excludes

- ❌ No Open Banking / Plaid / bank API connections — balances are entered by the user.
- ❌ No card issuing / processor integration.
- ❌ No real payments, transfers, or money movement.
- ❌ No live market data feeds (stock prices are manual or seeded; live feeds are Phase 1+).
- ❌ No KYC/AML provider, no credit bureau.

## Document index

| # | Document | Purpose |
|---|---|---|
| 00 | [Overview & Scope](./00-overview.md) | Goals, personas, scope guardrails |
| 01 | [Architecture](./01-architecture.md) | System, deployment, and runtime architecture |
| 02 | [Tech Stack](./02-tech-stack.md) | Concrete technology choices and rationale |
| 03 | [Domain Model](./03-domain-model.md) | Entities, aggregates, relationships, data dictionary |
| 04 | [API Design](./04-api-design.md) | REST contract, conventions, sample payloads |
| 05 | [Monitoring System](./05-monitoring.md) | Observability, financial-health monitoring, alerting |
| 06 | [Security & Compliance](./06-security.md) | AuthN/Z, data protection, posture for Phase 0 (design narrative) |
| 07 | [Roadmap & Phasing](./07-roadmap.md) | Phase 0 milestones and what unlocks Phase 1 |
| 08 | [Cost & FinOps](./08-cost-finops.md) | Per-environment cost model, cost-by-scale, dev cost-saving mode |
| 09 | [Security Standard](./09-security-standard.md) | **Enforceable application/runtime security rules every service must meet** |
| 10 | [Client Dashboard & Analytics](./10-dashboard-analytics.md) | Money analytics, net worth, allocation, debt & goal KPIs |

> **Engineering process** (repositories, CI/CD, versioning, testing, secure SDLC) lives separately in the
> **[engineering handbook](../engineering/README.md)** — it is not tied to Phase 0.

## Tech stack at a glance

- **Backend:** C# / **.NET 10**, **stateless microservices** (≥2 pods each, autoscaling), ASP.NET Core, Clean Architecture + DDD, EF Core.
- **Orchestration:** **Kubernetes** (AKS) · **RabbitMQ** event bus · YARP API gateway.
- **Database:** PostgreSQL 16 **(database-per-service)** (+ Redis for cache/refresh-tokens, never pod-local).
- **Web client:** Next.js 15 (App Router) + TypeScript + React.
- **Mobile (iPhone):** native Swift/SwiftUI (ratified) — thin client over the same OpenAPI contract.
- **Monitoring:** OpenTelemetry → Prometheus + Grafana + Loki + Tempo; Serilog; health checks; Alertmanager.
- **Cloud:** Azure (primary recommendation) — **AKS** (Kubernetes), Postgres Flexible Server, Key Vault.
