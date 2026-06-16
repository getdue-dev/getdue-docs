# Engineering Handbook

Engineering process, tooling, and conventions for GetDue — **separated from the product/business docs** in
[`../phase-0/`](../phase-0/README.md). This material is not tied to a single phase; it carries forward unchanged into
later phases.

## GitHub Flow (the whole workflow)

The repo workflow is deliberately minimal:

- **All repositories are private** in the `get-due-dev` org.
- **`main` is protected:** branch off `main` → open a **pull request** → **CI must be green** → merge. **No direct
  pushes, no force-push.** Solo phase: the maintainer self-merges once CI is green.

That is it. No CODEOWNERS, signed-commit, ruleset, or environment-approval machinery is configured — those are an
organisational decision for if/when a team forms, and are out of scope here.

## Documents

| Doc | Covers |
|---|---|
| [01 · Repositories & Contracts](./01-repositories.md) | Polyrepo strategy, repo inventory, contracts/buildingblocks, CI/CD pipeline, branch protection, local dev |
| [02 · Versioning System](./02-versioning.md) | SemVer across APIs, services, contracts, events, DB migrations, infra |
| [03 · Testing Standard](./03-testing-standard.md) | 100% coverage, mutation testing, architecture/guardrail tests |
| [04 · Secure SDLC](./04-secure-sdlc.md) | Change-control governance + CI security gates + supply chain |

## Relationship to phase-0

The **application/runtime security** controls (auth, network, data, secrets, logging, incident response) live in the
product docs: [phase-0/09 · Security Standard](../phase-0/09-security-standard.md). This handbook covers only the
**build-and-ship process** around them.
