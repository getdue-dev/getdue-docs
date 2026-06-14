---
name: commit-changes
description: Commit the current working-tree changes from a feature branch with a clear conventional-commit message. Use when the user says "commit", "commit my changes", "commit this", or wants to save work-in-progress on a branch. Ensures work is never committed directly to the default branch.
---

# Commit changes (from a feature branch)

Commit the current working-tree changes safely and with a well-formed message. **Never commit directly to the
default branch** — branch first if needed.

## Steps

1. **Inspect what's changing.** Run, in parallel:
   - `git status --short`
   - `git diff --stat` and `git diff` (staged + unstaged) to understand the actual change
   - `git rev-parse --abbrev-ref HEAD` (current branch)
   - `git log --oneline -5` (recent commit style to match)

   If there are **no changes**, stop and tell the user there's nothing to commit.

2. **Guard the default branch.** Determine the default branch (usually `main` or `master`:
   `git symbolic-ref refs/remotes/origin/HEAD` or fall back to `main`). If the current branch **is** the default
   branch, **create a feature branch first**:
   - Derive a short, kebab-case name from the change (e.g. `feat/loan-schedule`, `docs/api-versioning`,
     `fix/fx-rounding`). Pick a conventional prefix (`feat/`, `fix/`, `docs/`, `chore/`, `refactor/`, `test/`).
   - `git checkout -b <name>`
   - Tell the user which branch you created and why.

   If already on a feature branch, stay on it.

3. **Stage the changes.** `git add -A` (or stage only the files the user named, if they scoped it). Do **not** stage
   files matched by `.gitignore`; never add secrets, `.env`, credentials, or large binaries — if you see one staged,
   stop and ask.

4. **Write the commit message.** Use Conventional Commits:
   - Subject: `<type>(<scope>): <imperative summary>` — ≤ ~72 chars, no trailing period.
   - Body (when the change isn't trivial): a few bullets on *what* and *why*, wrapped ~72 cols.
   - Match the tone/format of recent commits from step 1.

5. **Commit.** Use a single `git commit -m "$(cat <<'EOF' ... EOF)"` heredoc so the multi-line message and trailer
   are preserved exactly.

6. **Report.** Print the resulting branch name, commit hash, and `git status` (should be clean). **Do not push**
   unless the user asked — offer it as a next step (`git push -u origin <branch>`), along with opening a PR.

## Rules

- One logical change per commit; if the working tree mixes unrelated changes, point that out and offer to split.
- Never use `git add -i`, `git rebase -i`, or other interactive git (unsupported here).
- Never `--force`, never amend or rewrite already-pushed commits unless explicitly told to.
- If a pre-commit hook fails, surface the output and fix the cause — don't bypass with `--no-verify` unless the user
  says so.
