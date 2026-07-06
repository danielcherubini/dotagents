# Skills

Specialized instruction sets that extend the agent's capabilities for specific workflows. Each skill is a self-contained directory with a `SKILL.md` file containing step-by-step guidance, checklists, and best practices.

Skills are loaded automatically when the agent detects matching triggers in your messages, or you can reference them explicitly.

---

## Development Workflow

Skills that cover the full feature lifecycle — from idea to merge:

| Skill | Trigger | What it does |
|-------|---------|--------------|
| [discuss](discuss/) | "let's discuss", "lets brainstorm" | Collaborative design dialogue before coding. Establishes shared terminology, proposes approaches with trade-offs, captures ADRs and glossary terms. Hard gate — no code until design is approved. |
| [specify](specify/) | After design approval | Turns an approved design into a structured implementation plan with independent, commitable tasks. Writes to `docs/plans/`. Includes reviewer pass against the actual codebase. |
| [implement](implement/) | After plan is ready | Executes the plan: creates feature branch, runs baseline checks, dispatches subagents per task, handles reviews, opens PRs. |
| [wrapup](wrapup/) | "merge this", "ship it", "finish up" | Completes the plan lifecycle: checks reviews and CI, fixes issues, merges the PR, syncs local main, updates the plan index. |

## Quality and Debugging

| Skill | Trigger | What it does |
|-------|---------|--------------|
| [review](review/) | "review", PR reviews, code quality checks | Systematic code review with explore → reviewer → present pipeline. Covers 12+ languages. Categorizes findings by severity (blocking, important, nit, suggestions). |
| [systematic-debugging](systematic-debugging/) | Any bug, test failure, unexpected behavior | Root-cause-first debugging process. Investigate → analyze → hypothesize → fix. No guessing. Stops after 3+ failed fixes to question architecture. |
| [test-driven-development](test-driven-development/) | Before writing implementation code | RED-GREEN-REFACTOR cycle. Write failing test first, verify it fails, write minimal code to pass, refactor. No production code without a failing test. |
| [verification-before-completion](verification-before-completion/) | Before claiming work is done | Run the verification command. Read the output. Then claim the result. No "should work" or "looks correct" without evidence. |

## Project Operations

| Skill | Trigger | What it does |
|-------|---------|--------------|
| [gitflow-branching](gitflow-branching/) | Starting new work requiring a branch | Trunk-based branching conventions: `main → feature/* or bugfix/* → main`. Always branch from main, keep branches short-lived. |
| [release](release/) | "release", "bump version", "publish vX.Y.Z" | Multi-language semver bumping with GitHub Actions verification. Detects all ecosystem files, checks CI status, applies custom `AGENTS.md` steps, tags and pushes. |
| [daily-summary](daily-summary/) | "daily summary", "standup", "what did I do today" | Scans git history across `~/Coding/AI` and `~/Coding/Ops`, synthesizes a Slack-ready bulleted summary to `/tmp/daily-summary-<date>.txt`. |

## Research and Discovery

| Skill | Trigger | What it does |
|-------|---------|--------------|
| [research](research/) | Comparing libraries, evaluating approaches, deep-dives | Phased research: classify → dispatch parallel researcher subagents → synthesize → hard-stop ask → deliver. Evidence-backed with citations. |
| [librarian](librarian/) | Library internals, "how does X implement Y", source references | Clones repos, searches code with grep, builds GitHub permalinks with commit SHAs. Every claim backed by actual code. |
| [find-skills](find-skills/) | "find a skill for X", "how do I do X" | Searches the open agent skills ecosystem (skills.sh) and helps discover, evaluate, and install new skills. |

## Profile and Documentation

| Skill | Trigger | What it does |
|-------|---------|--------------|
| [create-readme](create-readme/) | "create a README", "write a README" | Surveys the project and writes a comprehensive, well-structured README.md inspired by top open-source projects. |
| [github-profile](github-profile/) | "improve my GitHub profile", "profile README" | Audits and optimizes GitHub profile pages — README, metadata, pinned repos, stats widgets. Scores four categories (x/40 total). |

## Meta

| Skill | Trigger | What it does |
|-------|---------|--------------|
| [writing-skills](writing-skills/) | Creating new skills, editing existing skills | TDD for process documentation. RED-GREEN-REFACTOR cycle applied to skill creation: write failing test (baseline), write skill, verify compliance, close loopholes. |

---

## How Skills Work

1. **Auto-detection** — The agent reads the `description` field in each skill's frontmatter to decide when to load it
2. **Manual reference** — You can explicitly ask the agent to use a skill by name
3. **Chaining** — Skills naturally flow into each other (e.g., `discuss` → `specify` → `implement` → `wrapup`)

## Adding New Skills

Place a new directory under `~/.agents/skills/` with a `SKILL.md` file containing YAML frontmatter (`name` + `description`). See the [writing-skills](writing-skills/) skill for the full TDD-driven creation process.
