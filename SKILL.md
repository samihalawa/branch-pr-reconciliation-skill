---
name: branch-pr-reconciliation-skill
description: "Reconcile a repo's open PRs, remote branches, and local branches by reading each diff against the current codebase, manually porting only the still-useful changes, then closing stale PRs and cleaning up branches without merging or cherry-picking. Use when the user wants branch and PR cleanup without losing worthwhile work."
---

# Branch PR Reconciliation Skill

Use this skill when the user wants to clean up branch and PR sprawl without merging old work blindly.

The job is:

1. inspect every relevant PR and branch
2. compare each one against the current codebase
3. manually implement the useful parts on the current mainline
4. close obsolete PRs
5. delete stale branches only after the useful work is preserved or confirmed obsolete

This skill is instructions only. Do not build helper tooling unless the user explicitly asks for it.

## Hard Rules

- Never merge old branches directly.
- Never cherry-pick old commits as the integration method.
- Never assume a PR is useful because the title sounds useful.
- Always compare old work against the current base branch and current files first.
- Treat the current verified codebase as the source of truth, not the old branch narrative.
- Port useful ideas manually so the final implementation fits the current codebase.
- If an old change conflicts with current architecture, adapt it or discard it.
- Close PRs only after you understand whether anything in them still matters.
- Delete branches only after the useful work is preserved or clearly obsolete.
- Do not delete the user's current branch or any branch with clearly active ongoing work unless the user asked for aggressive cleanup and the branch is proven redundant.

## Workflow

### 1. Inspect Current Repo State

- Work in the real local repo.
- Read the repo context first:
  - `tree -I node_modules -L 2`
  - `git status --short`
  - `git branch --all`
  - `git remote -v`
  - `git log --oneline --decorate -20`
- Fetch the latest remote state:
  - `git fetch --all --prune`
- List the open PRs and their head branches:
  - `gh pr list --state open`
- If useful, also inspect closed PRs that still have unmerged branches or suspicious overlap.

### 2. Build The Reconciliation Ledger

For each open PR or suspicious branch, record:

- PR number, title, author, head branch, base branch
- whether the branch still exists on remote and locally
- changed files
- whether the same area has already been modified on current `main`
- first impression:
  - `candidate`
  - `duplicate`
  - `obsolete`
  - `needs-manual-port`

### 3. Review Every PR And Branch Properly

For each PR:

- Read the PR body and commit list.
- Read the actual diff against current base, not just the PR summary:
  - `gh pr view <number> --comments --files`
  - `git diff origin/main...origin/<head-branch> -- <relevant-files>`
- Compare the old change with the current code in `main`.
- Decide which parts are:
  - still valuable
  - already implemented elsewhere
  - outdated
  - harmful or incompatible

For remote branches without an open PR:

- inspect the branch tip and diff against `origin/main`
- review commits and changed files
- apply the same classification

### 4. Manually Port Only The Useful Parts

- Re-implement useful behavior directly on the current branch.
- Follow the current codebase patterns, naming, architecture, validation, and UX.
- Do not copy obsolete code just to preserve history.
- Do not revive hacks that were later fixed another way.
- If a PR contains one useful idea buried in messy code, port only that idea.
- If a PR is fully obsolete because the current codebase already solved it better, do not port anything.

### 5. Verify The Ported Work

After each meaningful manual port:

- run the fastest real verification for the affected area
- check type errors, lint or tests if relevant
- inspect runtime behavior when the change is user-facing
- verify no regression was introduced by adapting old ideas to current code

Do not close the source PR until the useful changes are either:

- manually ported and verified, or
- proven obsolete, duplicate, or invalid

### 6. Close PRs Deliberately

When a PR is no longer needed:

- close it explicitly
- add a short closing note when helpful:
  - useful parts were manually ported to current `main`
  - or the PR is obsolete because current `main` already superseded it

Do not merge just to make the PR disappear.

### 7. Clean Up Branches

After PR decisions are complete:

- delete stale remote branches that are closed, obsolete, or fully superseded
- prune local tracking refs
- delete matching stale local branches that are no longer needed
- keep any branch that still has unresolved active work

Preferred order:

1. port useful changes manually
2. verify
3. close PR
4. delete remote branch if appropriate
5. delete local branch if appropriate

### 8. Finish Cleanly

- commit the manually integrated work as a fresh commit on the current branch
- push the final branch
- confirm which PRs were closed
- confirm which branches were deleted
- confirm which branches were intentionally kept

## Decision Standard

Keep and manually port something only if it still improves the current codebase.

Discard it if any of these are true:

- current `main` already includes the same behavior
- the old implementation no longer fits the architecture
- the change solves a problem that no longer exists
- the branch is mostly noise with no clear surviving value

If you are unsure, inspect deeper before deleting. Cleanup is only successful when worthwhile work is preserved and dead branch clutter is removed.
