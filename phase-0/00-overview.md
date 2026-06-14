# 00 · Overview & Scope

## 1. Vision

**GetDue** is a *personal home bank in the cloud*: a single place where a person (and their household) can model
their entire financial life — what they own, what they owe, what they're saving for — and watch it move over time.

Phase 0 is the **foundation slice**. It proves the architecture end-to-end without taking on the regulatory and
integration weight of connecting to real banks or issuing real instruments. Every number in Phase 0 is **entered by
the user**; the platform's job is to **store, aggregate, visualize, and monitor**.

## 2. Goals of Phase 0

| Goal | Success criterion |
|---|---|
| Prove the architecture | One API serves web + mobile; clean separation of concerns |
| Model the core domain | All six entity types persist and validate correctly |
| Compute net worth | Aggregation across all entities is correct and auditable |
| Track goals | A goal shows progress %, target date, and funding trend |
| Be observable | Every request traced; financial-health metrics dashboarded; alerts fire |
| Be safe-by-default | Auth, encryption at rest/in transit, no real-money side effects |

## 3. Non-goals (Phase 0)

- No real bank/brokerage connectivity.
- No card issuance or payment rails.
- No automated market pricing.
- No **live** FX rate feed — **multi-currency is supported**, but exchange rates are user-sourced/seeded in Phase 0
  (a live FX provider is Phase 1), consistent with all other valuations being manual.
- No tax engine, no advisory/recommendations.

## 4. Personas

| Persona | Needs | Primary surface |
|---|---|---|
| **Account owner** | See net worth, manage entities, set goals | iPhone app + web |
| **Household co-member** | Shared view of joint finances (read or co-edit) | Web |
| **Platform operator (you)** | Uptime, errors, latency, data integrity | Monitoring system |

## 5. Core user journeys (Phase 0)

1. **Onboard** → register, verify email, create household, pick base currency.
2. **Build the picture** → add bank accounts, loans, real estate, mortgages, stock holdings.
3. **See the number** → the **client dashboard** ([10](./10-dashboard-analytics.md)): net worth, allocation, debt, goals, currency exposure, and trends at a glance.
4. **Aim** → create a financial goal (e.g., "€50k emergency fund by 2027"), link funding source, watch progress.
5. **Stay informed** → receive monitoring/health signals (e.g., "mortgage payment due", "goal off-track").

## 6. Net worth model (the heart of Phase 0)

Each entity is held in **its own native currency** (a USD account, a EUR property, a GBP mortgage can coexist). Net
worth rolls everything up into the **household's base currency** using exchange rates:

```
Net Worth(base) = Σ Assets(→base) − Σ Liabilities(→base)

  where  X(→base) = X.amount × fxRate(X.currency → base, as_of)

Assets       = Bank Account balances
             + Real Estate market values
             + Stock Portfolio market values (qty × manual/seeded price)

Liabilities  = Loan Debt outstanding balances
             + Mortgage Loan outstanding principal
```

- Amounts in the **base currency** convert at rate `1.0`.
- The user can also view net worth in any **display currency** (a presentation-time conversion).
- Every change to an entity produces a **valuation snapshot** (in native currency); the FX rate used is recorded, so
  net worth is **reconstructable and auditable** for any past date — powering the trend charts and the
  financial-health monitoring in [05](./05-monitoring.md).

## 7. Scope guardrails

To keep Phase 0 honest, the codebase enforces these invariants:

- **No outbound calls** to financial-institution APIs exist in the build (lint/architecture test fails if added).
- **No payment/transfer endpoints** are exposed — money never moves.
- **All valuations and FX rates are user-sourced** — stored with `source = MANUAL` (or `SEED` for demo data); no
  live market or FX feed is called.

These guardrails are documented as architecture tests (see [01 · Architecture](./01-architecture.md#9-architecture-tests)).
