---
name: create-new-repo
description: Create a new GetDue repository that follows the polyrepo conventions in engineering/01-repositories.md — public repo in the getdue-dev org, protected main branch, SECURITY.md, and the shared reusable CI workflow. Use when the user wants to spin up a new service/client/shared/infra repo (e.g. "/create-new-repo getdue-stocks", "create the goals service repo").
---

# Create New Repo

Create and configure a new repository in the GetDue polyrepo, following
[engineering/01-repositories.md](../../../engineering/01-repositories.md). Every repo
is **public** in the `getdue-dev` org, has a **protected `main`**, ships a
`SECURITY.md`, and consumes the **shared reusable CI workflow** from the org `.github`
repo.

## Inputs

The user passes (or you infer) three things:

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
gh auth status                                    # gh must be authenticated
gh api orgs/getdue-dev >/dev/null 2>&1 && echo OK # org access sanity check
```

Do **not** probe a specific repo as the canary — when bootstrapping the very first
repos in the org, none may exist yet, and you may end up probing the repo you're
about to create. `gh api orgs/getdue-dev` confirms org access without that ambiguity.

Then confirm the target repo does not already exist:

```bash
gh repo view getdue-dev/<name> >/dev/null 2>&1 && echo "EXISTS" || echo "OK"
```

If it prints `EXISTS`, STOP — do not overwrite an existing repo.

Finally, probe whether the org-shared reusable CI workflow is in place. This decides
whether step 2 scaffolds `ci.yml` and whether step 5's branch protection requires it
as a status check:

```bash
gh api repos/getdue-dev/.github/contents/.github/workflows >/dev/null 2>&1 \
  && echo "REUSABLE_WORKFLOWS_PRESENT" || echo "NO_REUSABLE_WORKFLOWS"
```

If the call 404s, the `.github` repo (or its workflows directory) doesn't exist yet
— this is normal during early bootstrap. Carry that state into steps 2 and 5.

## Procedure

Run in order. STOP on any failure rather than leaving a half-configured repo.

### 1. Create the public repo

Create it with an initial commit (`--add-readme`) so `main` exists from the start —
the commit step relies on the `commit-feature` skill, which branches off an existing
`main`. The scaffold README in step 2 overwrites this placeholder.

```bash
gh repo create getdue-dev/<name> --public --add-readme --description "<one-line purpose>"
```

Enable **automatically delete head branches** so merged feature branches are cleaned
up — this keeps the repo aligned with the `commit-feature` flow (branch → PR → merge):

```bash
gh api -X PATCH repos/getdue-dev/<name> -F delete_branch_on_merge=true
```

### 2. Clone and scaffold

Clone into the **clone location the user gave you** (see Inputs §3) — referred to below
as `<clone-dir>`. Confirm the path doesn't already exist before cloning.

```bash
gh repo clone getdue-dev/<name> <clone-dir>
```

Add the baseline files every repo must carry:

- **`README.md`** — repo name, type, and what it publishes (from the §2 inventory).
- **`SECURITY.md`** — required in **every** repo; point it at the secure-SDLC doc.
  Minimal content:

  ```markdown
  # Security Policy

  Security practices for this repository follow GetDue's
  [Secure SDLC](https://github.com/getdue-dev/getdue-docs/blob/main/engineering/04-secure-sdlc.md).

  Report vulnerabilities privately to the maintainer — do not open a public issue.
  ```

- **`.github/CODEOWNERS`** — make the maintainer the default owner so they are
  **auto-requested as a reviewer** on every PR. In the solo phase this is a *request,
  not a gate*: keep `require_code_owner_reviews: false` in step 5 (GitHub won't let you
  approve your own PR, so requiring it would block self-merge).

  ```
  # Default owner — auto-requested for review on all PRs
  *       @paulbuzakov
  ```

- **`.github/workflows/ci.yml`** — only scaffold this if the preconditions probe
  reported `REUSABLE_WORKFLOWS_PRESENT`. The shared workflow lives in
  `getdue-dev/.github` and runs build → tests → coverage/mutation gates →
  architecture tests → SAST/secret/SCA/IaC/container scans → SBOM + cosign sign →
  publish image → bump tag in `getdue-deploy`.

  ```yaml
  name: CI
  on:
    push:
      branches: [main]
    pull_request:
      branches: [main]
  jobs:
    ci:
      uses: getdue-dev/.github/.github/workflows/<reusable-workflow>.yml@main
      secrets: inherit
  ```

  Pick the reusable workflow that matches the repo **type** (service image pipeline,
  client build, package publish, etc.). Inspect the `.github` repo to confirm the
  filename: `gh api repos/getdue-dev/.github/contents/.github/workflows`.

  **If preconditions reported `NO_REUSABLE_WORKFLOWS`:** do **not** write `ci.yml` —
  pointing at a workflow that doesn't exist will fail every PR check and (combined
  with step 5's required-status-checks rule) wedge the repo. Instead, add a one-line
  CI section to the README noting that CI will be wired up once `getdue-dev/.github`
  ships its reusable workflows, and disable the `required_status_checks` rule in step 5
  by sending `"required_status_checks": null` instead of the `contexts` block.

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

**Working directory matters.** `commit-feature` runs bare `git`/`gh` commands that
follow the shell CWD, not the parent skill's CWD. Before invoking it, either:

- `cd <clone-dir>` and run the skill, or
- run its procedure inline using `git -C <clone-dir>` for every step and a single
  `cd <clone-dir>` for the `gh pr create` call.

The parent CWD (the docs repo you're running this skill from) is the wrong target —
mis-targeting will branch and commit inside `getdue-docs` instead of the new repo.

Pass it a description so the branch/commit/PR read sensibly, e.g.
`chore: bootstrap repo with CI, SECURITY.md, README`.

This replaces the old manual feature-branch + `gh pr create` steps — `commit-feature`
handles both the commit/push and the PR.

### 5. Protect the default branch

Apply full GitHub Flow protection on `main`: work lands only through a PR from a
feature branch, CI must be green and the branch up to date, history stays linear, all
review threads are resolved, and the branch can never be force-pushed or deleted.

> **Free-plan note.** The `getdue-dev` org is on the **GitHub Free** plan, where the
> classic branch-protection API works on **public** repos only — for a **private** repo
> it returns **403** (`Upgrade to GitHub Pro/Team/Enterprise...`). All GetDue repos are
> **public**, so this call works on the free plan with no upgrade needed. (If a repo is
> ever made private, this step will 403 until the org moves to a paid plan.)

The payload below assumes the reusable CI workflow exists (preconditions reported
`REUSABLE_WORKFLOWS_PRESENT`). **If preconditions reported `NO_REUSABLE_WORKFLOWS`**,
no `ci` check exists yet — replace the entire `"required_status_checks"` object with
`"required_status_checks": null`, otherwise the rule waits forever on a check that
never runs and wedges the repo. Re-run this step with the `contexts` block once the
workflow ships.

```bash
gh api -X PUT repos/getdue-dev/<name>/branches/main/protection \
  --input - <<'JSON'
{
  "required_status_checks": {
    "strict": true,
    "contexts": ["ci"]
  },
  "enforce_admins": false,
  "required_pull_request_reviews": {
    "required_approving_review_count": 0,
    "dismiss_stale_reviews": true,
    "require_code_owner_reviews": false,
    "require_last_push_approval": false
  },
  "restrictions": null,
  "required_linear_history": true,
  "required_conversation_resolution": true,
  "allow_force_pushes": false,
  "allow_deletions": false,
  "block_creations": false,
  "lock_branch": false,
  "allow_fork_syncing": false
}
JSON
```

Then **require signed commits**. This is a *separate sub-resource* in the classic API
(not a field in the payload above) — enable it with a `POST`:

```bash
gh api -X POST repos/getdue-dev/<name>/branches/main/protection/required_signatures
```

> **Prerequisite — local commit signing.** Required signed commits means every commit
> that lands on `main` must carry a **verified** signature. The maintainer must have
> signing configured (`git config commit.gpgsign true` plus a GPG/SSH signing key
> **registered with GitHub**) — check with `git config --get commit.gpgsign` and
> `git config --get user.signingkey`. If signing isn't set up, STOP and tell the user:
> `commit-feature`'s commits would be unsigned and the PR couldn't merge. (PRs merged
> through the GitHub UI are re-signed by GitHub, but the feature-branch commits still
> need a valid signature to pass the rule.)

What each field enforces:
- **`required_status_checks.strict: true`** — the feature branch must be **up to date
  with `main`** before merge (no stale merges); **`contexts: ["ci"]`** — the CI check
  must be green. The context must match the job name from step 2's workflow.
- **`required_pull_request_reviews`** present (even with `0` approvals) is what makes a
  **PR mandatory** — it blocks direct pushes to `main`. `dismiss_stale_reviews`
  dismisses existing approvals when new commits land, so a freshly pushed change can't
  ride in on a stale approval. `require_code_owner_reviews` stays
  **off**: the `CODEOWNERS` file (step 2) auto-requests the maintainer as a reviewer,
  but in the solo phase that's a request only — turning it into a required gate would
  block self-merge (you can't approve your own PR). `require_last_push_approval` stays
  off for the same reason.
- **`required_linear_history: true`** — merges must be squash/rebase, keeping a clean
  linear history (matches the `commit-feature` flow).
- **`required_conversation_resolution: true`** — every PR comment thread must be
  resolved before merge.
- **`allow_force_pushes: false` / `allow_deletions: false`** — `main` can't be
  force-pushed or deleted. The remaining toggles stay at their non-restrictive
  defaults: **`lock_branch: false`** (branch isn't read-only — normal PR merges still
  land), **`block_creations: false`**, and **`allow_fork_syncing: false`**. They're
  listed explicitly so the protection is fully pinned rather than left to API defaults.
- **`required_signatures`** (the POST above) — every commit on `main` must be
  cryptographically signed and verified.

Notes:
- In the **solo phase**, `required_approving_review_count` is `0` and `enforce_admins`
  is `false` so the maintainer can self-merge once CI is green. Heavier controls
  (multiple reviewers, `enforce_admins: true`, separation of duties) are a future team
  decision — do not add them now.

### 6. Report

Give the user:
- the repo URL,
- the type and what it publishes,
- confirmation that `main` is protected and `SECURITY.md` + CI are in place,
- the bootstrap PR link (from `commit-feature`).

## Notes

- Always make the repo public — all GetDue repos are public in `getdue-dev` (this is
  what lets branch protection work on the free plan).
- Never configure floating dependency ranges or local-path contract deps — only
  pinned, published, tagged versions.
- Branch protection (step 5) needs **org-admin rights**. On the GitHub Free plan it
  works for **public** repos (which all GetDue repos are); it would 403 only for a
  private repo. If admin rights are missing, complete the scaffold and STOP at step 5,
  telling the user to grant rights; the call then succeeds unchanged.
- This skill does **not** wire up the dependency-update bot or consumer-driven contract
  tests beyond the reusable CI — those are configured per the spec's §5 once the repo's
  dependencies exist.
