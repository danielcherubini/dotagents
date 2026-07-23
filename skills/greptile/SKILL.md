---
name: greptile
description: Use when the user wants Greptile to review the current local branch iteratively until all findings are resolved, before opening a PR or merging to main.
---

# Greptile

Run `greptile review` on the current local branch, fix findings, and re-run until clean — all before opening a PR.

## Overview

This skill runs the Greptile CLI review against the local branch in a loop. Each iteration: run review → parse findings → fix actionable items → re-run. The loop exits when there are zero actionable findings, max iterations are reached, or the user aborts. After the loop, the skill asks whether to **Open a PR** (using the implement skill's PR logic) or **Merge to main** (using the finish skill).

## When to Use

- After `implement` tasks are complete and the user selects "Greptile review loop"
- When the user says "review with greptile", "greptile loop", "run greptile review"
- When the user wants Greptile feedback on the local branch before opening a PR

**Not for:** reviewing an already-open PR (use `greptileai/skills@greploop`), debugging a specific bug (use `systematic-debugging`), general code review (use `review`)

## Architecture

```
Main agent (YOU — running this skill)
  ├── Phase 1: Setup (repo context, greptile CLI, authentication)
  ├── Phase 2: Run greptile review — capture JSON output
  ├── Phase 3: Parse findings — classify actionable vs informational
  ├── Phase 4: Fix findings — dispatch general subagent to fix each actionable item
  ├── Phase 5: Re-run review — loop back to Phase 2 if findings remain
  ├── Phase 6: Exit conditions met — present summary
  └── Phase 7: Final ask() — Open a PR or Merge to main
```

**You (the main agent) own all orchestration AND all user interaction.** Subagents dispatched in Phase 4 are pure fixing leaves — they apply fixes and return summaries, never call `ask()` or present to the user.

## Workflow

### Phase 1: Setup

1. **Confirm repository context:**
   ```bash
   git rev-parse --show-toplevel
   ```
   If this fails, tell the user the skill must be run from a git repository.

2. **Check for Greptile CLI:**
   ```bash
   command -v greptile
   ```
   If missing, do NOT install automatically. Ask the user for permission, then show:
   ```bash
   npm i -g greptile
   ```
   Fallback if npm unavailable:
   ```bash
   curl -fsSL "https://greptile.com/cli/install" | sh
   ```

3. **Ensure authentication:**
   ```bash
   greptile whoami
   ```
   If auth is missing, run `greptile login` and wait for the user to complete the flow.

### Phase 2: Run Greptile Review

Run the review with JSON output for machine-parseable findings:

```bash
greptile review --json
```

If JSON output fails (unsupported version), fall back to:
```bash
greptile review --agent
```

Capture the full output. Do not hide raw command failures if both commands fail — report the failing command and the next action the user needs to take.

### Phase 3: Parse Findings

Parse the review output and classify each finding:

| Type | Description | Action |
|------|-------------|--------|
| **Actionable** | Code change needed (bug, refactor, missing validation, etc.) | Fix in Phase 4 |
| **Informational** | FYI, suggestion, or false positive | Note but don't fix |
| **Already addressed** | Resolved by prior commits | Skip |

If there are **zero findings**, skip to Phase 6 (exit condition met).

### Phase 4: Fix Findings

For each actionable finding, dispatch a `general` subagent with the context:

```
// Dispatch one general subagent per actionable finding (or batch related findings):
subagent({
  agent: "general",
  task: "Fix the following Greptile findings on the local branch. For each finding: read the file, understand the issue, make the fix, and stage the change with git add. Findings: [list with file paths and descriptions]. Return a summary of what was changed.",
  description: "Fix Greptile finding: [short summary]"
})
```

The `general` subagent (not `reviewer`) is the correct dispatch target for applying code fixes — it is the same agent the `implement` skill uses for task execution. The `reviewer` subagent is prohibited from making changes and only returns reports.

> **Ownership boundary:** The main agent owns the loop orchestration (run review → parse → fix → re-run). The `general` subagent owns the fix application. The main agent does NOT fix code directly — it delegates to the subagent and then re-runs the review.

### Phase 5: Re-run Review (Loop)

After fixes are applied:

1. Re-run `greptile review --json` (back to Phase 2)
2. Parse new findings (Phase 3)
3. If new actionable findings exist, fix them (Phase 4)
4. Repeat

**Loop guard:** Max 5 iterations to avoid runaway loops. Each iteration is one full cycle of Phase 2 → Phase 3 → Phase 4.

### Phase 6: Exit Conditions

Stop the loop if **any** of these are true:

| Condition | Behavior |
|-----------|----------|
| Zero actionable findings | Exit loop, present summary, go to Phase 7 |
| Max 5 iterations reached | Exit loop, report remaining findings, go to Phase 7 |
| Review command fails | Report the failure, go to Phase 7 |

### Phase 7: Final Ask

After the loop exits, present a summary and ask:

```
ask({
  questions: [{
    id: "final-action",
    question: "Greptile review loop complete. X findings resolved, Y remaining. What would you like to do?",
    options: [
      { label: "Open a PR" },
      { label: "Merge to main" }
    ],
    description: "**Summary:**\n- Iterations: N\n- Findings resolved: X\n- Remaining: Y\n- Final confidence: [score if available]\n\n**Note:** 'Open a PR' uses the implement skill's PR opening logic. 'Merge to main' uses the finish skill to merge and sync."
  }]
})
```

- **Open a PR** → Follow the implement skill's "Open PR only" procedure:
  ```bash
  git push -u origin [branch-name]
  gh pr create --title "[title]" --body "$(cat <<'EOF'
  ## Summary
  - [bullets]

  ## Test plan
  - [ ] [verification steps]
  EOF
  )"
  ```
  Report the PR URL to the user. Clear the todo list.

- **Merge to main** → Load the `finish` skill and run its full pipeline:
  - Check reviews and comments FIRST
  - Check CI and mergeability
  - Merge the PR (squash by default)
  - Sync local main
  - Update plan index

## Integration with Implement Skill

The `implement` skill's "After All Tasks Complete" section adds a new option:

```
{ label: "Greptile review loop" }
```

When selected:
1. Load the `greptile` skill
2. Run it to completion (Phases 1–6)
3. At Phase 7, the greptile skill asks "Open a PR" or "Merge to main"
4. If "Open a PR" → use implement's "Open PR only" logic
5. If "Merge to main" → load the `finish` skill

## Quick Reference

| Phase | Command | Purpose |
|-------|---------|---------|
| 1 | `git rev-parse --show-toplevel` | Confirm repo context |
| 1 | `command -v greptile` | Check CLI installed |
| 1 | `greptile whoami` | Check auth |
| 2 | `greptile review --json` | Run review (preferred) |
| 2 | `greptile review --agent` | Fallback if JSON unsupported |
| 5 | (loop back to Phase 2) | Re-run after fixes |
| 7 | `ask()` | Open a PR or Merge to main |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Running review once and stopping | Loop back to Phase 2 after fixes until zero findings |
| Not checking auth first | Always run `greptile whoami` in Phase 1 |
| Hiding CLI failures | Report the failing command and next action |
| Exceeding max iterations | Stop at 5 iterations and report remaining findings |
| Opening a PR during the loop | The greptile skill does NOT open PRs — only Phase 7 does, per user choice |
