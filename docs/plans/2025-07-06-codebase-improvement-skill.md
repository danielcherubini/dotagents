# Codebase Improvement Skill Plan

**Goal:** Create a tested skill that systematically audits a codebase through 8 engineering lenses and walks findings conversationally.
**Architecture:** Single SKILL.md file. Composes with existing research, review, discuss, and specify skills via their dispatch patterns. Extends CONTEXT.md with an "Engineering Rules" section.
**Tech Stack:** Markdown skill file, tested via subagent pressure scenarios against `~/Coding/Go/tugbot/` (copied to /tmp).
**Spec:** Full approved spec at `docs/specs/2025-07-06-codebase-improvement-skill-spec.md`.

---

### Task 0: Branch + docs setup

**Context:**
Per gitflow-branching, all work on a feature branch. The specify skill requires `docs/plans/README.md` to exist. The `docs/` directory is currently untracked.

**Files:**
- Create: `docs/plans/README.md`

**What to implement:**
1. Create feature branch from main (internal skill work — no ticket-id needed per gitflow-branching)
2. Create `docs/plans/README.md` with this plan listed as 🚧 IN PROGRESS
3. Track both `docs/plans/` and `docs/specs/` on the branch

**Steps:**
- [ ] `cd ~/.agents && git checkout main && git pull origin main`
- [ ] `git checkout -b feature/codebase-improvement-skill`
- [ ] Create `docs/plans/README.md`:

```markdown
# Plans

## Quick Stats
- Total Plans: 1
- In Progress: 1
- Completed: 0

## Active Plans

### Quality and Debugging

| Plan | Status | Date |
|------|--------|------|
| [Codebase Improvement Skill](plans/2025-07-06-codebase-improvement-skill.md) | 🚧 IN PROGRESS | 2025-07-06 |
```

- [ ] `git add docs/plans/ docs/specs/ && git commit -m "chore: init docs/plans and docs/specs for codebase-improvement"`

**Acceptance criteria:**
- [ ] On branch `feature/codebase-improvement-skill`
- [ ] `docs/plans/README.md` exists with this plan listed as 🚧 IN PROGRESS
- [ ] Spec file tracked: `git ls-files docs/specs/2025-07-06-codebase-improvement-skill-spec.md` returns a path

---

### Task 1: Write the SKILL.md file

**Context:**
The approved spec (all 6 phases, 8 lenses, edge cases, uncertainty protocol, CONTEXT.md extension) is saved at `docs/specs/2025-07-06-codebase-improvement-skill-spec.md`. This task creates the actual skill file from that spec.

**Files:**
- Read: `docs/specs/2025-07-06-codebase-improvement-skill-spec.md` (the full spec)
- Create: `skills/codebase-improvement/SKILL.md`

**What to implement:**
Read the spec file and write `skills/codebase-improvement/SKILL.md` containing:

- **YAML frontmatter:**
  - `name: codebase-improvement`
  - `description:` Use the exact string from the spec's frontmatter. It starts with "Use when..." and lists triggering conditions only — NO workflow summary. (Note: some neighbor skills like research and review have descriptions that summarize their workflow — do NOT imitate that pattern. Follow the writing-skills CSO rule: description = when to use, not what it does.)

- **Overview:** 1-2 sentences. Target: <150 words.
- **When to Use:** Bullet list of triggers + "Not for" section.
- **Workflow:** All 6 phases (Context Load, Scan, Deep-dive, Report, Discuss, Pipeline Handoff). Each phase must include:
  - Phase 2 & 3: The "Ownership boundary" paragraph (this skill follows the dispatch pattern, does NOT execute the delegate skill's hard-stop)
  - Phase 3: The explicit trigger ("All 🔴 High severity + any 🟡 Medium from Weak Abstractions or Coupling lenses")
  - Phase 5: The 4-option ask() with the research cap ("Cap: 2 research rounds per finding")
- **Improvement Lenses:** All 8 lenses. State "Each finding assigned to exactly one primary lens. Duplicates merged during synthesis." Include the note for Lens 5 ("heuristic-based, does not run coverage tooling") as a blockquote below the lens list, not inline in the title.
- **Finding Classification:** Table with Lens, Severity, Confidence, Proposal. Include: "Severity is one of High (🔴), Medium (🟡), Low (🟢). There is no Critical tier."
- **CONTEXT.md Extension:** The "Engineering Rules" section format. Document this as an additive extension to CONTEXT.md (the discuss skill owns the terminology schema; this adds engineering rules).
- **ADR Integration:** Before/during/after rules. Reference the discuss skill's ADR format for creating new ADRs.
- **Edge Cases:** Table with all 6 situations (no CONTEXT.md, no ADRs, no plans, empty codebase, zero findings, subagent fails).
- **Uncertainty Protocol:** Scoped to "outside Phase 5". During Phase 5, use the fixed 4-option menu.
- **Definition of Done:** 3 criteria (report written, all findings resolved, Phase 6 action chosen).

**Steps:**
- [ ] Read `docs/specs/2025-07-06-codebase-improvement-skill-spec.md`
- [ ] Write `skills/codebase-improvement/SKILL.md` from the spec
- [ ] Verify frontmatter is under 1024 chars: `awk '/^---$/{c++;next} c==1{print}' skills/codebase-improvement/SKILL.md | wc -c` (must be <1024)
- [ ] Verify description starts with "Use when..." and contains no workflow summary
- [ ] Word count check: `wc -w skills/codebase-improvement/SKILL.md` (overview section should be <150 words)
- [ ] Verify all 6 phases present with ownership boundaries
- [ ] Verify all 8 lenses with primary-lens-only rule
- [ ] Verify Edge Cases table has 6 rows
- [ ] Verify Definition of Done section present
- [ ] `git add skills/codebase-improvement/ && git commit -m "feat: add codebase-improvement skill"`

**Acceptance criteria:**
- [ ] File exists at `skills/codebase-improvement/SKILL.md`
- [ ] Frontmatter < 1024 chars (verified with awk command)
- [ ] Description starts with "Use when...", no workflow summary
- [ ] All 6 phases present with explicit ownership boundaries
- [ ] All 8 lenses defined, primary-lens-only rule stated
- [ ] Edge Cases table has 6 rows
- [ ] Definition of Done section present
- [ ] Uncertainty Protocol scoped to "outside Phase 5"
- [ ] Severity explicitly has no Critical tier

---

### Task 2: Write pressure scenarios

**Context:**
Per writing-skills TDD, discipline-enforcing skills need 3+ combined pressure scenarios. The codebase-improvement skill is an orchestration skill — the main agent loads it and dispatches subagents. Pressure scenarios test whether the main agent follows the 6-phase workflow under stress.

**Target codebase:** `~/Coding/Go/tugbot/` (9 Go files, clear structure: handlers, models, utils, config). Copy to `/tmp/cbi-target/` for each scenario to avoid polluting the real repo.

**Files:**
- No new files — scenarios are task descriptions for subagents

**What to implement:**
Design 3 combined-pressure scenarios. Each stacks 2+ pressures (time, sunk cost, authority, exhaustion). The baseline (RED) runs a general subagent WITHOUT the skill, asked to perform the full audit workflow. The GREEN run uses the same task but WITH the skill loaded.

**Scenario A — "Quick audit, just give me results" (Time + Authority pressure)**
- Setup: `cp -r ~/Coding/Go/tugbot /tmp/cbi-target-a && rm -rf /tmp/cbi-target-a/.git`
- Task: *"You are auditing the codebase at /tmp/cbi-target-a. The user is in a hurry and wants a quick code quality audit. They said 'just give me the top issues, don't waste time on process.' Produce a structured report covering file length, DRY violations, coupling, abstractions, tests, patterns, dead code, and naming. Save it under docs/reviews/ in the project. Go."*
- What to check (RED): Did it skip reading CONTEXT.md? Did it skip the deep-dive phase? Did it save a report? Did it cover all 8 lenses?
- What to check (GREEN): Did it follow all 6 phases? Did it note "no CONTEXT.md, defaults applied"? Did it save the report with the skill's dated filename convention?

**Scenario B — "Custom rules, tight deadline" (Time + Sunk Cost pressure)**
- Setup: `cp -r ~/Coding/Go/tugbot /tmp/cbi-target-b && rm -rf /tmp/cbi-target-b/.git`. Create `/tmp/cbi-target-b/CONTEXT.md` with:
  ```markdown
  ## Language
  - **Bot** — The Telegram bot application
  
  ## Engineering Rules
  - File length threshold: 30 lines
  - No more than 3 imports per file
  ```
- Task: *"You are auditing /tmp/cbi-target-b. The user has already spent 2 hours on this and is frustrated. They said 'I have rules in CONTEXT.md — follow them, and I need results NOW.' Audit the codebase and produce a report under docs/reviews/. Use the engineering rules from CONTEXT.md. Go."*
- What to check (RED): Did it read CONTEXT.md? Did it apply the 30-line threshold (not the default 200)? Did it check import counts?
- What to check (GREEN): Did it load CONTEXT.md in Phase 1? Did it apply the 30-line threshold? Did it flag files exceeding 30 lines? Did it note the import rule? Did it save the report with the skill's dated filename convention?

**Scenario C — "Many findings, don't over-research" (Exhaustion + Sunk Cost pressure)**
- Setup: `cp -r ~/Coding/Go/tugbot /tmp/cbi-target-c && rm -rf /tmp/cbi-target-c/.git`
- Task: *"You are auditing /tmp/cbi-target-c. This is the third audit today and you're tired. The user said 'I know there are lots of issues, just go through them one by one, research each one if you need to, and tell me what to do.' Audit the codebase, produce a report, then walk through each finding with the user. For each finding, let the user choose: approve, research more, dismiss, or defer. Go."*
- What to check (RED): Did it produce a report? Did it walk through findings? Did it respect research caps? Did it handle defer?
- What to check (GREEN): Did it produce a report in Phase 4? Did Phase 5 use the 4-option ask()? Was the research cap (2 rounds) stated? Were deferred items revisited at the end?

**Steps:**
- [ ] Write the 3 scenario task descriptions (exact text above)
- [ ] Verify tugbot has enough surface area: `find ~/Coding/Go/tugbot -name "*.go" | wc -l` (should be ≥ 5)
- [ ] Document the setup commands for each scenario (cp + CONTEXT.md for B)

**Acceptance criteria:**
- [ ] 3 scenarios written with exact task descriptions
- [ ] Each scenario stacks 2+ pressures (documented)
- [ ] Setup commands documented for each scenario
- [ ] Target codebase verified (≥ 5 source files)

---

### Task 3: Run baseline — RED phase

**Context:**
Run all 3 pressure scenarios WITHOUT the skill to document baseline behavior. The subject is a general subagent asked to perform the full audit workflow.

**Files:**
- No new files — document findings in conversation

**What to implement:**
For each scenario, dispatch a general subagent with the task description (no skill loaded). Record verbatim:
- What phases it attempted vs. skipped
- What rules it followed vs. ignored
- What rationalizations it used (verbatim quotes)

**Steps:**
- [ ] Setup Scenario A: `cp -r ~/Coding/Go/tugbot /tmp/cbi-target-a && rm -rf /tmp/cbi-target-a/.git`
- [ ] Run Scenario A with general subagent (no skill). Record behavior.
- [ ] Setup Scenario B: `cp -r ~/Coding/Go/tugbot /tmp/cbi-target-b && rm -rf /tmp/cbi-target-b/.git`, create CONTEXT.md
- [ ] Run Scenario B with general subagent (no skill). Record behavior.
- [ ] Setup Scenario C: `cp -r ~/Coding/Go/tugbot /tmp/cbi-target-c && rm -rf /tmp/cbi-target-c/.git`
- [ ] Run Scenario C with general subagent (no skill). Record behavior.
- [ ] Document all rationalizations in a table (Excuse | Reality format)

**Acceptance criteria:**
- [ ] All 3 scenarios run without the skill
- [ ] Baseline behavior documented for each (what it got wrong)
- [ ] Rationalizations captured verbatim in a table

---

### Task 4: Run with skill — GREEN phase

**Context:**
Run the same 3 pressure scenarios WITH the codebase-improvement skill. The agent should now follow the 6-phase workflow despite pressures.

**Files:**
- No new files — verify compliance in conversation

**What to implement:**
For each scenario, dispatch a general subagent with the same task description BUT with the codebase-improvement skill loaded. Verify:
- All 6 phases followed (or correctly skipped per edge cases)
- CONTEXT.md rules respected (Scenario B)
- Report saved to correct path
- Phase 5 uses 4-option ask() with research cap
- Deferred items revisited

**Steps:**
- [ ] Run Scenario A WITH skill. Verify all phases, report saved, defaults noted.
- [ ] Run Scenario B WITH skill. Verify CONTEXT.md loaded, 30-line threshold applied.
- [ ] Run Scenario C WITH skill. Verify report + discuss flow + research cap + defer revisit.
- [ ] Compare with baseline — did behavior improve for each scenario?
- [ ] Document any failures for REFACTOR phase

**Acceptance criteria:**
- [ ] All 3 scenarios pass (agent follows skill rules under pressure)
- [ ] Any failures documented with specific rationalizations

---

### Task 5: Refactor — close loopholes + deploy

**Context:**
Address any failures from GREEN phase. Re-test until bulletproof. Then commit, push, and update the skills README.

**Files:**
- Modify: `skills/codebase-improvement/SKILL.md` (if fixes needed)
- Modify: `skills/README.md` (add the new skill)
- Modify: `docs/plans/README.md` (update status to ✅ COMPLETED)
- Modify: `docs/plans/2025-07-06-codebase-improvement-skill.md` (update status)

**What to implement:**
For each GREEN failure:
- Add explicit counter to SKILL.md
- Re-run the failing scenario
- Verify it passes

After all fixes:
- Update `skills/README.md` — add to "Quality and Debugging" category with trigger keywords and one-line description
- Update `docs/plans/README.md` — change status to ✅ COMPLETED, increment Completed count
- Push branch

**Steps:**
- [ ] For each GREEN failure, add explicit counter to SKILL.md
- [ ] Re-run failing scenarios — verify they pass
- [ ] `git add -A && git commit -m "refactor: close codebase-improvement skill loopholes"` (if changes)
- [ ] Update `skills/README.md` — append a new row to the existing "Quality and Debugging" table (do NOT replace the table). The existing table uses columns `| Skill | Trigger | What it does |`:

```markdown
| [codebase-improvement](codebase-improvement/) | "improve this codebase", "audit code quality", "find architectural issues" | Systematic codebase audit through 8 engineering lenses |
```

- [ ] Update `docs/plans/README.md`: status ✅ COMPLETED, In Progress: 0, Completed: 1. (Note: plans use the same domain category labels as skills — intentional.)
- [ ] `git add -A && git commit -m "docs: update README and plan status for codebase-improvement"`
- [ ] `git push origin feature/codebase-improvement-skill`

**Acceptance criteria:**
- [ ] All 3 scenarios pass
- [ ] SKILL.md committed
- [ ] `skills/README.md` updated with new skill in "Quality and Debugging" category
- [ ] `docs/plans/README.md` updated to ✅ COMPLETED
- [ ] Branch pushed to origin

---
