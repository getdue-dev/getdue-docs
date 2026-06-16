# 08 · Cost & FinOps

Cost is a **first-class concern**, even for a solo learning/portfolio project. The architecture here is
deliberately complex for resume and learning value — 8 stateless .NET services + a gateway on AKS, a web app, a
mobile client, Postgres/Redis/RabbitMQ, and a full OTel/Grafana observability stack. That complexity is the point
and is **not** to be simplified away. But a portfolio project that quietly exhausts a personal credit card gets
switched off — so the *cost envelope* has to be designed as carefully as the architecture. This doc is the missing
FinOps view: what drives spend, what it costs per environment and per scale, and how to run the whole thing for
**tens of dollars rather than thousands** while it is mostly idle.

> **All figures are rough order-of-magnitude** (Azure, USD/month) for orientation only. Real numbers depend on
> region, tier, reserved capacity, and live traffic — **refine with the [Azure pricing calculator](https://azure.microsoft.com/pricing/calculator/)**
> before committing budget. Use these as relative weights, not quotes.

## 1. Cost drivers

| Driver | What it is (Phase 0) | Why it costs |
|---|---|---|
| **AKS node pool** | Compute for 8 services + gateway + web, each at an **HA floor of ≥2 pods, autoscaling 2→10** ([01 §6](./01-architecture.md#6-statelessness--scaling-model)) | The dominant line item: ~18+ pods at floor, more under load. Sized by node VM family × count |
| **PostgreSQL** | **One Flexible Server hosting 8 separate databases**, each with its own least-privilege role ([02 §2](./02-tech-stack.md#2-data-layer)) | A deliberate cost choice: the database-per-service *logical* boundary is preserved on a single server. True per-instance isolation (one server per service) is a **Target/Phase 1** option that multiplies this line ~8× |
| **Redis** | Cache + refresh tokens + rate-limit counters (shared, never pod-local) | One managed instance; small tier suffices in Phase 0. Scales with connections/memory |
| **RabbitMQ** | Async event bus (outbox → broker → consumers) | **Managed** (e.g. CloudAMQP) or **self-hosted StatefulSet** in-cluster — self-hosted trades dollars for ops effort |
| **Observability** | Self-hosted Grafana/Prometheus/Loki/Tempo **OR** Azure Managed Grafana + Monitor; includes **WORM audit-log storage (≥1 yr)** | Either in-cluster compute + storage, or per-GB ingest/retention on managed. The audit-log WORM bucket is a compliance line that only grows |
| **Edge / WAF** | Cloudflare or Azure Front Door (DNS, CDN, WAF, DDoS, TLS) ([02 §6](./02-tech-stack.md#6-platform--devops)) | Free tier covers Phase 0; WAF/Front Door tiers add cost at scale |
| **ACR** | Azure Container Registry — signed images, per-service tags | Per-GB storage + bandwidth; Basic/Standard tier |
| **Key Vault** | Secrets, certs ([02 §6](./02-tech-stack.md#6-platform--devops)) | Per-operation + per-secret; negligible at this scale |
| **Object storage** | Attachments + **WORM audit log** archive | Per-GB; grows linearly with usage and retention |
| **Egress** | Outbound data (APNs, email, HIBP, FX seeds, backups, user traffic) | Per-GB outbound; small in Phase 0, watch as traffic grows |

> **The pod ceiling is a cost cap.** `maxReplicas: 10` per service is not a scaling limit — it is the
> Phase-0 spend ceiling. Compute scales with replicas, so the HPA ceiling directly bounds the AKS bill.

## 2. Per-environment monthly estimate

Three environments, very different cost profiles. Ranges are rough and per-driver.

| Driver | `local` (Docker Compose) | `staging` (smallest viable) | `production` (Phase 0) |
|---|---|---|---|
| Compute / AKS | $0 | $30–80 (single small node) | $150–700 (node pool at floor → load) |
| PostgreSQL | $0 | $15–40 (B-series burstable) | $80–350 (Flexible Server, 1 server / 8 DBs) |
| Redis | $0 | $0–20 (in-cluster or smallest tier) | $20–150 |
| RabbitMQ | $0 | $0 (in-cluster) | $0–120 (in-cluster vs managed) |
| Observability | $0 | $0–20 (Grafana Cloud free tier) | $30–200 (self-hosted compute + WORM storage, or managed ingest) |
| Edge / WAF | $0 | $0 (Cloudflare free) | $0–80 |
| ACR / Key Vault / storage / egress | $0 | $5–20 | $20–100 |
| **Total (rough)** | **$0** | **~$50–200** | **~$400–1,600** |

> **`local` is free by design.** The full mesh — all services + Postgres + Redis + RabbitMQ + the Grafana stack —
> runs under Docker Compose / .NET Aspire on a laptop ([02 §6](./02-tech-stack.md#6-platform--devops),
> [05 §8](./05-monitoring.md#8-local--cloud-parity)). Most day-to-day development costs nothing.

> **`staging` is intentionally minimal:** a single small node, burstable Postgres, in-cluster Redis/RabbitMQ,
> free-tier observability and edge. It exists to validate deploys and parity, not to take load.

## 3. Cost by scale

Phase-0 cost is driven by the **pod ceiling**, not raw user count — the cluster sits near its floor until traffic
pushes the HPA toward `maxReplicas: 10`. The single-region, single-broker, single-Postgres-server topology holds
comfortably to ~100k users; **beyond that the ceiling and the topology must be revisited — that is Phase 1**
([07 · Roadmap](./07-roadmap.md)).

| Users | Cloud / AKS | Database | Observability | Storage | AI | Total (rough) | Notes |
|---|---|---|---|---|---|---|---|
| **1** | $150 (floor pods) | $80 | $30 | ~$0 | $0 | **~$260** | Floor-dominated; cost ≈ fixed baseline |
| **1k** | $180 | $100 | $40 | $5 | $0 | **~$325** | Still near floor; HPA rarely fires |
| **10k** | $300 | $180 | $80 | $30 | $0 | **~$590** | HPA starts scaling; storage growing |
| **100k** | $700 (near ceiling) | $350 | $200 | $150 | $0 | **~$1,400** | **At the Phase-0 ceiling** — `maxReplicas: 10` saturating |
| **1M** | revisit | revisit | revisit | revisit | $0 | **Phase 1** | Ceiling, single region, single broker/server topology must be re-architected |

> **AI cost is $0.** There are **no LLMs in Phase 0** — insights are computed by a scheduled hosted service over
> valuation snapshots ([05 §5](./05-monitoring.md#5-financial-health-monitoring-the-product-facing-half)), not a model.

> **Storage grows linearly and needs a retention policy.** Valuation snapshots and audit logs accumulate forever
> if left unmanaged. Telemetry already has tiered retention (traces 7d, logs 30d, metrics 13mo —
> [05 §9](./05-monitoring.md#9-data-privacy-in-telemetry)); the **WORM audit log keeps ≥1 yr** and is the one line
> that legitimately only grows. Set retention/archival on snapshots and non-audit storage explicitly.

## 4. Dev / cost-saving mode

How to keep the learning cluster cheap and alive — running this portfolio project for **tens of dollars, not
thousands** — without simplifying the architecture:

| Lever | Effect |
|---|---|
| **AKS spot node pools** | Run non-critical workloads on spot VMs at a fraction of on-demand price |
| **Scale-to-zero / stop the cluster when idle** | Stop the AKS cluster (or scale node pools to 0) when you are not actively demoing — the single biggest saver for a project that is idle most of the time |
| **Grafana Cloud free tier** | Replace self-hosted observability compute + storage with the managed free tier |
| **Azure free credits / free tiers** | New-account and student credits; free-tier Key Vault ops, small storage |
| **B-series burstable Postgres** | Burstable Flexible Server instead of a provisioned tier — fine for low, spiky dev load |
| **Single small node for staging** | One node hosts the whole staging mesh; no HA there |
| **Self-hosted RabbitMQ / Redis in-cluster** | StatefulSets instead of managed services — trades dollars for ops effort (acceptable for a learning project) |
| **Cloudflare free tier** | DNS, CDN, WAF, DDoS, Full (strict) TLS at $0 |

> **The discipline:** treat "cluster running" as a billable action. Bring the production cluster up to demo or to
> exercise a feature, then stop it. `local` (Docker Compose) is the default workspace; the cloud is for proving it
> works end-to-end, not for sitting idle.

## 5. FinOps practices

Lightweight habits that keep the bill honest:

- **Tag every resource** (environment, service, owner) so cost reports break down by what you can actually act on.
- **Set an Azure Budget + cost alert** on the subscription — get emailed *before* the card is the surprise, not after.
- **Review spend monthly**, against the per-environment estimates above; investigate any driver that drifts.
- **Right-size from telemetry**, not guesswork — use the runtime/DB/pod metrics in
  [05 · Monitoring](./05-monitoring.md) to size VMs, replicas, and DB tiers to observed load (USE/RED panels show
  whether you are over- or under-provisioned).
- **Reserved / savings plans only once usage is stable** — commitments save money only when the baseline is known;
  for a project that scales to zero between demos, on-demand + spot is usually cheaper.

## 6. Cross-links

- [01 · Architecture](./01-architecture.md) — the HPA pod ceiling (`maxReplicas: 10`) as the Phase-0 cost cap.
- [02 · Tech Stack](./02-tech-stack.md) — the components priced above (AKS, Postgres single-server topology, Redis, RabbitMQ, edge).
- [05 · Monitoring](./05-monitoring.md) — telemetry that drives right-sizing and retention.
- [07 · Roadmap](./07-roadmap.md) — when the ceiling and single-region/single-broker topology get revisited (Phase 1).
