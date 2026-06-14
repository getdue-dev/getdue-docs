# 05 · Monitoring System

The monitoring system is a **first-class Phase 0 deliverable**, not an afterthought. It has **two faces**:

1. **Operational monitoring** — for you, the operator: is it up, fast, correct?
2. **Financial-health monitoring** — for the user: is *their* money on track?

Both are powered by the same telemetry pipeline (OpenTelemetry → Prometheus/Loki/Tempo → Grafana).

## 1. Telemetry pipeline

```mermaid
graph LR
    subgraph App[".NET 10 microservices (2–3 pods each) + Web"]
        OT[OpenTelemetry SDK\ntraces · metrics · logs\nper service/pod]
    end
    OT -->|OTLP| Col[OTel Collector]
    Col --> Prom[(Prometheus\nmetrics)]
    Col --> Loki[(Loki\nlogs)]
    Col --> Tempo[(Tempo\ntraces)]
    Prom --> Graf[Grafana]
    Loki --> Graf
    Tempo --> Graf
    Prom --> AM[Alertmanager]
    AM --> Notify[Email / Slack / Push]
    BB[Blackbox exporter\nuptime probes] --> Prom
```

- **One instrumentation API:** OpenTelemetry. Backends are swappable without code changes (ADR-005).
- **Correlation:** every log line and metric exemplar carries the active `traceId`, so a Grafana panel jumps
  log → trace → metric in one click.

## 2. The three pillars

### Metrics (Prometheus)
| Category | Example metrics |
|---|---|
| **RED (per endpoint)** | `http_server_request_duration_seconds` (Rate, Errors, Duration) |
| **Runtime** | GC pauses, heap, thread pool, CPU, working set |
| **EF Core / DB** | query duration, active connections, pool saturation (per-service DB) |
| **Outbox** | pending events, dispatch lag, retries, dead-letter count |
| **Broker (RabbitMQ)** | queue depth, consumer lag, unacked messages, publish/consume rate, DLQ size |
| **Per-service / per-pod** | replica count vs desired (2–3), pod restarts, readiness flaps |
| **Business** | net-worth recompute count, goals created, snapshots written |

> All RED/runtime metrics are labeled by `service` and `pod`, so dashboards break down per microservice and per
> replica — essential for spotting one unhealthy pod among the 2–3 behind a service.

### Logs (Serilog → Loki)
- Structured JSON, one event per line, with `traceId`, `householdId` (hashed), `userId` (hashed), `requestId`.
- Levels: `Information` (business events), `Warning` (recoverable), `Error` (faults). **No PII or money values in logs.**

### Traces (Tempo)
- End-to-end spans: HTTP → handler → EF query → outbox dispatch → projector.
- Used to debug latency and to prove the valuation/net-worth flow executed correctly.

## 3. Health checks

| Endpoint | Checks | Used by |
|---|---|---|
| `GET /v1/health` | process alive | Container liveness probe |
| `GET /v1/health/ready` | Postgres reachable, Redis reachable, migrations applied, outbox drainable | Load balancer / readiness probe |

Implemented with **ASP.NET Core HealthChecks** (`AddNpgSql`, `AddRedis`, custom outbox check).

## 4. Operational dashboards (Grafana)

| Dashboard | Panels |
|---|---|
| **Service Overview** | RPS, error rate, p50/p95/p99 latency, saturation (USE), uptime % |
| **API Detail** | per-endpoint RED, top slow endpoints, status-code heatmap |
| **Data Layer** | DB query latency, connection pool, slow queries, deadlocks |
| **Async / Outbox** | pending backlog, dispatch lag, retry/DLQ counts |
| **Runtime** | GC, memory, CPU, thread pool by pod |

## 5. Financial-health monitoring (the product-facing half)

This is what makes a *home bank* monitoring system, not just an ops dashboard. It watches the user's **data**, not
the servers, and surfaces via `GET /v1/insights` + push notifications.

| Signal | Rule (example) | Surfaced as |
|---|---|---|
| **Net-worth drop** | net worth ↓ > X% month-over-month | Insight + push |
| **Goal off-track** | projected completion date > target date | Goal status `AT_RISK` |
| **Goal achieved** | `current ≥ target` | Status `ACHIEVED` + celebration |
| **Mortgage/loan due** | next `SCHEDULED_INSTALLMENT.due_date` within N days, still unpaid | Reminder push |
| **Loan paid off** | `LoanPaidOff` (full early payoff) | Celebration + net-worth update |
| **High leverage** | total liabilities / total assets > threshold | Insight |
| **Stale data** | an entity not updated in > 90 days | "Update your balances" nudge |
| **Concentration risk** | one holding > X% of portfolio | Insight |

These are computed by a **scheduled hosted service** that reads the valuation snapshots and emits
`Insight` / `GoalStatusChanged` events. Thresholds are configuration, tunable per household later.

```mermaid
graph TB
    Sched[Daily Insights Job] --> Read[Read valuation snapshots]
    Read --> Eval{Evaluate rules}
    Eval -->|net worth drop| I1[Insight]
    Eval -->|goal off-track| I2[GoalStatusChanged]
    Eval -->|payment due| I3[Reminder]
    I1 & I2 & I3 --> Store[(insights table)]
    Store --> API[GET /v1/insights]
    Store --> Push[Push / email]
```

## 6. Alerting (Alertmanager + Grafana alerts)

| Severity | Example condition | Route |
|---|---|---|
| **Critical (page)** | error rate > 5% for 5m; readiness failing; DB down; service below 2 healthy pods | Operator — immediate |
| **Warning** | p95 latency > 1s for 10m; outbox/broker backlog > N; consumer lag rising; disk > 80% | Operator — batched |
| **Info** | deploy completed; nightly job finished | Channel log |
| **User-facing** | goal at-risk, payment due, goal achieved | End user — push/email |

Every alert links to a **runbook** entry (`docs/runbooks/`) describing diagnosis + remediation.

## 7. SLOs (Phase 0 targets)

| SLO | Target |
|---|---|
| API availability | 99.5% monthly |
| Read latency (p95) | < 300 ms |
| Write latency (p95) | < 600 ms |
| Net-worth recompute lag | < 5 s after an edit |
| Error budget | 0.5% / month, burn-rate alerted |

## 8. Local & cloud parity

- **Local:** `ops/docker-compose.yml` brings up Postgres, Redis, OTel Collector, Prometheus, Grafana, Loki, Tempo —
  the *same* dashboards run on a laptop as in prod.
- **Cloud:** the Grafana stack runs alongside the app (or use **Azure Managed Grafana** + **Azure Monitor** if you
  prefer managed). The instrumentation code is identical either way.

## 9. Data privacy in telemetry

- Logs/metrics/traces carry **hashed** identifiers, never raw email/name.
- **No monetary amounts** in telemetry — only counts/durations and ratios. (Net-worth *values* live in the DB and
  are shown only to the authenticated owner via the API.)
- Telemetry retention: traces 7d, logs 30d, metrics 13mo (tunable).
