# Greptile Skill Plan

**Goal:** Create a skill that runs `greptile review` iteratively on the local branch until clean (no actionable findings), then asks whether to open a PR or merge to main — without opening a PR during the review loop.
**Architecture:** Single SKILL.md file as an orchestration skill. Main agent runs greptile CLI review, parses JSON findings, fixes issues via subagent dispatch, re-runs until clean (max iterations), then hands off to `implement` (open PR) or `finish` (merge to main).
**Tech Stack:** Markdown skill file, tested via subagent pressure scenarios. Greptile CLI already installed globally (`greptile@3.2.3`).
**Spec:** Full approved spec at `docs/specs/spec-001-greptile-skill-spec.md`.

---

### Task 0: Branch + docs setup

**Context:**
Per gitflow-branching, all work on a feature branch. The `docs/plans/README.md` already exists from the codebase-improvement plan. This plan needs to be added to it.

**Files:**
- Modify: `docs/plans/README.md` (add this plan as 🚧 IN PROGRESS)

**What to implement:**
1. Create feature branch from main
2. Add this plan to `docs/plans/README.md` as 🚧 IN PROGRESS

**Steps:**
- [ ] `cd ~/.agents && git checkout main && git pull origin main`
- [ ] `git checkout -b feature/greptile-skill`
- [ ] Edit `docs/plans/README.md`: increment Total Plans to 2, add this plan as 🚧 IN PROGRESS in a new "Code Quality" category (or "Quality and Debugging" — see note)
- [ ] `git add docs/plans/ && git commit -m "chore: add greptile-skill plan"`

**Acceptance criteria:**
- [ ] On branch `feature/greptile-skill`
- [ ] `docs/plans/README.md` lists this plan as 🚧 IN PROGRESS

---

### Task 1: Write the spec

**Context:**
Per writing-skills TDD, write the spec first. The spec defines the greptile skill's behavior: iterative local-branch review loop, fix-and-rerun cycle, exit conditions, and the final ask() for Open PR vs Merge to main.

**Files:**
- Create: `docs/specs/spec-001-greptile-skill-spec.md`

**What to implement:**
Write a spec covering:
- **Overview:** Run greptile CLI review on local branch, iterate until clean, then offer PR or merge
- **When to Use:** After implement tasks are complete, when user wants greptile review before PR; standalone when user says "review with greptile" or "greptile loop"
- **Architecture:** Main agent orchestration flow (setup → review → parse → fix → rerun → loop → ask)
- **Workflow phases:** Setup, Review, Parse, Fix, Rerun, Loop, Final Ask
- **Exit conditions:** Zero findings, max iterations, user abort
- **Integration with implement:** New option in "After All Tasks Complete" ask()
- **Handoff:** Open PR (implement) or Merge to main (finish)

**Steps:**
- [ ] Write `docs/specs/spec-001-greptile-skill-spec.md`
- [ ] Verify spec covers all workflow phases and exit conditions

**Acceptance criteria:**
- [ ] Spec file created at `docs/specs/spec-001-greptile-skill-spec.md`
- [ ] Spec covers: setup, review loop, fix mechanism, exit conditions, final ask(), implement integration
- [ ] Spec defines handoff to implement (open PR) and finish (merge to main)

---

### Task 2: Write the SKILL.md file

**Context:**
The spec is saved at `docs/specs/spec-001-greptile-skill-spec.md`. This task creates the actual skill file from that spec.

**Files:**
- Read: `docs/specs/spec-001-greptile-skill-spec.md`
- Create: `skills/greptile/SKILL.md`

**What to implement:**
Write `skills/greptile/SKILL.md` following the writing-skills SKILL.md structure:
- YAML frontmatter (name + description, max 1024 chars)
- Overview (1-2 sentences)
- When to Use (triggers + not for)
- Workflow (phases with clear loop structure)
- Exit conditions table
- Final ask() for Open PR vs Merge to main
- Integration note with implement skill

**Steps:**
- [ ] Read `docs/specs/spec-001-greptile-skill-spec.md`
- [ ] Write `skills/greptile/SKILL.md`
- [ ] Verify frontmatter < 1024 chars
- [ ] Verify description starts with "Use when..." and contains triggers only
- [ ] Verify workflow covers all phases
- [ ] `git add skills/greptile/ && git commit -m "feat: add greptile skill"`

**Acceptance criteria:**
- [ ] File exists at `skills/greptile/SKILL.md`
- [ ] Frontmatter < 1024 chars
- [ ] Description starts with "Use when..." with triggers only
- [ ] All workflow phases present
- [ ] Exit conditions defined
- [ ] Final ask() for Open PR vs Merge to main

---

### Task 3: Integrate into implement skill

**Context:**
The implement skill's "After All Tasks Complete" section has an ask() with 4 options. This task adds a "Greptile review loop" option that loads the greptile skill, runs it to completion, then asks Open PR or Merge to main.

**Files:**
- Modify: `skills/implement/SKILL.md`

**What to implement:**
Add a new option to the "After All Tasks Complete" ask():
```
{ label: "Greptile review loop" }
```

Then add a new section "Greptile Review Loop" that:
1. Loads the greptile skill
2. Runs it to completion (iterative review until clean)
3. After greptile skill completes, asks: "Open a PR" or "Merge to main"
4. If Open a PR → uses the existing "Open PR only" logic
5. If Merge to main → loads the finish skill

**Steps:**
- [ ] Read `skills/implement/SKILL.md` "After All Tasks Complete" section
- [ ] Add "Greptile review loop" option to the ask()
- [ ] Add the "Greptile Review Loop" section with handoff logic
- [ ] `git add skills/implement/ && git commit -m "feat: add greptile review loop option to implement"`

**Acceptance criteria:**
- [ ] "Greptile review loop" option added to implement ask()
- [ ] New section documents the greptile → implement/finish handoff
- [ ] No existing options removed or broken

---

### Task 4: Update skills README

**Context:**
The skills README needs to list the new greptile skill in the appropriate category.

**Files:**
- Modify: `skills/README.md`

**What to implement:**
Add greptile skill to the "Quality and Debugging" table (or create a new "Review Automation" category if more appropriate).

**Steps:**
- [ ] Add row to `skills/README.md` for greptile skill
- [ ] `git add skills/README.md && git commit -m "docs: add greptile skill to README"`

**Acceptance criteria:**
- [ ] Greptile skill listed in skills/README.md with trigger keywords and one-line description

---

### Task 5: Update plan index

**Context:**
Mark the plan as completed in the plan index.

**Files:**
- Modify: `docs/plans/README.md`

**What to implement:**
1. Move plan from 🚧 IN PROGRESS to ✅ COMPLETED
2. Increment Completed count

**Steps:**
- [ ] Update `docs/plans/README.md` status to ✅ COMPLETED
- [ ] `git add docs/plans/ && git commit -m "docs: mark greptile-skill plan as completed"`

**Acceptance criteria:**
- [ ] Plan listed as ✅ COMPLETED in docs/plans/README.md
