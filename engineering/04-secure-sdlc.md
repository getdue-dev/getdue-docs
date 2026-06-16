# 04 · Secure SDLC, Governance & Supply Chain

The engineering-process companion to the application **[Security Standard](../phase-0/09-security-standard.md)**.
This document owns **change-control governance** and the **CI security pipeline**; the phase-0 standard owns the
runtime/application controls (auth, network, data, logging, IR).

## 0. Solo-phase scope

Phase 0 is solo-maintained in the private `get-due-dev` org. Governance is intentionally light — **GitHub Flow** on
**private repos + a protected `main`** ([01 §7](./01-repositories.md#7-branch-protection-all-repos)). The CI security
gates below run from day one regardless of team size.

## 1. Governance, ownership & change control

- **GOV-01** Every repo is **private** in the `get-due-dev` org and ships a `SECURITY.md` referencing this document.
- **GOV-02** Default branch protected: **PR-only from a feature branch**, no direct pushes, no force-push, all CI
  gates green before merge ([01 §7](./01-repositories.md#7-branch-protection-all-repos)). Solo phase: the maintainer
  self-merges once CI is green.
- **GOV-03** Every production change is traceable to a **PR with passing CI** and a ticket reference in the commit/PR
  body — an auditable change record.
- **GOV-04 (SHOULD)** Periodic access review of repo, cloud, and database grants.

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
| **Code coverage** | Coverlet / Vitest / xccov | **< 100% line OR branch** ([03 · Testing](./03-testing-standard.md)) |
| **Mutation testing** | Stryker | mutation score < 85% (no surviving mutants in money/authz/tenant code) |

- **SDLC-01** Container images are **minimal** (distroless/chiseled), run as **non-root**, read-only filesystem, no
  shell where avoidable.
- **SDLC-02** Images are **signed (cosign)** and ship an **SBOM**; deploy admission **rejects unsigned images**.
- **SDLC-03** **Provenance attestation (SLSA)** for build artifacts; dependencies **pinned** (no floating ranges).
- **SDLC-04** **DAST** against staging before promotion to production for any externally-reachable surface.
- **SDLC-05 (SHOULD)** A **threat model** is maintained per service; reviewed when the trust boundary changes.

## 3. Dependency updates & patching

Base images and dependencies are patched continuously, with a nightly rescan of running images. The dependency-update
bot ([01 §5](./01-repositories.md#5-versioning--dependency-flow)) opens upgrade PRs; each is gated by the same CI
pipeline before merge. Remediation **SLAs by severity** are defined in the application security standard
([09 §10](../phase-0/09-security-standard.md#10-vulnerability--patch-management-remediation-slas)).
