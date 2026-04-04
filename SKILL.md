---
name: branch-pr-reconciliation-skill
description: "Reconcile a repo's open PRs, remote branches, local branches, worktrees, and stashes by reading each diff against the current codebase, manually porting only the still-useful changes, then closing stale PRs and cleaning up all artifacts without merging or cherry-picking (except clean fast-forwards). Use when the user wants branch/PR cleanup without losing worthwhile work."
---

# Branch PR Reconciliation Skill

Use this skill when the user wants to clean up branch, PR, and worktree sprawl without merging old work blindly.

The job is:

1. fully reconcile branch, PR, worktree, stash, and duplicate-work state first
2. compare each relevant branch against the current codebase
3. manually implement the useful parts on the current mainline
4. close obsolete PRs
5. delete stale branches and worktrees only after the useful work is preserved or confirmed obsolete
6. sync local and remote so everything is clean

This skill is instructions only. Do not build helper tooling unless the user explicitly asks for it.

## Hard Rules

- Reconciliation comes before implementation. No code edits before the reconciliation pass is complete.
- Never merge old branches directly. The ONE exception: a branch whose tip is a direct descendant of current main HEAD (verified with `git merge-base --is-ancestor main <branch>`) may be merged with `--ff-only`.
- Never cherry-pick old commits as the integration method unless the cherry-pick applies with zero conflicts AND the result passes type-check.
- Never assume a PR is useful because the title sounds useful.
- Always compare old work against the current base branch and current files first.
- Treat the current verified codebase as the source of truth, not the old branch narrative.
- Port useful ideas manually so the final implementation fits the current codebase.
- If an old change conflicts with current architecture, adapt it or discard it.
- Close PRs only after you understand whether anything in them still matters.
- Delete branches only after the useful work is preserved or clearly obsolete.
- Do not delete the user's current branch or any branch with clearly active ongoing work unless the user asked for aggressive cleanup and the branch is proven redundant.
- `git diff --stat` alone does not count as reconciliation.
- UI fixes, app edits, or build runs before branch and worktree reconciliation mean the procedure is mis-sequenced.
- Never force-push to main.
- Never use `git branch -D` (force delete) unless safe-delete `-d` fails AND you have confirmed the work is captured or valueless.

## Required First Pass

Before touching application code, complete and record all of this:

1. branch inventory (local + remote)
2. worktree inventory (paths, branches, status)
3. stash inventory
4. local uncommitted work inventory (in main worktree AND all linked worktrees)
5. branch graph and recent commit inventory
6. open PR inventory (with comments, reviews, and review feedback)
7. closed/merged PR inventory (last 90 days)
8. branch-to-main diff review for all relevant branches
9. remote orphan detection (tracking refs with no remote, branches with no local)
10. duplicate and overlap detection
11. explicit keep, port, close, delete, or defer decision per branch or PR

Do not start implementation until this pass is complete.

## Workflow

### 1. Discover All State

Work in the real local repo. Read everything:

```bash
# Repo structure
tree -I node_modules -L 2

# Git state
git status --short --branch
git worktree list
git branch --all --verbose --no-abbrev
git remote -v
git stash list
git log --oneline --decorate --graph -30 --all

# Sync remote refs (read-only)
git fetch --all --prune

# PRs
gh pr list --state open
gh pr list --state closed --limit 30
gh pr list --state merged --limit 30
```

Record:
- local-only branches
- remote-only branches
- detached or abandoned worktrees
- worktrees with uncommitted changes
- stash entries and what they contain

### 2. Deep-Read Every Branch and PR

For each open PR and recently closed PR:

```bash
gh pr view <number> --comments
gh pr view <number> --json title,body,comments,reviews,files,additions,deletions,commits
git diff origin/main...origin/<head-branch> --stat
git diff origin/main...origin/<head-branch> -- <relevant-files>
```

Read ALL review comments, inline code comments, requested changes, and the full PR body. Extract learnings even from PRs that will be discarded.

For each non-main local branch:

```bash
git log main..<branch> --oneline --stat
git log main..<branch> --pretty=format:'%H %s%n%b'
git diff main...<branch> --stat
```

For each worktree:

```bash
git -C <worktree-path> status
git -C <worktree-path> log --oneline -5
```

### 3. Build The Reconciliation Ledger

For each open PR, closed PR, or branch, record:

- PR number, title, author, head branch, base branch
- whether the branch still exists on remote and locally
- changed files and diff size
- last activity date
- whether the same area has already been modified on current `main`
- whether the branch overlaps another branch or PR
- whether there is uncommitted local work in the same area
- all key feedback from PR comments and reviews
- classification:
  - `ADOPT` - still valuable, code is compatible or nearly so
  - `ADAPT` - valuable idea but code is stale, needs reimplementation
  - `LEARN` - contains useful insights/feedback but code is obsolete
  - `MERGED` - already in main
  - `STALE` - outdated, superseded, or zero value
  - `ORPHANED` - points to non-existent remote or invalid ref
  - `ACTIVE` - do not touch, ongoing work
- merge eligibility:
  - `FF-ELIGIBLE` - verified fast-forward possible (`git merge-base --is-ancestor main <branch>`)
  - `MANUAL-ONLY` - requires reimplementation
  - `SKIP` - no merge needed (stale/orphaned/merged)

Detect overlap:
- compare changed file sets between branches
- compare commit ancestry
- compare branch diff themes
- if two branches touch the same area, decide which supersedes

This ledger is mandatory. No app edits before it exists.

### 4. Present Plan For User Review

Before executing, present the reconciliation plan to the user:
- Items to implement (ADOPT/ADAPT) with priority order
- Branches eligible for fast-forward merge
- Items to skip with reasons
- Cleanup targets (branches, worktrees, PRs to close/delete)
- Risk flags (uncommitted work, potential conflicts, force-delete needed)

Wait for user confirmation before proceeding.

### 5. Implement Enhancements

For FF-ELIGIBLE branches:
1. Re-verify: `git merge-base --is-ancestor main <branch>` must return 0
2. `git merge --ff-only <branch>`
3. If no longer FF (main moved): fall back to manual reimplementation

For ADOPT items (clean code):
1. Read the changes from the source branch
2. If cherry-pick applies cleanly with zero conflicts: `git cherry-pick <sha>`
3. Verify: `NODE_OPTIONS="--max-old-space-size=4096" npx tsc --noEmit`
4. If type errors: `git reset HEAD~1` and reimplement manually instead

For ADAPT items (stale code):
1. Read the diff to understand the intent
2. Reimplement the logic against current codebase patterns
3. Do not copy obsolete code
4. Verify type-check passes after each change

After each enhancement:
- Run type-check
- Commit: `git add -A && git commit -m 'feat: port <description> from <branch/PR#>'`

### 6. Verify Ported Work

After each meaningful port:

- run the fastest real verification for the affected area
- check type errors, lint or tests if relevant
- inspect runtime behavior when the change is user-facing
- verify no regression was introduced

Do not close the source PR until the useful changes are either:
- ported and verified, or
- proven obsolete, duplicate, or invalid

### 7. Close PRs Deliberately

When a PR is no longer needed:
- close it explicitly with `gh pr close <number> --comment '<reason>'`
- add a short closing note:
  - useful parts were manually ported to current `main`
  - or the PR is obsolete because current `main` already superseded it

Do not merge just to make the PR disappear.

### 8. Clean Up All Artifacts

After PR decisions are complete:

**Branches:**
- delete stale remote branches: `git push origin --delete <branch>`
- delete matching stale local branches: `git branch -d <branch>` (safe delete only)
- if `-d` fails: SKIP and document, do NOT force unless work is confirmed captured
- prune local tracking refs: `git remote prune origin`

**Worktrees:**
- remove worktrees with no uncommitted changes and merged/stale branch: `git worktree remove <path>`
- for worktrees with uncommitted changes: SKIP and warn the user
- prune stale refs: `git worktree prune`

**Stashes:**
- drop stash entries confirmed as already implemented: `git stash drop stash@{n}`
- keep any uncertain stash entries

**Garbage collection:**
- `git gc --prune=now`

### 9. Sync and Push

```bash
# Pre-push checks
git status  # must be clean
NODE_OPTIONS="--max-old-space-size=4096" npx tsc --noEmit  # must pass

# Push
git push origin main  # standard push, NEVER force

# If rejected (remote ahead):
git pull --rebase origin main
# If rebase conflicts: STOP and document

# Verify sync
git fetch origin
git log HEAD..origin/main --oneline  # must be empty
git log origin/main..HEAD --oneline  # must be empty
```

### 10. Write Reconciliation Report

Write to `.auto-claude/reconciliation-report.md`:

```markdown
# Git Reconciliation Report
Date: <today>

## Summary
- Items discovered: N
- Enhancements ported: N (list commit SHAs)
- Branches merged (FF): N (list)
- Deferred (needs manual work): N (list with reasons)
- PRs closed: N (list with numbers)
- Branches deleted: N (local + remote)
- Worktrees removed: N

## Implemented Enhancements
<per-item: source branch/PR, what was ported, commit SHA>

## Learnings Extracted
<key insights from PR comments and reviews that inform future work>

## Deferred Items
<per-item: what, why, recommended action>

## Final State
- Local main SHA: <sha>
- Remote main SHA: <sha>
- Remaining branches: <list>
- Remaining worktrees: <list>
- Sync status: SYNCED / BLOCKED
```

### 11. Final Verification

```bash
git branch -a           # only main + intentionally kept
git worktree list       # only main worktree
git stash list          # empty or intentionally kept
git status              # clean
git diff origin/main    # empty (fully pushed)
gh pr list --state open # no stale PRs
```

Cross-check: every VALUABLE item from the original ledger must be in one of: implemented, merged, deferred-with-docs. Zero items in unaccounted state.

## Decision Standard

Keep and manually port something only if it still improves the current codebase.

Discard it if any of these are true:

- current `main` already includes the same behavior
- the old implementation no longer fits the architecture
- the change solves a problem that no longer exists
- the branch is mostly noise with no clear surviving value

If you are unsure, inspect deeper before deleting. Cleanup is only successful when worthwhile work is preserved and dead branch clutter is removed.

If the first logged actions in a session are app-file edits, UI fixes, or build runs before the reconciliation ledger exists, treat that attempt as not correctly sequenced.
