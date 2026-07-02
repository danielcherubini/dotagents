---
name: review
description: |
  Provides comprehensive code review guidance for React 19, Vue 3, Rust, TypeScript, Java, Python, C/C++, Go, and WASM.
  Helps catch bugs, improve code quality, and give constructive feedback through systematic analysis.
  Features detailed language-specific guides, severity classification, review techniques, and hard-stop protocol.
  Use when: reviewing pull requests, conducting PR reviews, code review, reviewing code changes,
  establishing review standards, mentoring developers, architecture reviews, security audits,
  checking code quality, finding bugs, giving feedback on code.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash      # Run lint/test/build commands to verify code quality
  - WebFetch  # Look up latest docs and best practices
---

# Koji Review

Transform code reviews from gatekeeping to knowledge sharing through constructive feedback, systematic analysis, and collaborative improvement.

## Architecture — Read This First

```
Main agent (YOU — running this skill)
  ├── Phase 1: Dispatch explore subagent(s) for context gathering
  │     ├── explore: gather files, diff, related code, CI status  ← pure leaf
  │     └── (optional) explore: additional angles (tests, docs)  ← pure leaf
  ├── Phase 2: Synthesize context → classify change, determine scope
  ├── Phase 3: Dispatch reviewer subagent with enriched brief
  │     └── reviewer: reads code, runs checks, returns report     ← pure leaf
  ├── Phase 4: Main agent presents report → calls ask()           ← YOU interact
  ├── Phase 5: (user responds) Main agent fixes selected issues
  └── Phase 6: Re-verify fixes, recommend merge
```

**You (the main agent) own orchestration AND user interaction.** Two subagent types, both pure leaves:

1. **explore** — gathers context: files to review, diffs, related code, test coverage, CI status, docs. Returns a concise context brief.
2. **reviewer** — receives the context brief + review scope from the main agent. Reads code, runs lint/test commands, performs analysis, returns a categorized report.

**Subagents never interact with the user.** The reviewer returns its findings as a report. The main agent presents the report to the user and calls `ask()`. No subagent ever fixes code or calls `ask()`.

### Phase 1: Dispatch explore subagent(s)

Use `subagent` with `agent: "explore"` to gather context. Dispatch one or more in parallel:

```
subagent({
  agent: "explore",
  task: "Gather review context for [target]. Find all changed files, " +
        "related imports/callers, test coverage, and any CI/lint status. " +
        "Return a concise brief: files, scope, languages, and risk areas."
})
```

For large changes, dispatch multiple explore subagents in parallel:
- One for the changed files and their dependencies
- One for tests and CI status
- One for docs and related ADRs/plans

### Phase 2: Synthesize and classify

The main agent collects all explore results, classifies the change (bug fix, feature, refactor, security), determines review depth, and identifies focus areas.

### Phase 3: Dispatch reviewer subagent

Use `subagent` with `agent: "reviewer"` and pass the synthesized context:

```
subagent({
  agent: "reviewer",
  task: "Review the following for [scope]: [files/context from explore]. " +
        "Change type: [classification]. Depth: [quick/standard/deep]. " +
        "Focus areas: [correctness, security, perf, etc.]. " +
        "Follow the review skill reviewer checklist. " +
        "Consult [language] guide. Return a report categorized by severity. " +
        "Do NOT call ask() — just return the report."
})
```

The reviewer subagent **returns a report only** — it never calls `ask()` and never fixes code.

### Phase 4: Present report and ask

The main agent receives the reviewer's report, displays it to the user, and calls `ask()` to let the user choose what to fix. **This is the only point of user interaction.**

### When to use subagents vs. review inline

| Scenario | Approach |
|---|---|
| Single file, < 100 lines, trivial change | Inline review (no subagents) |
| Multiple files or > 100 lines | **explore → reviewer → present → ask** pipeline |
| Cross-cutting concern (security, perf, architecture) | **explore → reviewer** with focused scope |
| User says "review" without specifics | **explore → reviewer → present → ask** pipeline |
| Large change (> 400 lines) | **Multiple explore** in parallel → **reviewer → present → ask** |

## When to Use This Skill

- Reviewing pull requests and code changes
- Establishing code review standards for teams
- Mentoring junior developers through reviews
- Conducting architecture reviews
- Creating review checklists and guidelines
- Improving team collaboration
- Reducing code review cycle time
- Maintaining code quality standards

## Core Principles

### 1. The Review Mindset

**Goals of Code Review:**
1. **Catch bugs and edge cases** — Prevent production issues before they ship
2. **Ensure code maintainability** — Make future changes easier, not harder
3. **Share knowledge** — Spread understanding across the team, reduce bus factor
4. **Enforce standards** — Consistency reduces cognitive load for everyone
5. **Improve design and architecture** — Catch structural problems early
6. **Build team culture** — Reviews are conversations, not verdicts

**Not the Goals:**
- Show off knowledge or prove you're smarter
- Nitpick formatting (use linters/formatters)
- Block progress unnecessarily
- Rewrite code to your personal preference
- A substitute for thorough testing

### 2. Effective Feedback

**Good Feedback is:**
- **Specific** — Point to exact lines, explain the impact
- **Actionable** — Suggest concrete fixes or alternatives
- **Educational** — Explain why, not just what
- **Focused on code** — Never attack the person
- **Balanced** — Praise good work, not just problems
- **Prioritized** — Distinguish must-fix from nice-to-have

```markdown
❌ Bad: "This is wrong."
✅ Good: "This could cause a race condition when multiple users
         access simultaneously. Consider using a mutex here."

❌ Bad: "Why didn't you use X pattern?"
✅ Good: "Have you considered the Repository pattern? It would
         make this easier to test. Here's an example: [link]"

❌ Bad: "Rename this variable."
✅ Good: "[nit] Consider `userCount` instead of `uc` for
         clarity. Not blocking if you prefer to keep it."

❌ Bad: "This is messy."
✅ Good: "This function does 5 different things. Consider extracting
         the validation logic into a separate function for clarity."
```

### 3. Review Depth Guidelines

**Match your review depth to the change:**

| Change Type | Depth | Focus Areas |
|------------|-------|-------------|
| Bug fix (< 50 lines) | Quick | Correctness, edge cases, test coverage |
| Feature addition (< 200 lines) | Standard | Logic, design, tests, docs |
| Major refactor (> 400 lines) | Deep | Architecture, migration safety, rollback plan |
| Security-sensitive code | Thorough | All security checklist items |
| Performance-critical path | Detailed | Algorithm complexity, allocations, memory |
| New dependency | Careful | Security, license, maintenance, alternatives |

**Rule of thumb:** Spend more time on changes that affect more code, touch critical paths, or introduce new dependencies.

### 4. Review Scope

**What to Review (Deep):**
- Logic correctness and edge cases
- Security vulnerabilities (injection, auth, data exposure)
- Performance implications (algorithm complexity, allocations, memory)
- Test coverage and quality (edge cases, meaningful assertions)
- Error handling (complete, informative, recoverable)
- Documentation (README, inline comments, API docs)
- API design and naming (clear, consistent, hard to misuse)
- Architectural fit (patterns, separation of concerns, coupling)
- Concurrency and async safety (race conditions, cancellation safety)

**What Not to Review Manually (delegate to tools):**
- Code formatting (use Prettier, Black, cargo fmt, etc.)
- Import organization (use import sorters)
- Linting violations (use ESLint, clippy, pylint, etc.)
- Simple typos (unless in user-facing text)
- Style preferences that are already codified in linters

## Review Process

See [Architecture](#architecture--read-this-first) for the high-level flow. This section expands each phase with detailed checklists.

### Explore Subagent — What to Gather

The explore subagent collects:
- Changed files and their contents
- Related imports, callers, and dependencies
- Test coverage for affected code
- CI/lint/build status
- PR description, linked issues, design docs

For large changes (> 400 lines), dispatch multiple explore subagents in parallel:
- **explore 1:** Changed files + dependencies + related patterns
- **explore 2:** Tests + CI status + coverage gaps
- **explore 3:** Docs + ADRs + migration notes

### Main Agent — Classification Checklist

After collecting explore results, classify:
1. **Change type** — bug fix, feature, refactor, security, perf
2. **Review depth** — quick (< 100 lines), standard (100-400), deep (> 400)
3. **Focus areas** — correctness, security, performance, architecture
4. **Risk level** — based on affected code paths and test coverage

Checklist:
- [ ] What problem does this solve? (from PR description)
- [ ] What's the PR size? (lines changed, files touched)
- [ ] Are tests passing? Is linting clean?
- [ ] Who is this for? What's the user impact?
- [ ] New dependencies? API changes? Schema changes? Breaking changes?
- [ ] How does this interact with existing code?

### Reviewer Checklist (what the reviewer subagent does)

The reviewer subagent follows this checklist for each changed file:

#### Architecture & Design
- Does the solution fit the problem?
- Are there anti-patterns?
- Is the design scalable?
- Consult [Architecture Review Guide](guides/guide-architecture-review.md)

#### Performance
- Algorithm complexity (Big-O)
- Memory usage patterns
- N+1 queries or excessive API calls
- Unnecessary allocations
- Consult [Performance Review Guide](guides/guide-performance-review.md)

#### Security
- Input validation
- Authentication/authorization
- Data exposure risks
- Consult [Security Review Guide](guides/guide-security-review.md)

#### Correctness
- Logic errors or off-by-one mistakes
- Edge case handling (empty inputs, null values, boundary conditions)
- Error handling completeness
- Concurrency issues (race conditions, deadlocks, TOCTOU)
- State machine transitions complete and valid

#### Testing
- Are tests covering edge cases?
- Are tests meaningful or just exercising code?
- Is there test coverage for error paths?

#### File Organization
- Are new files in the right places?
- Is there logical grouping?
- Are there unnecessary files?

#### Maintainability
- Clear naming conventions (express intent, not implementation)
- Single responsibility principle (one reason to change)
- DRY violations (duplicate logic)
- Comment quality and relevance (why, not what)
- API design (hard to misuse, clear contracts)
- Error messages (helpful for debugging)

#### Documentation
- README updated if needed
- Inline comments explain non-obvious logic
- API docs for public functions/types
- Migration guides for breaking changes
- Changelog entries for user-facing changes

### Main agent presents report and calls ask() — HARD STOP

> 🛑 **HARD STOP POINT — READ THIS FIRST**
>
> After the reviewer subagent returns its report, the **main agent** presents the findings to the user and calls `ask()`. This is the **ONLY** place in the entire review process where execution pauses for user input.
>
> **The reviewer subagent never calls `ask()`.** It returns a report. The main agent presents it and asks.
>
> **EXECUTION FLOW (MUST follow this exact sequence):**
> ```
> reviewer returns report → main agent presents to user → main agent calls ask() → [STOP — WAIT] → (user responds) → main agent fixes
>                                                                                                              ↑
>                                                                                                              └── HARD STOP.
> ```
>
> **Main agent PRE-FLIGHT CHECKLIST (Complete ALL before proceeding):**
> - [ ] I have received the reviewer's report ✅
> - [ ] I have presented the findings to the user ✅
> - [ ] I have called `ask()` with the fix-priority question ✅
> - [ ] I am NOT about to fix, suggest, or modify any code ✅
> - [ ] I am waiting for user input before doing anything else ✅
>
> **❌ NEVER do any of these:**
> - Present findings and then immediately start fixing issues
> - Say "What would you like to do?" in plain text instead of using `ask()`
> - Begin fixes before receiving a response from `ask()`
> - Assume the user wants everything fixed — let them choose
> - Have the reviewer subagent call `ask()` — only the main agent calls `ask()`
>
> **✅ ALWAYS do this — IN EXACT ORDER:**
> 1. Receive reviewer's report
> 2. Present the findings to the user (count and categorize)
> 3. Call `ask()` with the question format below
> 4. STOP. Wait for the user's response.
>
> **This is a hard break in execution.** After `ask()` the main agent waits — nothing happens until the user responds.
>
> **VIOLATION DETECTION:** If you find yourself about to fix, suggest, or modify code after presenting findings, you have violated this rule. Go back and call `ask()` first.

#### Report format

Tally findings from the reviewer's report:

```typescript
// Tally your findings:
let countBlocking = 0;    // 🔴 Must fix before merge
let countImportant = 0;   // 🟡 Should fix, discuss if disagree
let countNit = 0;         // 🟢 Nice to have, not blocking
let countSuggestions = 0; // 💡 Alternative approaches
let countLearning = 0;    // 📚 Educational, no action needed
let countPraise = 0;      // 🎉 Good work, keep it up!
```

#### Present the summary to the user

```markdown
## Code Review Summary

**🔴 Blocking (N):**
1. [Issue description] - `file.ts:42`
   - Impact: [What breaks if unfixed]
   - Fix: [Suggested approach]

**🟡 Important (N):**
1. [Issue description] - `file.ts:100`
   - Impact: [What degrades if unfixed]

**🟢 Nit (N):**
1. [Issue description] - `file.ts:150`
   - Impact: [Minor improvement]

**💡 Suggestions (N):**
1. [Suggestion] - `file.ts:200`
   - Why: [Benefit of change]

**📚 Learning (N):**
1. [Educational note] - `file.ts:250`

**🎉 Praise (N):**
1. [Positive feedback] - `file.ts:300`
```

#### Call ask() immediately after presenting

```typescript
ask({
  questions: [{
    id: "fix-priority",
    question: `Found ${countBlocking + countImportant + countNit + countSuggestions} issues. What would you like to fix?`,
    options: [
      { label: `Fix 🔴 blocking only (${countBlocking})` },
      { label: `Fix 🔴 + 🟡 (${countBlocking + countImportant})` },
      { label: `Fix all (blocking + important + nit)` },
      { label: `Review all findings without fixing` }
    ],
    description: `**🔴 Blocking:** Must fix before merge\n**🟡 Important:** Should fix, discuss if disagree\n**🟢 Nit:** Nice to have, not blocking\n**💡 Suggestions:** Alternative approach to consider\n**📚 Learning:** Educational note, no action needed\n**🎉 Praise:** Good work, keep it up!`
  }]
})
```

#### STOP and wait for user response

After calling `ask()`, the review is complete. Do nothing else. Do not start fixing. Do not suggest fixes. Wait for the user to respond, then proceed to Phase 5 based on their answer.

**If you skip this step or bypass ask(), you have violated a core rule of the review process.**

### Phase 5: Main agent fixes issues (Based on User Choice)

**⚠️ The main agent handles all fixes — the reviewer subagent never fixes code.**

After `ask()` returns with the user's selection, the main agent fixes issues ONE AT A TIME:
1. Explain the problem clearly
2. Show the fix
3. Verify with tests/linting
4. Move to next issue

### Phase 6: Re-review (If Issues Were Fixed)

After fixes are applied:
1. Re-read the fixed code
2. Verify the fix addresses the issue
3. Check for any new issues introduced
4. If issues persist, escalate to user
5. If all clear, recommend merge

## Severity Classification Guide

### 🔴 Blocking — Must Fix Before Merge

Issues that will cause:
- Runtime crashes or panics
- Data loss or corruption
- Security vulnerabilities (injection, auth bypass, data exposure)
- Build failures
- Test failures
- Breaking changes without migration path
- Deadlocks or race conditions in production code

### 🟡 Important — Should Fix, Discuss if Disagree

Issues that will cause:
- Incorrect behavior in edge cases
- Performance degradation (significant)
- Memory leaks
- Missing error handling
- Test coverage gaps for critical paths
- API design that's hard to use correctly
- Code that will be difficult to maintain

### 🟢 Nit — Nice to Have, Not Blocking

Issues that are:
- Minor style inconsistencies
- Unnecessary complexity that could be simplified
- Naming that could be clearer
- Redundant code that could be extracted
- Missing documentation that would help future readers

### 💡 Suggestions — Alternative Approaches

Ideas for:
- Different patterns that might work better
- Libraries or tools that could help
- Architectural improvements
- Performance optimizations
- Code organization improvements

### 📚 Learning — Educational, No Action Needed

Notes for:
- Alternative approaches worth knowing
- Language/framework features not being used
- Best practices that apply elsewhere
- "Nice to know" information

### 🎉 Praise — Good Work, Keep It Up

Recognition for:
- Well-designed components
- Excellent test coverage
- Clear documentation
- Creative solutions to hard problems
- Consistent adherence to patterns

## Review Techniques

### Technique 1: The Checklist Method

Use checklists for consistent, thorough reviews. Start with the language-specific guide, then apply the appropriate domain guide (security, performance, architecture).

### Technique 2: The Question Approach

Instead of stating problems, ask questions that guide the author to the solution:

```markdown
❌ "This will fail if the list is empty."
✅ "What happens if `items` is an empty array? Should we handle that case?"

❌ "You need error handling here."
✅ "How should this behave if the API call fails? Should we retry or show an error?"

❌ "This is too complex."
✅ "This function does validation, transformation, and persistence. Could we split these into separate functions?"
```

### Technique 3: Suggest, Don't Command

Use collaborative language that invites discussion:

```markdown
❌ "You must change this to use async/await"
✅ "Suggestion: async/await might make this more readable. What do you think?"

❌ "Extract this into a function"
✅ "This logic appears in 3 places. Would it make sense to extract it into a shared function?"

❌ "This is a bad pattern"
✅ "Have you considered using the Strategy pattern here? It would make testing easier."
```

### Technique 4: The Impact Framework

For each finding, explain the impact:

```markdown
🟡 **Important: Missing error handling in API endpoint** - `api/users.rs:42`

**Impact:** If the database is unreachable, the API returns a 500 with no context.
Users see a generic error, and we lose visibility into the failure.

**Suggested fix:** Add error context with `.with_context()` and return a proper
error response with a user-friendly message.
```

### Technique 5: The Before/After Pattern

Show concrete before/after examples:

```markdown
💡 **Suggestion: Extract validation logic**

**Before:**
```rust
fn process_order(order: &str) -> Result<()> {
    if order.is_empty() {
        return Err("empty".into());
    }
    if !order.chars().all(|c| c.is_alphanumeric()) {
        return Err("invalid".into());
    }
    // ... 50 more lines
}
```

**After:**
```rust
fn validate_order_id(order: &str) -> Result<()> {
    if order.is_empty() || !order.chars().all(|c| c.is_alphanumeric()) {
        return Err("invalid order ID".into());
    }
    Ok(())
}

fn process_order(order: &str) -> Result<()> {
    validate_order_id(order)?;
    // ... 50 more lines
}
```
```

## Language-Specific Guides

When reviewing code in a specific language/framework, consult the corresponding detailed guide:

| Language/Framework | Reference File | Key Topics |
|-------------------|----------------|------------|
| **React** | [React Guide](languages/lang-react.md) | Hooks, useEffect, React 19 Actions, RSC, Suspense, TanStack Query v5 |
| **Vue 3** | [Vue Guide](languages/lang-vue.md) | Composition API, Reactivity System, Props/Emits, Watchers, Composables |
| **Rust** | [Rust Guide](languages/lang-rust.md) | Ownership/Borrowing, Unsafe Review, Async Code, Error Handling, Cancellation Safety, Testing, Macros |
| **TypeScript** | [TypeScript Guide](languages/lang-typescript.md) | Type Safety, async/await, Immutability, Generics |
| **Python** | [Python Guide](languages/lang-python.md) | Mutable Default Args, Exception Handling, Class Attributes, Context Managers |
| **Java** | [Java Guide](languages/lang-java.md) | Java 21/25 Features, Spring Boot 4, Virtual Threads, Stream/Optional |
| **Go** | [Go Guide](languages/lang-go.md) | Error Handling, goroutine/channel, context, Interface Design |
| **C** | [C Guide](languages/lang-c.md) | Pointers/Buffers, Memory Safety, UB, Error Handling |
| **C++** | [C++ Guide](languages/lang-cpp.md) | RAII, Lifetime, Rule of 0/3/5, Exception Safety |
| **SQL/PostgreSQL** | [SQL/PG Guide](languages/lang-sql-pg.md) | SQL Injection Prevention, EXPLAIN ANALYZE, Indexing, Concurrency |
| **CSS/Less/Sass** | [CSS Guide](languages/lang-css-less-sass.md) | Variable Conventions, !important, Performance Optimization, Responsive Design |
| **Qt** | [Qt Guide](languages/lang-qt.md) | Object Model, Signals/Slots, Memory Management, Thread Safety |
| **WASM/Frontend** | [WASM Guide](languages/lang-wasm.md) | Memory management, JS interop, bundle size, accessibility, browser compat |

## Additional Resources

- [Architecture Review Guide](guides/guide-architecture-review.md) - Architecture design review guide (SOLID, anti-patterns, coupling)
- [Performance Review Guide](guides/guide-performance-review.md) - Performance review guide (Web Vitals, N+1, complexity)
- [Security Review Guide](guides/guide-security-review.md) - Security review guide (OWASP, injection, auth)
- [Code Review Best Practices](guides/guide-code-review-best-practices.md) - Code review best practices
- [Common Bugs Checklist](checklists/checklist-common-bugs.md) - Common bugs by language
- [PR Review Template](assets/pr-review-template.md) - Standard PR review template
- [Review Checklist](assets/review-checklist.md) - Quick reference checklist
