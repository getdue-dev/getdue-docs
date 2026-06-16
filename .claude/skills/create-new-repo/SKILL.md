---
name: create-new-repo
description: Create a new GetDue repository that follows the polyrepo conventions in engineering/01-repositories.md — private repo in the get-due-dev org, protected main branch, SECURITY.md, and the shared reusable CI workflow. Use when the user wants to spin up a new service/client/shared/infra repo (e.g. "/create-new-repo getdue-stocks", "create the goals service repo").
---

# Create New Repo

Create and configure a new repository in the GetDue polyrepo, following
[engineering/01-repositories.md](../../../engineering/01-repositories.md). Every repo
is **private** in the `get-due-dev` org, has a **protected `main`**, ships a
`SECURITY.md`, and consumes the **shared reusable CI workflow** from the org `.github`
repo.

## Inputs

The user passes (or you infer) two things:

1. **Repo name** — must be `getdue-<name>` kebab-case (e.g. `getdue-stocks`). The
   `.github` org-config repo is the one exception to the prefix.
2. **Repo type** — one of: `Service`, `Edge`, `Client`, `Shared`, `Infra`, `GitOps`,
   `Org config`. This drives what the repo publishes and which scaffolding it gets
   (see the inventory table in §2 of the spec). If the type is ambiguous, ask the
   user before proceeding.

3. **Clone location** — the local directory where the repo should be cloned and
   scaffolded. **Always ask the user where to save the repo** before cloning — do not
   assume `/tmp` or the current working directory. Suggest a sensible default (e.g. a
   sibling of the other GetDue repos), but let the user confirm or override it.

If the name doesn't match `getdue-*` (and isn't `.github`), confirm with the user
before continuing.

## Preconditions

Run these checks first. If any fails, STOP and tell the user what's missing.

```bash
gh auth status          # gh must be authenticated
gh repo view get-due-dev/getdue-contracts >/dev/null 2>&1; echo $?   # org access sanity check
```

Also confirm the repo does not already exist:

```bash
gh repo view get-due-dev/<name> >/dev/null 2>&1 && echo "EXISTS" || echo "OK"
```

If it prints `EXISTS`, STOP — do not overwrite an existing repo.

## Procedure

Run in order. STOP on any failure rather than leaving a half-configured repo.

### 1. Create the private repo

Create it with an initial commit (`--add-readme`) so `main` exists from the start —
the commit step relies on the `commit-feature` skill, which branches off an existing
`main`. The scaffold README in step 2 overwrites this placeholder.

```bash
gh repo create get-due-dev/<name> --private --add-readme --description "<one-line purpose>"
```

### 2. Clone and scaffold

Clone into the **clone location the user gave you** (see Inputs §3) — referred to below
as `<clone-dir>`. Confirm the path doesn't already exist before cloning.

```bash
gh repo clone get-due-dev/<name> <clone-dir>
```

Add the baseline files every repo must carry:

- **`README.md`** — repo name, type, and what it publishes (from the §2 inventory).
- **`SECURITY.md`** — required in **every** repo; point it at the secure-SDLC doc.
  Minimal content:

  ```markdown
  # Security Policy

  Security practices for this repository follow GetDue's
  [Secure SDLC](https://github.com/get-due-dev/getdue-docs/blob/main/engineering/04-secure-sdlc.md).

  Report vulnerabilities privately to the maintainer — do not open a public issue.
  ```

- **`.github/workflows/ci.yml`** — call the org reusable workflow rather than
  redefining the pipeline. The shared workflow lives in `get-due-dev/.github` and runs
  build → tests → coverage/mutation gates → architecture tests → SAST/secret/SCA/IaC/
  container scans → SBOM + cosign sign → publish image → bump tag in `getdue-deploy`.

  ```yaml
  name: CI
  on:
    push:
      branches: [main]
    pull_request:
      branches: [main]
  jobs:
    ci:
      uses: get-due-dev/.github/.github/workflows/<reusable-workflow>.yml@main
      secrets: inherit
  ```

  Pick the reusable workflow that matches the repo **type** (service image pipeline,
  client build, package publish, etc.). If you don't know the exact workflow filename,
  inspect the `.github` repo: `gh api repos/get-due-dev/.github/contents/.github/workflows`.

### 3. Type-specific notes

- **Service / Edge** — publishes a **container image**; CI bumps the image tag in
  `getdue-deploy` (GitOps). Depends only on **published, tagged** `getdue-contracts`
  and `getdue-buildingblocks` versions (pinned, no floating ranges, no local paths).
- **Client** (`getdue-web` / `getdue-mobile`) — consumes `@getdue/contracts` (npm) or
  the `GetDueContracts` Swift package; web publishes an image, mobile an App Store build.
- **Shared** (`getdue-contracts` / `getdue-buildingblocks`) — publishes versioned
  packages with **SemVer**; a breaking change needs a **major bump + ADR**, and the
  schema-diff/SemVer gate must be in CI.
- **Infra / GitOps / Org config** — no app image; `getdue-platform` ships tagged Helm/
  Terraform, `getdue-deploy` holds declarative Argo CD state, `.github` holds the
  reusable workflows.

### 4. Commit, push, and open the PR via `commit-feature`

Don't hand-roll the git commands — invoke the **`commit-feature`** skill from inside
the scaffold clone (`<clone-dir>`). It branches off `main`, commits with the required
`Co-Authored-By` trailer, pushes, opens the PR with the
`🤖 Generated with [Claude Code]` footer, and returns to a clean `main`.

Pass it a description so the branch/commit/PR read sensibly, e.g.
`chore: bootstrap repo with CI, SECURITY.md, README`.

This replaces the old manual feature-branch + `gh pr create` steps — `commit-feature`
handles both the commit/push and the PR.

### 5. Protect the default branch

Apply GitHub Flow protection on `main`: no direct pushes, PR required, required status
checks green, no force-push.

```bash
gh api -X PUT repos/get-due-dev/<name>/branches/main/protection \
  --input - <<'JSON'
{
  "required_status_checks": { "strict": true, "contexts": ["ci"] },
  "enforce_admins": false,
  "required_pull_request_reviews": { "required_approving_review_count": 0 },
  "restrictions": null,
  "allow_force_pushes": false,
  "allow_deletions": false
}
JSON
```

Notes:
- In the **solo phase**, `required_approving_review_count` is `0` and `enforce_admins`
  is `false` so the maintainer can self-merge once CI is green. Heavier controls
  (multiple reviewers, separation of duties) are a future team decision — do not add
  them now.
- The status check context (`ci`) must match the job name from step 2's workflow.

### 6. Report

Give the user:
- the repo URL,
- the type and what it publishes,
- confirmation that `main` is protected and `SECURITY.md` + CI are in place,
- the bootstrap PR link (from `commit-feature`).

## Notes

- Never make the repo public — all GetDue repos are private in `get-due-dev`.
- Never configure floating dependency ranges or local-path contract deps — only
  pinned, published, tagged versions.
- If `gh` lacks org-admin rights to set branch protection, complete the scaffold and
  STOP at step 5, telling the user to apply protection (or grant rights).
- This skill does **not** wire up the dependency-update bot or consumer-driven contract
  tests beyond the reusable CI — those are configured per the spec's §5 once the repo's
  dependencies exist.
