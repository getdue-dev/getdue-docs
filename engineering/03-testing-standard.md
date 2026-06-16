# 03 · Testing Standard — 100% Coverage, Enforced

> **Status:** Mandatory baseline for **every** repo (services, gateway, web, mobile, shared libs). **100% code
> coverage is a hard, release-blocking CI gate** — no merge to the default branch and no deploy below the threshold.
> Conformance language is RFC 2119 (**MUST** / **SHOULD**), as in [09 · Security Standard](../phase-0/09-security-standard.md).

## 0. The rule

- **TEST-COV-01 (MUST)** Every repo enforces **100% line coverage AND 100% branch coverage** as a CI gate. The build
  **fails** below 100%; coverage cannot regress.
- **TEST-COV-02 (MUST)** Coverage is **measured on every PR** and reported as a required status check ([08 §6](./01-repositories.md#6-cicd-per-service-repo)); a PR cannot merge red.
- **TEST-COV-03 (MUST)** 100% is enforced **after** the agreed exclusions in [§4](#4-what-counts-what-is-excluded) — and
  every exclusion is explicit, justified, and reviewed. "100%" therefore means *100% of code that carries logic*, never
  a silent denominator shrink.
- **TEST-COV-04 (MUST)** A **mutation-testing** score gate backs the line/branch gate so coverage cannot be gamed with
  assertion-free tests ([§5](#5-mutation-testing-anti-gaming)).

> **Why define "100%" precisely:** an undefined 100% rule is gamed two ways — by excluding files until the number hits
> 100, or by writing tests that execute lines without asserting anything. This standard closes both: exclusions are a
> reviewed allow-list ([§4](#4-what-counts-what-is-excluded)), and mutation testing proves the tests actually assert
> behavior ([§5](#5-mutation-testing-anti-gaming)).

## 1. Test pyramid (what to write)

| Level | Scope | Tooling | Where the coverage comes from |
|---|---|---|---|
| **Unit** | `Domain` + `Application` logic in isolation | xUnit + FluentAssertions; Vitest (web); XCTest (iOS) | the bulk of coverage |
| **Integration** | `Infrastructure` against **real** Postgres/Redis/RabbitMQ | **Testcontainers** | EF queries, outbox, consumers |
| **Contract** | producer/consumer message + API compatibility | Pact-style / schema validation ([08 §5](./01-repositories.md#5-versioning--dependency-flow)) | event & API shape |
| **Architecture** | layering, tenant filter, guardrails | NetArchTest | invariants ([01 §9](../phase-0/01-architecture.md#9-architecture-tests)) |
| **E2E** | critical user journeys end-to-end | Playwright (web), XCUITest (iOS) | smoke, not counted toward unit coverage |

Coverage is measured across unit + integration (the layers that execute application code). E2E/UI tests are required
for confidence but are **not** the vehicle for hitting 100% — they're slow and flaky as a coverage tool.

## 2. Tooling per stack

| Stack | Coverage tool | Gate enforcement |
|---|---|---|
| **.NET 10 services / libs** | **Coverlet** (line + branch) + **ReportGenerator** | `dotnet test` fails under threshold; `--threshold 100 --threshold-type line --threshold-type branch` |
| **Next.js / TypeScript** | **Vitest** (V8/c8 coverage) | `vitest --coverage` with `thresholds: { lines: 100, branches: 100, functions: 100, statements: 100 }` |
| **SwiftUI / iOS** | **XCTest** code coverage (Xcode/`xccov`) | CI parses `xccov`, fails under 100% |

## 3. CI enforcement

Each repo's reusable pipeline ([08 §6](./01-repositories.md#6-cicd-per-service-repo)) runs the coverage gate as a
**required, blocking** step:

```
build → unit + integration tests (Testcontainers) →
COVERAGE GATE: line 100% AND branch 100% (fail otherwise) →
MUTATION GATE: mutation score ≥ threshold (§5) →
architecture tests → security gates (§09) → publish image
```

- The **coverage report is published** as a PR artifact/comment and as a required status check
  ([01 §7 branch protection](./01-repositories.md#7-branch-protection-all-repos)).
- **No ratchet-down:** the threshold is fixed at 100; it is never lowered to make a build pass.

## 4. What counts, what is excluded

100% applies to **code that carries behavior**. The following are **excluded** — but only via an explicit,
version-controlled allow-list (e.g. `[ExcludeFromCodeCoverage]` with a reason, or coverage-config globs), each
reviewed in PR:

| Excluded | Why | How |
|---|---|---|
| **Generated code** | EF migrations, Mapperly mappers, OpenAPI/Swift/TS clients, source generators | not authored; excluded by path/attribute |
| **Composition root** | `Program.cs` / DI wiring with no branching | exercised by integration boot, but excluded if pure wiring |
| **Plain data holders** | records/DTOs with no logic (no methods/branches) | nothing to assert |
| **Framework glue** | trivial `[ApiController]` plumbing with no logic | covered by integration where it has behavior |

Anything with a **branch, a calculation, or a side effect** is **never** excluded — money math, validation,
authorization filters, outbox/idempotency, event handlers, and FX conversion are all in scope and must be fully
covered, including error/edge branches.

## 5. Mutation testing (anti-gaming)

- **TEST-MUT-01 (MUST)** Run **mutation testing** (**Stryker.NET** for C#, **StrykerJS** for TS) on `Domain` +
  `Application`; the **mutation score MUST be ≥ 85%** (target 100% for money-critical modules: net-worth aggregation,
  FX conversion, loan amortization/payment, idempotency).
- A surviving mutant in money math, authorization, or tenant-isolation code is a **release blocker** — these paths
  carry financial and security risk and must be killed by an assertion.
- Mutation runs can be scoped/sampled per PR (changed files) and run in full nightly, to keep PR latency reasonable.

## 6. Test quality rules

- **TEST-Q-01 (MUST)** Tests assert **behavior and outcomes**, not implementation details; no assertion-free tests.
- **TEST-Q-02 (MUST)** Integration tests use **real** dependencies via Testcontainers — **no** mocking of the
  database, broker, or cache (mock only true externals, of which Phase 0 has none).
- **TEST-Q-03 (MUST)** **Determinism:** no real clock/RNG/network in unit tests; inject time and ids. Flaky tests are
  treated as failing and quarantined+fixed, never retried-until-green.
- **TEST-Q-04 (MUST)** **Money & multi-currency** logic has property-based / boundary tests (rounding, currency
  mismatch rejection, missing-FX-rate path) — see [03 §5](../phase-0/03-domain-model.md#5-multi-currency-model).
- **TEST-Q-05 (SHOULD)** New bug fixes ship with a **regression test** that fails before the fix.

## 7. Acceptance gate addition

This extends the per-service release gate in [09 §13](../phase-0/09-security-standard.md#13-per-service-security-acceptance-gate-release-checklist):

- [ ] **100% line + branch coverage** enforced in CI, report attached (TEST-COV-01/02)
- [ ] Exclusions limited to the reviewed allow-list (TEST-COV-03)
- [ ] **Mutation score ≥ 85%** (100% for money-critical modules); no surviving mutants in money/authz/tenant code (TEST-MUT-01)
- [ ] Integration tests run against real Postgres/Redis/RabbitMQ via Testcontainers (TEST-Q-02)
- [ ] Architecture/guardrail tests green ([01 §9](../phase-0/01-architecture.md#9-architecture-tests))

## 8. Trade-offs (recorded honestly)

100% coverage is a deliberate, high bar. It is justified here because this is **financial data** with strict
correctness and audit needs, and the codebase is greenfield (cheaper to start at 100 than to retrofit). The known
costs and how this standard contains them:

| Cost | Containment |
|---|---|
| Coverage can be gamed | Mutation gate (§5) proves assertions are real |
| 100% can be hit by excluding files | Exclusions are a reviewed allow-list (§4), not silent |
| Diminishing returns on trivial code | Trivial data holders / generated code are excluded, so effort lands on logic |
| Slower PRs | Mutation sampled per-PR + full nightly; Testcontainers cached |

> If the 100% line-coverage gate ever proves counterproductive for a specific repo, the change is a **documented,
> security-owner-approved** adjustment to *this standard* — never an ad-hoc per-PR threshold lowering.
