---
name: execute-plan
description: Use when you have a written implementation plan to execute
---

# Execute Plan

Read the plan, create a feature branch, dispatch subagents per task, review the branch, open a PR.
## Branch Setup
- Create feature branch using gitflow conventions (load `gitflow-branching` skill if needed)
- Create a todo list with `manage_todo_list` with all tasks from the plan

## Task Dispatch Protocol

For each task, dispatch a `general` subagent (sequentially, not parallel):

```
Subagent prompt template:
  description: "Implement Task N: [task name]"
  prompt: |
    You are BUILDING code in [project]. DO NOT PLAN — write actual file edits using the Edit and Write tools.

    ## CRITICAL INSTRUCTIONS
    1. READ the files listed below first to understand the existing patterns
    2. USE the Edit tool to modify existing files — do NOT rewrite entire files
    3. USE the Write tool to create new files
    4. RUN `cargo fmt --all`, then `cargo check --workspace`, then `cargo test --workspace` to validate
    5. COMMIT your work with: `git commit -am "[commit message]"`
    6. If cargo check fails, READ the error, READ the relevant source file, FIX the code with Edit tool, then re-run check
    7. DO NOT output planning text — output actual file changes

    ## Task
    [FULL TEXT of task from plan - paste it, don't make subagent read file]

    ## Context
    [Where this fits, dependencies, what's already done]

    ## Instructions
    - Implement exactly what the task specifies
    - Write tests (TDD: failing test first, then implementation)
    - Validate your work by running each step **independently and in order** — wait for each to finish before starting the next:
      1. Formatting (e.g. `cargo fmt`, `prettier`, etc.)
      2. Build / compile (e.g. `cargo build`, `npm run build`, etc.)
      3. Tests (e.g. `cargo test`, `npm test`, etc.)
    - If any step fails: **STOP. Do not re-run it yet.** Read the error output, read the relevant source files, then use Edit/Write tools to fix the root cause. Only re-run after you have made file changes.
    - **Loop-break rule:** If you run a step and it fails, and you have made no file edits since the last time it failed, you are looping. Stop immediately and report back with status BLOCKED — describe the error and what you tried.
    - Commit your work with a descriptive message
    - Work from: [directory]

    ## Report back with:
    - Status: DONE | BLOCKED | NEEDS_CONTEXT
    - What you implemented
    - Files changed
    - Any concerns
```

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
     task: "You are BUILDING code in [project]. DO NOT PLAN — write actual file edits using the Edit and Write tools.\n\n## CRITICAL INSTRUCTIONS\n1. READ the files listed below first to understand the existing patterns\n2. USE the Edit tool to modify existing files — do NOT rewrite entire files\n3. RUN `cargo fmt --all`, then `cargo check --workspace`, then `cargo test --workspace` to validate\n4. COMMIT your work with: `git commit -am \"[commit message]\"`\n5. If cargo check fails, READ the error, READ the relevant source file, FIX the code with Edit tool, then re-run check\n6. DO NOT output planning text — output actual file changes\n\n## Issue to Fix\n[FULL TEXT from the todo item]\n\n## Instructions\n- Load the `review` skill for guidance on best practices\n- Fix the issue exactly as described\n- Validate your fix by running tests/linting/build\n- Update the corresponding todo in the todo list to \"completed\" using manage_todo_list\n- Commit your fix with a descriptive message\n- Work from: [directory]\n\n## Report back with:\n- Status: DONE | BLOCKED\n- What you fixed\n- Files changed",
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
     task: "You are BUILDING code in [project]. DO NOT PLAN — write actual file edits using the Edit and Write tools.\n\n## CRITICAL INSTRUCTIONS\n1. READ the files listed below first to understand the existing patterns\n2. USE the Edit tool to modify existing files — do NOT rewrite entire files\n3. RUN `cargo fmt --all`, then `cargo check --workspace`, then `cargo test --workspace` to validate\n4. COMMIT your work with: `git commit -am \"[commit message]\"`\n5. If cargo check fails, READ the error, READ the relevant source file, FIX the code with Edit tool, then re-run check\n6. DO NOT output planning text — output actual file changes\n\n## Issue to Fix\n[FULL TEXT from the todo item]\n\n## Instructions\n- Load the `review` skill for guidance on best practices\n- Fix the issue exactly as described\n- Validate your fix by running tests/linting/build\n- Update the corresponding todo in the todo list to \"completed\" using manage_todo_list\n- Commit your fix with a descriptive message\n- Work from: [directory]\n\n## Report back with:\n- Status: DONE | BLOCKED\n- What you fixed\n- Files changed",
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
