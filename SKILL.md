---
name: branch-pr-reconciliation-skill
description: "Reconcile a repo's open PRs, remote branches, local branches, worktrees, stashes, and parallel-agent work by reading real diffs and raw session evidence, then manually port only the still-useful changes onto the current mainline and clean up safely."
---

# Branch PR Reconciliation Skill

Use this skill when the user wants branch, PR, worktree, or agent-work sprawl cleaned up without losing worthwhile work.

The real job is:

1. discover **all** state first, not just GitHub PRs
2. recover intent from raw diffs, worktrees, stashes, and agent/session logs
3. compare every relevant item against the current verified codebase
4. preserve useful work on the current mainline
5. close or delete only what is proven obsolete or fully captured
6. leave the repo, remote, worktrees, and session-related artifacts in a clean truthful state

This skill is instructions only. Do not build helper tooling unless the user explicitly asks for it.

## Why This Skill Exists

Typical failure mode:

- agent checks open PRs only
- misses local worktrees, stashes, or parallel agents
- trusts prior "done" claims
- deletes or ignores work that only existed locally
- reports "clean" while useful work is still sitting in another worktree or branch

This skill exists to prevent that.

## Hard Rules

- Reconciliation comes before implementation. No app edits before the reconciliation ledger exists.
- Never assume GitHub is the full source of truth. Local branches, worktrees, stashes, reflog, and parallel-agent sessions matter.
- Never assume a PR is useful because the title sounds useful.
- Never assume a branch is stale because it is closed, old, or merged-looking. Verify against current code.
- Never assume a prior report, prior agent summary, or prior "done" claim is accurate. Treat it as a lead only.
- Never attribute a commit, branch, or uncommitted diff to a specific agent from naming alone. Use timestamps, reflog, session logs, and actual branch state.
- Always compare old work against the current base branch and current files first.
- Port useful ideas manually so the final implementation fits the current codebase.
- If an old change conflicts with current architecture, adapt it or discard it.
- Close PRs only after you understand whether anything in them still matters.
- Delete branches, worktrees, or stashes only after the useful work is preserved or clearly obsolete.
- Do not delete the user's current branch or any branch with clearly active ongoing work unless the user asked for aggressive cleanup and the branch is proven redundant.
- `git diff --stat` alone does not count as reconciliation.
- UI fixes, app edits, or build runs before branch and worktree reconciliation mean the procedure is mis-sequenced.
- Never force-push to main.
- Never use `git branch -D` unless safe-delete `-d` fails and you have confirmed the work is captured or valueless.
- Never claim "everything is clean" until every potentially relevant state bucket is accounted for: local branches, remote branches, worktrees, stashes, session-derived work, and uncommitted changes.

## When To Go Beyond GitHub

If the user mentions any of these, you must inspect local agent/session history before deciding:

- a named conversation, thread, or report
- Codex, Claude, Kimi, Cursor, Aider, Continue, or "parallel agents"
- "worktrees", "stashes", "logs", "sessions", "previous agents", "check again", "missed work"
- claims that a prior reconciliation missed local work

When that happens, raw session files are evidence. Read them sequentially. Do not rely on summaries.

## Required First Pass

Before touching application code, complete and record all of this:

1. repo identity and current branch
2. local + remote branch inventory
3. worktree inventory
4. stash inventory
5. local uncommitted work inventory in the main worktree and every linked worktree
6. branch graph, reflog, and recent commit inventory
7. open PR inventory with comments, reviews, and review feedback
8. recently closed/merged PR inventory
9. branch-to-main diff review for all relevant branches
10. remote orphan detection
11. duplicate and overlap detection
12. parallel-agent/session inventory
13. raw session/log review for any referenced branch, task, or report
14. explicit keep, port, close, delete, or defer decision per branch, PR, worktree, stash, and session-derived work item

Do not start implementation until this pass is complete.

## Workflow

### 1. Confirm The Real Repo And Current State

Work in the real local repo first:

```bash
pwd
git rev-parse --show-toplevel
git status --short --branch
git remote -v
git branch --show-current
git fetch --all --prune
```

Record:

- repo path
- current branch
- main branch name
- whether the current checkout is the main worktree or a linked worktree

### 2. Discover All Git State

Read everything:

```bash
# Repo structure
tree -I node_modules -L 2

# Git state
git status --short --branch
git worktree list --porcelain
git branch --all --verbose --no-abbrev
git remote -v
git stash list
git log --oneline --decorate --graph -50 --all
git reflog --date=iso -50

# PRs
gh pr list --state open
gh pr list --state closed --limit 50
gh pr list --state merged --limit 50
```

Record:

- local-only branches
- remote-only branches
- worktrees and their branch bindings
- detached or abandoned worktrees
- worktrees with uncommitted changes
- stash entries and what they contain
- recently moved or deleted branches revealed by reflog

### 3. Discover Parallel Agent And Session State

Do not assume one agent or one session store.

Check for likely agent/session roots:

```bash
ps aux | rg -i 'codex|claude|kimi|cursor|aider|continue' || true

find ~/.codex ~/.claude ~/.kimi ~/.cursor ~/.aider ~/.continue \
  -maxdepth 4 \
  \( -name '*.jsonl' -o -name '*.md' -o -name '*.log' \) \
  2>/dev/null | tail -200
```

If the user provided a specific session ID, conversation title, or report name:

- locate the exact session/report file
- read the raw file sequentially
- extract branches mentioned, worktrees mentioned, files touched, commits claimed, and unresolved items

At minimum, identify:

- main orchestrator sessions
- spawned worker/subagent sessions
- which repo each session refers to
- whether a session produced code edits, a commit, a stash, a report, or only analysis

When multiple agents are active on the same repo, explicitly map:

- which branch each agent used
- which files each agent touched
- whether a commit landed on `main` or on a side branch
- whether one agent branched from another agent's already-pushed commit

Do not infer authorship from commit-message wording or from issue IDs that may have been copied between agents.

### 4. Inventory Worktrees And Uncommitted Work Properly

For each worktree:

```bash
git -C <worktree-path> status --short --branch
git -C <worktree-path> log --oneline -5
git -C <worktree-path> diff --stat
git -C <worktree-path> stash list
```

Also inspect possible abandoned worktree directories the repo no longer lists if the user suspects cleanup drift.

Do not stop at clean worktrees. A clean worktree can still point to a branch with unique commits.

### 5. Deep-Read Every Branch And PR

For each open PR and recently relevant closed PR:

```bash
gh pr view <number> --comments
gh pr view <number> --json title,body,comments,reviews,files,additions,deletions,commits,headRefName,baseRefName,author
git diff origin/main...origin/<head-branch> --stat
git diff origin/main...origin/<head-branch> -- <relevant-files>
git log origin/main..<head-branch> --oneline --stat
```

Read all useful review comments, requested changes, and follow-up comments. Extract learnings even from PRs that will be discarded.

For each non-main local branch:

```bash
git log main..<branch> --oneline --stat
git log main..<branch> --pretty=format:'%H %s%n%b'
git diff main...<branch> --stat
git diff main...<branch> -- <relevant-files>
```

### 6. Build The Reconciliation Ledger

For each open PR, closed PR, local branch, remote-only branch, worktree, stash, or session-derived work item, record:

- identifier: PR number, branch name, worktree path, stash ID, session ID
- author / agent / source
- head branch and base branch when applicable
- whether the branch still exists locally and remotely
- changed files and rough diff size
- last activity date
- whether the same area has already been modified on current `main`
- whether the item overlaps another branch, PR, worktree, or stash
- whether there is uncommitted local work in the same area
- all key feedback from PR comments and reviews
- whether the item contains actual code, only notes, or only claims
- classification:
  - `ADOPT` - still valuable and implementation is compatible or nearly so
  - `ADAPT` - valuable idea but code is stale and needs reimplementation
  - `LEARN` - useful insights or feedback only
  - `MERGED` - already in main
  - `STALE` - outdated, superseded, or zero value
  - `ORPHANED` - invalid ref, missing remote, detached worktree, or abandoned artifact
  - `ACTIVE` - ongoing work, do not touch without explicit coordination
- merge eligibility:
  - `FF-ELIGIBLE` - verified fast-forward possible
  - `MANUAL-ONLY` - requires reimplementation
  - `SKIP` - no merge needed

Detect overlap using all of:

- changed file sets
- commit ancestry
- branch themes
- session/log evidence
- actual code already on main

This ledger is mandatory. No app edits before it exists.

### 7. Decide Autonomy Level Correctly

Default behavior:

- if the user explicitly asked for end-to-end cleanup, proceed after the ledger and a brief checkpoint
- if the user explicitly asked to review the plan first, stop for confirmation

Do not hide behind "waiting for confirmation" when the user clearly asked you to finish the reconciliation.

### 8. Implement Preserved Work

For `FF-ELIGIBLE` branches:

1. Re-verify: `git merge-base --is-ancestor main <branch>`
2. Use `git merge --ff-only <branch>`
3. If no longer FF-eligible, fall back to manual integration

For `ADOPT` items:

1. Read the source diff and current target files
2. Prefer manual porting
3. Only cherry-pick if it applies with zero conflicts and the result passes verification
4. If a cherry-pick causes issues, back it out and reimplement manually

For `ADAPT` items:

1. Read the old diff to understand intent
2. Reimplement the behavior against current architecture
3. Do not copy obsolete code blindly

After each meaningful port:

- run the fastest relevant verification
- run type-check if appropriate
- commit with a scoped message

### 9. Verify Ported Work

After each meaningful port:

- run the fastest real verification for the affected area
- inspect runtime behavior if the change is user-facing
- verify no regression was introduced
- confirm the source item is now implemented, obsolete, or still deferred

Do not close the source PR until the useful parts are either ported and verified or proven obsolete.

### 10. Close PRs Deliberately

When a PR is no longer needed:

```bash
gh pr close <number> --comment '<reason>'
```

Closing note should say either:

- useful parts were manually ported to current `main`, or
- current `main` already supersedes the PR

Do not merge just to make a PR disappear.

### 11. Clean Up Artifacts Safely

After PR decisions are complete:

**Branches**

```bash
git push origin --delete <branch>
git branch -d <branch>
git remote prune origin
```

- Use safe delete first
- If safe delete fails, skip and document unless the work is fully accounted for

**Worktrees**

```bash
git worktree remove <path>
git worktree prune
```

- remove only when branch is merged/stale and the worktree has no unique uncommitted work
- if dirty, either preserve the work first or explicitly defer it

**Stashes**

```bash
git stash show -p stash@{n}
git stash drop stash@{n}
```

- drop only when the work is clearly captured or valueless
- keep uncertain stashes

**Garbage collection**

```bash
git gc --prune=now
```

### 12. Sync And Push

```bash
git status
NODE_OPTIONS="--max-old-space-size=4096" npx tsc --noEmit
git push origin main
```

If push is rejected:

```bash
git pull --rebase origin main
```

If rebase conflicts, stop and document the exact blocker.

Verify sync:

```bash
git fetch origin
git log HEAD..origin/main --oneline
git log origin/main..HEAD --oneline
```

Both must be empty before claiming success.

### 13. Write Reconciliation Report

Write to `.auto-claude/reconciliation-report.md`:

```markdown
# Git Reconciliation Report
Date: <today>

## Summary
- Items discovered: N
- Enhancements ported: N
- Branches merged (FF): N
- Deferred: N
- PRs closed: N
- Branches deleted: N
- Worktrees removed: N
- Stashes dropped: N

## Ledger
<per item: identifier, classification, decision, proof>

## Implemented Enhancements
<source branch/PR/session, what was ported, commit SHA>

## Learnings Extracted
<key insights from PR comments, reviews, and session logs>

## Deferred Items
<what, why, recommended action>

## Final State
- Local main SHA: <sha>
- Remote main SHA: <sha>
- Remaining branches: <list>
- Remaining worktrees: <list>
- Remaining stashes: <list>
- Remaining active sessions affecting repo: <list or none>
- Sync status: SYNCED / BLOCKED
```

### 14. Final Verification

Run:

```bash
git branch -a
git worktree list
git stash list
git status
git diff origin/main
gh pr list --state open
```

Cross-check:

Every valuable item from the ledger must end in exactly one of:

- implemented
- already merged
- deliberately deferred with reason
- proven stale

Zero unaccounted items.

## Decision Standard

Keep and manually port something only if it still improves the current codebase.

Discard it if any of these are true:

- current `main` already includes the behavior
- the old implementation no longer fits the architecture
- the change solves a problem that no longer exists
- the branch is mostly noise with no clear surviving value

If you are unsure, inspect deeper before deleting.

Cleanup is only successful when worthwhile work is preserved and dead branch clutter is removed without losing local-only work, session-only discoveries, or uncommitted branch state.

If the first logged actions in a session are app-file edits, UI fixes, or build runs before the reconciliation ledger exists, treat that attempt as not correctly sequenced.
