# 04 · Secure SDLC, Governance & Supply Chain

The engineering-process companion to the application **[Security Standard](../phase-0/09-security-standard.md)**.
This document owns **change-control governance** and the **CI security pipeline**; the phase-0 standard owns the
runtime/application controls (auth, network, data, logging, IR).

## 0. Solo-phase scope

Phase 0 is solo-maintained in the `getdue-dev` org. Governance is intentionally light — **GitHub Flow** on
**public repos + a protected `main`** ([01 §7](./01-repositories.md#7-branch-protection-all-repos)); repos are public
because classic branch protection only applies to public repos on the GitHub Free plan. The CI security gates below
run from day one regardless of team size.

## 1. Governance, ownership & change control

- **GOV-01** Every repo is **public** in the `getdue-dev` org and ships a `SECURITY.md` referencing this document.
- **GOV-02** Default branch protected: **PR-only from a feature branch**, no direct pushes, no force-push, all CI
  gates green before merge ([01 §7](./01-repositories.md#7-branch-protection-all-repos)). Solo phase: the maintainer
  self-merges once CI is green.
- **GOV-03** Every production change is traceable to a **PR with passing CI** and a ticket reference in the commit/PR
  body — an auditable change record.
- **GOV-04 (SHOULD)** Periodic access review of repo, cloud, and database grants.

> **Two operating modes.** Governance here runs in **Solo-phase (now)** mode: best-effort, single maintainer who
> self-merges once CI is green (GOV-02). The full **reviewer / separation-of-duties** model — required independent
> approvals, environment approvals, segregated merge authority — is the **Target operating model (team)**, adopted
> when a team forms; it is not configured now.

## 2. Secure SDLC & supply chain

Every service pipeline ([01 §6](./01-repositories.md#6-cicd-per-service-repo)) MUST run these gates and **MUST fail
the build** on threshold breach:

| Gate | Tool (example) | Blocking threshold |
|---|---|---|
| **SAST** | Semgrep | any High/Critical |
| **Secret scanning** | gitleaks | any verified secret |
| **SCA (dependencies)** | OWASP Dependency-Check | Critical, or High past SLA |
| **IaC scanning** | tfsec / Checkov | any High misconfig |
| **Container scanning** | Trivy / Grype | Critical OS/lib CVE |
| **License compliance** | SCA | disallowed license |
| **Architecture tests** | NetArchTest | any violation (incl. money-movement / outbound-finance guard) |
| **Code coverage (backend .NET)** | Coverlet | **blocking** — `Domain`+`Application` < 100% line OR branch fails ([03 · Testing](./03-testing-standard.md)) |
| **Code coverage (web / mobile / infra)** | Vitest / xccov / IaC tooling | **non-blocking** — ≥80% line target, measured & reported on every PR ([03 · Testing](./03-testing-standard.md)) |
| **Mutation testing (backend .NET)** | Stryker | **blocking on backend** — mutation score < 85% (no surviving mutants in money/authz/tenant code); not applied to web/mobile/infra |

- **SDLC-01** Container images are **minimal** (distroless/chiseled), run as **non-root**, read-only filesystem, no
  shell where avoidable.
- **SDLC-02** Images are **signed (cosign)** and ship an **SBOM**; deploy admission **rejects unsigned images**.
- **SDLC-03** **Provenance attestation (SLSA)** for build artifacts; dependencies **pinned** (no floating ranges).
- **SDLC-04** **DAST** against staging before promotion to production for any externally-reachable surface.
- **SDLC-05 (SHOULD)** A short **per-service threat-model template/stub** is **maintained as a checked-in artifact**
  for each service (assets, trust boundaries, entry points, top threats + mitigations) and is **reviewed and updated
  whenever the trust boundary changes** (new external surface, new integration, new data class).

## 3. Dependency updates & patching

Base images and dependencies are patched continuously, with a nightly rescan of running images. The dependency-update
bot ([01 §5](./01-repositories.md#5-versioning--dependency-flow)) opens upgrade PRs; each is gated by the same CI
pipeline before merge. Remediation **SLAs by severity** are defined in the application security standard
([09 §10](../phase-0/09-security-standard.md#10-vulnerability--patch-management-remediation-slas)).

**Secret/key rotation** is owned by the application security standard — see **SEC-SEC-04** (app secrets ≤90d; JWT
signing keys ≤180d with JWKS overlap; DB credentials ≤30d auto-rotated; TLS certs auto via cert-manager) in
[09 §security & secrets](../phase-0/09-security-standard.md).
