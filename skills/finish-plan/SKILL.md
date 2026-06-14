---
name: finish-plan
description: Use when a plan's PR is ready to merge — marks the plan completed, checks PR status, merges to main, and syncs local main
---

# Finish Plan

Complete the plan lifecycle: mark it done, merge the PR, sync local main.

## When to Use

- After a plan's PR is open and all reviews/CI have passed
- When the user says "merge this", "ship it", "finish up", or "land this"
- After `execute-plan` has opened the PR

**Don't use when:**
- PR has failing CI — fix issues first
- Plan doesn't have a PR yet — use `execute-plan` to create one
- Design isn't approved yet — use `brainstorming`

## Process

### 1. Check Reviews and Comments FIRST

Before checking anything else, inspect the human review state:

```bash
gh pr view [PR-NUMBER] --json reviewRequests,reviews,comments
```

**A. Check for pending human review requests:**
Look at `reviewRequests` — any entries mean a human hasn't reviewed yet.

**B. Check for unresolved review comments:**
Look at `reviews` for:
- `state: "COMMENTED"` — has comments that may need addressing
- `state: "CHANGES_REQUESTED"` — blocking, must fix

Look at `comments` for:
- `isMinimized: false` — active comments (not resolved)
- Comments from human reviewers (not bots) that ask for changes

**C. Decision gate — ask the user if there are pending reviews:**

> If `reviewRequests` is non-empty (pending human reviewers):
> - **STOP** and ask the user: "@X hasn't reviewed yet. Wait or merge anyway?"
> - If user says wait, stop and report back
> - If user says proceed, continue to Step 2

> If there are `CHANGES_REQUESTED` reviews:
> - **STOP** and fix the issues (see Step 1a below)

> If there are `COMMENTED` reviews from bots or humans:
> - Read the latest comment body — if it says "no new issues found" or similar, continue
> - If it lists issues to fix, go to Step 1a

### 2. Check CI and Mergeability

Only after reviews are clear, check technical readiness:

```bash
gh pr view [PR-NUMBER] --json state,statusCheckRollup,reviewDecision,mergeable
```

Verify all of:
- **State**: `OPEN`
- **CI checks**: All passing (green) or no checks yet
- **Reviews**: `APPROVED` (or no review required)
- **Mergeable**: `MERGEABLE`

If CI is still running, wait and re-check. If CI is failing, go to Step 1a.

### 1a. Fix PR Issues (loop)

**A. Read each outstanding comment/issue** — understand what needs to change

**B. Fix on the feature branch:**
```bash
git checkout [feature-branch]
# Make the fix
git add -A && git commit -m "fix: address review comment — [summary]"
git push origin [feature-branch]
```

**C. Respond to the comment:**
```bash
gh pr comment [PR-NUMBER] --body "Fixed — [explanation of what changed]"
```

**D. Re-check reviews** — go back to Step 1 from the top (check reviews/comments first)

**E. Loop until:**
- All review comments are addressed
- All CI checks pass
- Review decision is `APPROVED` (or no review required)
- PR is `MERGEABLE`

**F. If a comment doesn't require a code change** (e.g., "nice work", general feedback):
- Reply with `gh pr comment` acknowledging the feedback
- Move on

**G. If you're blocked** (can't reproduce an issue, need clarification):
- Comment on the PR asking for clarification
- Report back to the user and stop

> **CRITICAL:** Do NOT skip unresolved review comments or failing CI. Loop until everything is green.

### 3. Merge the PR

```bash
gh pr merge [PR-NUMBER] --squash --delete-branch
```

Use squash merge by default to keep main history clean. If the user prefers merge commits, use `--merge` instead.

### 4. Sync Local Main

```bash
git checkout main
git pull origin main
```

### 5. Update Plan Index

Edit `docs/plans/README.md`:

1. Change the plan's status emoji from 🚧 to ✅
2. Add the PR number and key git refs to the entry (if not already present)
3. Update Quick Stats: decrement remaining count, increment completed count
4. If the plan was in the "Remaining Work" table, move it to the appropriate "Completed Plans" category

Commit the update:

```bash
git add docs/plans/README.md
git commit -m "docs: mark [plan-name] as completed (PR #[number])"
```

### 6. Report

Tell the user:
- PR was merged (with link)
- Local main is synced
- Plan index is updated
- Any follow-up items noted in the plan

## Common Issues

| Issue | Action |
|-------|--------|
| Pending human reviewers | Ask user: wait or merge anyway? |
| CI still running | Wait and re-check, or ask user if they want to proceed |
| CI failing | Fix on feature branch, push, re-check (loop until green) |
| Review comments unresolved | Fix each issue, push, reply to comment, re-check (loop until resolved) |
| `CHANGES_REQUESTED` review | Fix the requested changes, push, re-check |
| Bot review with issues | Fix the issues, push, re-check |
| Bot review clean ("no new issues") | Continue to next step |
| Merge conflicts on PR | Do NOT merge locally. Ask user to resolve on the branch. |
| PR already merged | Skip merge step, sync main, update index |
| Plan not in README.md | Add it as ✅ COMPLETED with the PR number |

## Rules

- **Always check reviews/comments BEFORE checking CI/status** — this is the first gate
- **Never skip pending human reviewers** — always ask the user
- **Never force-merge a PR with failing CI** — always report the issue
- **Always sync main after merge** — prevents stale branch issues
- **Always update the plan index** — this is the single source of truth for plan status
- **Use squash merge by default** — keeps main history clean
