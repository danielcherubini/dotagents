---
name: execute-plan
description: Use when you have a written implementation plan to execute
---

# Execute Plan

Read the plan, create a feature branch, dispatch subagents per task, review the branch, open a PR.

## Plan Selection

If no plan file path is provided by the user:

1. **Read the plans index** — Load `docs/plans/README.md` (or the project's equivalent plan index) to find available plans.
2. **Identify candidates** — Collect all plans marked as 🚧 IN PROGRESS or 📋 DRAFT / ❌ NOT STARTED (exclude ✅ COMPLETED, 🔁 SUPERSEDED, and any "remaining work" / roadmap items that lack a plan file).
3. **Ask the user** which plan to execute:

```
ask({
  questions: [{
    id: "plan",
    question: "Which plan would you like to execute?",
    options: [
      { label: "[Plan Name] — [short description]" },
      ...
    ]
  }]
})
```

4. If only one candidate exists, you may skip asking and proceed directly.
5. Once selected, read the full plan file and continue to Branch Setup.

## Branch Setup
- Create feature branch using gitflow conventions (load `gitflow-branching` skill if needed)
- Create a todo list with `manage_todo_list` with all tasks from the plan

## Baseline Check (Mandatory Before Task 1)

Before writing a single line of code, run the full test/lint/build suite and record the results:

```bash
# Run whatever the project's CI check is (go test, npm test, cargo test, etc.)
# Capture every FAIL, ERROR, and lint warning
```

**Rule: You own every failure in CI — pre-existing or not.**
If a test or lint check was already broken before your branch, you must still fix it before opening a PR. The CI gate does not know or care what was broken before you arrived. A PR that introduces zero regressions but leaves pre-existing failures will still be rejected. Fix them or report BLOCKED to the user before proceeding.

Document the baseline results in your working notes so you know which failures were pre-existing vs introduced by your changes.

## Task Dispatch Protocol

For each task, dispatch the `general` agent (sequentially, not parallel).

> **DO NOT add a `model` parameter to any subagent call.** The agent definition controls its own model. Adding `model` causes hallucinated model names that break the call.

```
subagent({
  agent: "general",
  task: "[FULL TEXT of task from plan — paste it, don't make the agent read a file]\n\nContext: [where this fits, dependencies, what's already done]",
  description: "Implement Task N: [task name]"
})
```

The `general` agent already knows to:
- Load TDD skill and follow RED-GREEN-REFACTOR
- Validate format → build → test → lint in order
- Commit with a descriptive message
- Report DONE | BLOCKED | NEEDS_CONTEXT

**Handle subagent responses:**
- **DONE:** Mark task complete in todo list, move to next task
- **NEEDS_CONTEXT:** Provide missing info, re-dispatch
- **BLOCKED:** Assess blocker, provide help or escalate to user

**Important:** Dispatch tasks sequentially (not in parallel) to avoid file conflicts.

## After All Tasks Complete

Once all tasks are done, ask the user what to do next:

```
ask({
  questions: [{
    id: "next-step",
    question: "All tasks complete. What would you like to do next?",
    options: [
      { label: "Code review then PR" },
      { label: "Open PR only" },
      { label: "Code review only" },
      { label: "Finish plan" }
    ]
  }]
})
```

Then follow the user's choice immediately — do NOT ask for additional confirmation.

### Code review then PR

1. Dispatch the **reviewer subagent** with: "Review the implementation against the plan at `docs/plans/YYYY-MM-DD-<feature>.md`. Check that all acceptance criteria are met and no planned work was missed."
2. Fix any critical/major issues from reviewer verdict
3. Load the `review` skill to conduct a thorough code review
4. **Clear the todo list** — remove all old task entries
5. **Create new todos** for each finding from the review:
   - One todo per issue found (blocking, important, nit, suggestion)
   - Title: `[severity] <brief description>`
   - Description: full details of the issue + suggested fix
6. Fix issues ONE AT A TIME using general subagents:

   **FOR EACH TODO ITEM, YOU MUST DISPATCH A SUBAGENT:**

   ```
   subagent({
     agent: "general",
     task: "Fix the following issue:\n\n[FULL TEXT from the todo item]\n\nLoad the `review` skill for guidance on best practices. Update the corresponding todo in the todo list to \"completed\" using manage_todo_list.",
     description: "Fix: [severity] <brief description>"
   })
   ```

   **CRITICAL:** Do NOT fix the issue yourself. You MUST dispatch a `general` subagent for each todo item. Wait for the subagent to complete, then mark that todo as completed.
7. Re-run the review once after all fixes
8. If issues persist, escalate to user
9. Then proceed to **Open PR** below

### Code review only

1. Dispatch the **reviewer subagent** with: "Review the implementation against the plan at `docs/plans/YYYY-MM-DD-<feature>.md`. Check that all acceptance criteria are met and no planned work was missed."
2. Fix any critical/major issues from reviewer verdict
3. Load the `review` skill to conduct a thorough code review
4. **Clear the todo list** — remove all old task entries
5. **Create new todos** for each finding from the review:
   - One todo per issue found (blocking, important, nit, suggestion)
   - Title: `[severity] <brief description>`
   - Description: full details of the issue + suggested fix
6. Fix issues ONE AT A TIME using general subagents:

   **FOR EACH TODO ITEM, YOU MUST DISPATCH A SUBAGENT:**

   ```
   subagent({
     agent: "general",
     task: "Fix the following issue:\n\n[FULL TEXT from the todo item]\n\nLoad the `review` skill for guidance on best practices. Update the corresponding todo in the todo list to \"completed\" using manage_todo_list.",
     description: "Fix: [severity] <brief description>"
   })
   ```

   **CRITICAL:** Do NOT fix the issue yourself. You MUST dispatch a `general` subagent for each todo item.
7. Re-run the review once after all fixes
8. If issues persist, escalate to user

### Open PR only

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

Report the PR URL to the user.

**Clear the todo list** — remove all remaining entries now that execution is complete.

### Finish Plan

1. **Clear the todo list** — remove all remaining entries
2. Load the `finish-plan` skill to check PR status, merge to main, and update the plan index

## Update Plan Index

After a PR is opened, update `docs/plans/README.md`:
1. Change the plan's status from 🚧 IN PROGRESS to ✅ COMPLETED
2. Add the PR number and key git commit refs to the entry
3. Decrement remaining count, increment completed count in Quick Stats
4. Commit this update with message: `docs: mark [plan-name] as completed`
5. **Clear the todo list** — remove all remaining entries
