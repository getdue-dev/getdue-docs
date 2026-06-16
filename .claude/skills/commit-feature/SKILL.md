---
name: commit-feature
description: Take the current working changes, create a fresh feature branch off main, commit, push, open a PR, and return to a clean main. Use when the user wants to ship the current changes as a PR in one step (e.g. "/commit-feature", "commit this as a feature", "open a PR for these changes").
---

# Commit Feature

Ship the current working-tree changes as a pull request, then leave the repo on a
clean `main`. The feature branch always starts from an up-to-date `main`, never from
whatever branch you happen to be on.

## Inputs

The user may pass an optional argument: a short description of the change. Use it for
the branch name and as the basis for the commit message / PR title. If no argument is
given, derive both from the actual diff.

## Procedure

Run these steps in order. If any step fails, STOP and report the failure — do not
continue and do not leave the repo in a half-migrated state.

### 0. Switch to main and pull the latest changes

Before inspecting the working tree, make sure `main` is checked out and up to date
with the remote. Any uncommitted changes follow the checkout and are handled by the
stash step below.

```bash
git checkout main
git pull --ff-only origin main
```

If `git checkout main` fails because uncommitted changes would be overwritten, STOP
and report it. If `git pull --ff-only` fails because local `main` diverged from
origin, STOP and report it rather than merging or rebasing automatically.

### 1. Confirm there is something to ship

```bash
git status --porcelain
```

If the output is empty there are no changes. Check whether the current branch already
has unpushed commits (`git status -sb`). If there is genuinely nothing to commit and
nothing unpushed, tell the user there is nothing to ship and STOP.

### 2. Stash the working changes

This makes the rest of the flow identical whether the user is on `main` or another
branch. Include untracked files.

```bash
git stash push --include-untracked -m "commit-feature wip"
```

If git reports "No local changes to save", there were no uncommitted changes (the work
may already be committed on the current branch) — note that and continue without a
stash to pop in step 5.

### 3. Confirm you are still on an up-to-date main

Step 0 already checked out `main` and pulled. After the stash, you should still be on
`main` with a clean tree. Sanity-check before branching:

```bash
git rev-parse --abbrev-ref HEAD   # expect: main
git status --porcelain            # expect: empty
```

If either check fails, STOP and report it.

### 4. Create the feature branch from main

Pick a short, kebab-case branch name. Prefer a conventional prefix that matches the
change type — `feat/`, `fix/`, `docs/`, `chore/`, etc. Derive the slug from the user's
argument or the diff.

```bash
git checkout -b <prefix>/<slug>
```

### 5. Restore the stashed changes onto the feature branch

Only if step 2 created a stash:

```bash
git stash pop
```

If `git stash pop` reports a conflict, STOP and report it — the user must resolve it.

### 6. Commit all changes

```bash
git add -A
git commit -m "<message>"
```

Keep the message short. Prefer a single Conventional-Commits subject line
(`type: summary`, ~50 chars, imperative mood) with no body. Only add a body if the
change genuinely needs it, and then keep it to one or two short lines — never a long
description. End the commit message with the required trailer:

```
Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
```

### 7. Push and open the PR

```bash
git push -u origin <prefix>/<slug>
gh pr create --base main --head <prefix>/<slug> --title "<title>" --body "<body>"
```

Write a concise PR body summarizing what changed and why. End it with:

```
🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

Capture the PR URL that `gh pr create` prints.

### 8. Return to a clean main and clean up

```bash
git checkout main
git branch -D <prefix>/<slug>
```

The work now lives entirely in the remote feature branch / PR, so deleting the local
branch is safe.

### 9. Report

Give the user the PR link as the headline of your reply, plus a one-line summary of the
branch name and commit message used.

## Notes

- Never force-push and never delete the remote branch — the PR depends on it.
- If the repo has no `origin` remote or `gh` is not authenticated, do the commit steps
  but STOP before pushing and tell the user what is missing.
