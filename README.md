# GetDue — Cloud Home Bank · Documentation

A personal "home bank in the cloud": mobile app (iPhone), a personal web cabinet, and a monitoring system,
backed by C#/.NET 10 stateless microservices (2–3 pods each) on Kubernetes.

## Phase 0 — Foundation Slice

End-to-end product skeleton with **no real financial-institution integrations** and **no real card issuance**.
The user records their financial life (accounts, debts, real estate, mortgages, stocks, goals) and the platform
aggregates, tracks, and monitors it.

➡️ **Start here: [Phase 0 documentation](./phase-0/README.md)**

| Doc | What's inside |
|---|---|
| [00 · Overview & Scope](./phase-0/00-overview.md) | Vision, personas, journeys, net-worth model, guardrails |
| [01 · Architecture](./phase-0/01-architecture.md) | C4 diagrams, Clean Architecture, deployment, ADRs |
| [02 · Tech Stack](./phase-0/02-tech-stack.md) | .NET 10 microservices, RabbitMQ, Kubernetes, Next.js, SwiftUI, monitoring |
| [03 · Domain Model](./phase-0/03-domain-model.md) | Entities, ERD, aggregates, valuation snapshots |
| [04 · API Design](./phase-0/04-api-design.md) | REST contract, conventions, sample payloads |
| [05 · Monitoring System](./phase-0/05-monitoring.md) | Ops + financial-health observability, alerting, SLOs |
| [06 · Security & Compliance](./phase-0/06-security.md) | AuthN/Z, data protection, Phase 0 compliance posture |
| [07 · Roadmap & Phasing](./phase-0/07-roadmap.md) | Milestones, Definition of Done, what unlocks Phase 1 |
| [09 · Security Standard](./phase-0/09-security-standard.md) | Enforceable application/runtime security rules for every service |
| [10 · Client Dashboard & Analytics](./phase-0/10-dashboard-analytics.md) | Money analytics home screen: net worth, allocation, debt, goals, KPIs |

## Engineering Handbook

Engineering process and tooling — **not tied to a phase** — live in [`./engineering/`](./engineering/README.md).

| Doc | What's inside |
|---|---|
| [01 · Repositories & Contracts](./engineering/01-repositories.md) | Multi-repo (one per service), shared contracts/libs, CI/CD, branch protection |
| [02 · Versioning System](./engineering/02-versioning.md) | Cloud-wide versioning: API, services, contracts, events, DB schema, infra |
| [03 · Testing Standard](./engineering/03-testing-standard.md) | Mandatory 100% line+branch coverage + mutation testing, enforced in CI |
| [04 · Secure SDLC](./engineering/04-secure-sdlc.md) | Change-control governance + CI security gates + supply chain |
