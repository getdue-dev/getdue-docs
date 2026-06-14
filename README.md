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
| [08 · Repositories & Contracts](./phase-0/08-repositories.md) | Multi-repo (one per service), shared contracts/libs, branch protection |
| [09 · Security Standard](./phase-0/09-security-standard.md) | Bank-grade, enforceable security rules for every service |
| [10 · Client Dashboard & Analytics](./phase-0/10-dashboard-analytics.md) | Money analytics home screen: net worth, allocation, debt, goals, KPIs |
| [11 · Versioning System](./phase-0/11-versioning.md) | Cloud-wide versioning: API, services, contracts, events, DB schema, infra |
| [12 · Testing Standard](./phase-0/12-testing-standard.md) | Mandatory 100% line+branch coverage + mutation testing, enforced in CI |
