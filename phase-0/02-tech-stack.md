# 02 · Tech Stack

All choices satisfy the stated constraint: **C# for the backend, Next.js + TypeScript for the web**, with an
iPhone client. Each row notes *why* and the *Phase 0 version*.

## 1. Backend (C# / .NET 10 microservices)

Each microservice is the same stack — a stateless ASP.NET Core service running **2–3 pods**.

| Concern | Choice | Why |
|---|---|---|
| Runtime | **.NET 10** | Required; latest LTS, top perf, minimal APIs, native AOT-ready |
| Web framework | **ASP.NET Core** (minimal APIs) | First-class, fast, OpenAPI built-in, small images |
| API gateway / BFF | **YARP** (.NET reverse proxy) | TLS, routing, authn, rate limiting in front of services |
| API style | REST + OpenAPI per service; **CQRS** internally | Simple contract; clean internals |
| Inter-service (async) | **MassTransit** over **RabbitMQ** | Reliable event bus, retries, idempotency, outbox integration |
| Inter-service (sync) | typed `HttpClient` / gRPC via gateway | Only when a request needs another service's current state |
| Mediator | **MediatR** (or built-in sender) | Decouples endpoints from handlers |
| ORM | **EF Core 10** (Npgsql provider) | Productive, migrations, LINQ; raw SQL when needed |
| Validation | **FluentValidation** | Declarative request + domain rules |
| Mapping | **Mapperly** (source-gen) | Fast, AOT-friendly DTO mapping |
| AuthN | **ASP.NET Core Identity** (Identity svc) + **stateless JWT** / refresh tokens | Stateless validation at every service; refresh tokens in Redis |
| Background work | **.NET Hosted Services** + **transactional outbox** | Reliable async event publish, crash-safe |
| Logging | **Serilog** → OTLP | Structured logs, correlation/trace IDs |
| Telemetry | **OpenTelemetry .NET SDK** | Distributed traces across service hops |
| Resilience | **Polly** (retry, circuit breaker, timeout) | Hardened sync calls between services |
| Testing | **xUnit**, **FluentAssertions**, **Testcontainers**, **NetArchTest**; **Coverlet** + **Stryker.NET** | Unit + integration + architecture tests; **100% coverage + mutation gate** ([12](./12-testing-standard.md)) |
| API docs | **OpenAPI** + **Scalar** UI | Live contract per service, aggregated at the gateway |

> **Statelessness rule:** services hold no in-pod session/cache that must survive a restart. JWTs are validated
> locally (no central session lookup); refresh tokens, rate-limit counters, and read caches live in **Redis**. This
> is what lets every service run 2–3 interchangeable pods behind a load balancer — see [01 §6](./01-architecture.md#6-statelessness--scaling-model).

## 2. Data layer

| Concern | Choice | Why |
|---|---|---|
| Primary DB | **PostgreSQL 16**, **one database per service** | ACID for money; service autonomy; no shared tables |
| Cache / sessions | **Redis 7** | Refresh tokens, rate-limit counters, read-model cache (shared, never pod-local) |
| Message broker | **RabbitMQ 3.13** | Async event integration across services (outbox → broker → consumers) |
| Migrations | **EF Core migrations** per service (CI-gated) | Versioned, reviewable schema changes, isolated per DB |
| Money type | `decimal` in C#, `numeric(19,4)` in PG | **Never** float for money |
| Currency | ISO-4217 code stored per amount; **multi-currency** | Each entity in its native currency; household has a base/reporting currency |
| FX rates | `EXCHANGE_RATE` table, `numeric(19,8)`, user-sourced (`MANUAL`/`SEED`) | No live FX feed in Phase 0; conversion done by the Net Worth service |

## 3. Web client (personal cabinet)

| Concern | Choice | Why |
|---|---|---|
| Framework | **Next.js 15 (App Router)** + **React 19** | SSR/RSC, great DX, matches the constraint |
| Language | **TypeScript (strict)** | Type safety end-to-end |
| Data fetching | **TanStack Query** + generated client | Cache, retries, optimistic updates |
| API types | **openapi-typescript** from backend's OpenAPI | Backend contract → TS types, zero drift |
| Styling | **Tailwind CSS** + **shadcn/ui** | Fast, consistent, accessible primitives |
| Charts | **Recharts** / **visx** | Net-worth + goal trend visualizations |
| Forms | **React Hook Form** + **Zod** | Validation mirrored from API rules |
| Auth | HTTP-only cookie (refresh) + in-memory access token | XSS-resistant session handling |
| State | Server state via Query; minimal client state (Zustand if needed) | Avoid over-engineering |
| Testing | **Vitest** (+coverage), **Testing Library**, **Playwright** | Unit + e2e; **100% coverage gate** ([12](./12-testing-standard.md)) |

## 4. iPhone app

**Decision (ratified): native Swift + SwiftUI.** Chosen for best iOS UX, smallest binary, full platform features,
and the smoothest App Store fit. The app is a **thin client** over the same REST API, so the only net-new surface is
the UI layer — business logic stays in the backend.

| Concern | Choice | Why |
|---|---|---|
| Language / UI | **Swift + SwiftUI** | Idiomatic iOS, best UX, native push/biometrics/Keychain |
| Min target | **iOS 17+** | Modern SwiftUI APIs (Observation, NavigationStack) |
| Networking | URLSession + generated client | See API client below |
| **API client** | **swift-openapi-generator** from backend OpenAPI | Backend contract → Swift types, zero drift (mirrors the web client) |
| Auth storage | **Keychain** (refresh token), in-memory access token | Secure session handling |
| Local state | SwiftData / lightweight cache | Offline-friendly read cache |
| Security | **Certificate pinning**, biometric (Face ID) unlock | Matches [06 · Security](./06-security.md) |
| Testing | XCTest + XCUITest | Unit + UI |

Alternatives considered and **not** chosen: **.NET MAUI** (C# reuse, but heavier/less-native UI) and
**React Native/Expo** (TS reuse, but not C# and adds a runtime). The decision stands; the thin, contract-driven
design merely keeps the cost of any future change low.

## 5. Monitoring stack

| Concern | Choice | Why |
|---|---|---|
| Instrumentation API | **OpenTelemetry** | One API for metrics/traces/logs |
| Metrics | **Prometheus** | De-facto standard, powerful querying (PromQL) |
| Dashboards | **Grafana** | Ops + financial-health panels in one place |
| Logs | **Loki** | Cheap, label-based, Grafana-native |
| Traces | **Tempo** | Distributed tracing, Grafana-native |
| Alerting | **Alertmanager** (+ Grafana alerts) | Route to email/Slack/push |
| Uptime | **Grafana Synthetic / blackbox exporter** | External "is it up?" checks |
| Errors | **Sentry** (optional) | Rich client + server error grouping |

Full design in [05 · Monitoring](./05-monitoring.md).

## 6. Platform & DevOps

| Concern | Choice | Why |
|---|---|---|
| Cloud | **Azure** (AWS valid alt) | Best .NET fit; managed Postgres/Redis/Key Vault |
| Orchestration | **Kubernetes** (AKS) | Run **2–3 pods per service**, rolling deploys, probes, autoscaling |
| Scaling | **HPA** floor 2 / ceiling 3 | Matches the 2–3-pods-per-service requirement |
| Ingress | **ingress-nginx** (the nginx baked into k8s) | L7 routing + TLS termination + per-IP rate limit; no hand-run standalone LB appliance |
| HTTPS / certs | **cert-manager + Let's Encrypt** (ACME **DNS-01** via Cloudflare) | Automated TLS issuance + renewal; works behind a proxied/IP-locked origin where HTTP-01 can't |
| Edge (recommended) | **Cloudflare** — DNS, CDN, WAF, DDoS, **Full (strict)** TLS | Managed edge protection; *optional* — Front Door/CloudFront are valid alts. See [01 §7.1](./01-architecture.md#71-edge-ingress--https) |
| Containers | **Docker**, multi-stage builds, distroless/chiseled base | Reproducible, small, fast-starting (helps stateless scale-out) |
| Local dev | **Docker Compose** (all services + Postgres + Redis + RabbitMQ + Grafana) or **.NET Aspire** | One command brings up the whole mesh |
| Service mesh | **Linkerd** (tool optional; mTLS itself is **required** per [09 SEC-NET-01](./09-security-standard.md#5-service-to-service--network-security-zero-trust)) | mTLS for all service-to-service traffic — via a mesh **or equivalent**; mesh also adds traffic metrics |
| IaC | **Terraform** (or **Bicep**) | Reproducible cluster + managed data services |
| CI/CD | **GitHub Actions** → build/test/archtest per service → push image → deploy manifests | Independent pipelines per service |
| GitOps (optional) | **Argo CD** / Helm | Declarative rollout of the K8s manifests |
| Secrets | **Azure Key Vault** (+ Sealed Secrets in-cluster) | No secrets in code/CI |
| Registry | **Azure Container Registry** | Signed images, per-service tags |

## 7. Repository strategy (multi-repo / polyrepo)

**Each microservice owns its own GitHub repository**, with independent CI/CD, versioning, branch protection, and
release cadence. Cross-cutting concerns live in dedicated **shared repos** consumed as **versioned packages**, never
copy-pasted. Full governance, ownership, and publishing rules in **[08 · Repositories & Contracts](./08-repositories.md)**.

| GitHub repo | Contains | Published artifact |
|---|---|---|
| `getdue-identity` | Identity service (4-project Clean Arch) | container image |
| `getdue-accounts` | Accounts service | container image |
| `getdue-debts` | Debts & Mortgages service | container image |
| `getdue-realestate` | Real Estate service | container image |
| `getdue-stocks` | Stocks Portfolio service | container image |
| `getdue-goals` | Goals service | container image |
| `getdue-networth` | Net Worth / Aggregation service | container image |
| `getdue-insights` | Insights service | container image |
| `getdue-gateway` | YARP API gateway / BFF | container image |
| `getdue-web` | Next.js + TypeScript web cabinet | container image |
| `getdue-mobile` | SwiftUI iPhone app | App Store build |
| **`getdue-contracts`** | **Event/message schemas, OpenAPI specs, shared DTOs** | NuGet + npm + Swift package |
| **`getdue-buildingblocks`** | Shared C# lib: Money VO, outbox, OTel, auth handlers, resilience | NuGet |
| **`getdue-platform`** | Terraform, Helm base charts, Kubernetes base manifests, policies | tagged release |
| **`getdue-deploy`** | GitOps (Argo CD app-of-apps) referencing each service's image tag | — |
| **`.github`** | Org-level **reusable workflows**, security policies (CODEOWNERS templates checked in as deferred controls — [08 §0](./08-repositories.md#0-ownership-model-phase-0)) | — |

Each **service repo** keeps the identical internal shape:

```
getdue-<service>/
├─ src/{Api,Application,Domain,Infrastructure}/
├─ tests/{Unit,Integration,Architecture}/
├─ deploy/
│  └─ k8s/        # Deployment (replicas 2, HPA max 3), Service, HPA, PDB, probes
├─ .github/workflows/   # build → test → archtest → scan → sign → publish image → deploy
├─ Dockerfile          # multi-stage, distroless/chiseled
├─ CODEOWNERS          # checked in; approval rule reactivates when the team grows (08 §0)
└─ SECURITY.md         # references the org security standard (doc 09)
```

> **Why polyrepo here:** independent deploy/rollback per service, blast-radius isolation, and per-repo branch
> protection — all properties a bank-grade platform wants. Per-team least-privilege access and separation of
> duties activate when the team grows ([08 §0](./08-repositories.md#0-ownership-model-phase-0)); the polyrepo
> shape is chosen now so those controls drop in without reshaping the org. The cost (cross-repo coordination,
> version bumps) is mitigated by the `contracts`/`buildingblocks` packages and org-level reusable workflows. A
> shared local dev experience is provided by the `getdue-platform` Docker Compose mesh.

## 8. Version pinning (Phase 0 targets)

| Component | Version |
|---|---|
| .NET | **10.x** |
| EF Core / Npgsql | 10.x |
| PostgreSQL | 16 (one DB per service) |
| Redis | 7.x |
| RabbitMQ | 3.13.x |
| Kubernetes | 1.30+ (AKS) |
| Next.js / React | 15 / 19 |
| Node | 22 LTS |
| Grafana / Prometheus / Loki / Tempo | latest stable |
