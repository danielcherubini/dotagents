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

### Phase 1: Context Gathering (2-5 minutes)

Before diving into code, understand the full picture:

1. **Read PR description carefully**
   - What problem does it solve?
   - What are the acceptance criteria?
   - Are there linked issues or design docs?

2. **Check PR size**
   - < 100 lines: Full review appropriate
   - 100-400 lines: Full review, focus on critical paths
   - 400-800 lines: Focus on critical paths, suggest splitting
   - > 800 lines: Request split into smaller PRs

3. **Review CI/CD status**
   - Are tests passing?
   - Is linting clean?
   - Are there build warnings?

4. **Understand the business requirement**
   - Who is this for?
   - What's the user impact?
   - What happens if this breaks?

5. **Note architectural decisions**
   - New dependencies?
   - API changes?
   - Database schema changes?
   - Breaking changes for users?

6. **Check related code**
   - How does this interact with existing code?
   - Are there similar patterns elsewhere?
   - Does this follow existing conventions?

### Phase 2: High-Level Review (5-15 minutes)

Start with the big picture before diving into details:

1. **Architecture & Design**
   - Does the solution fit the problem?
   - Are there anti-patterns?
   - Is the design scalable?
   - Consult [Architecture Review Guide](guides/guide-architecture-review.md)

2. **Performance Assessment**
   - Algorithm complexity (Big-O)
   - Memory usage patterns
   - N+1 queries or excessive API calls
   - Unnecessary allocations
   - Consult [Performance Review Guide](guides/guide-performance-review.md)

3. **Security Assessment**
   - Input validation
   - Authentication/authorization
   - Data exposure risks
   - Consult [Security Review Guide](guides/guide-security-review.md)

4. **Testing Strategy**
   - Are tests covering edge cases?
   - Are tests meaningful or just exercising code?
   - Is there test coverage for error paths?

5. **File Organization**
   - Are new files in the right places?
   - Is there logical grouping?
   - Are there unnecessary files?

### Phase 3: Line-by-Line Review (15-30 minutes)

For each changed file, systematically check:

#### Correctness
- Logic errors or off-by-one mistakes
- Edge case handling (empty inputs, null values, boundary conditions)
- Error handling completeness
- Concurrency issues (race conditions, deadlocks, TOCTOU)
- State machine transitions complete and valid

#### Security
- Input validation and sanitization
- SQL injection prevention (parameterized queries)
- XSS prevention (output encoding)
- Authentication/authorization checks
- Sensitive data handling (PII, credentials, tokens)
- CSRF protection (for state-changing operations)
- Rate limiting (for public APIs)
- Dependency security (known vulnerabilities)

#### Performance
- Algorithmic complexity (avoid O(n²) when O(n) is possible)
- Database query efficiency (N+1 queries, missing indexes)
- Memory usage patterns (memory leaks, excessive allocations)
- Unnecessary allocations or copies
- Hot path optimization opportunities
- Lazy evaluation vs eager evaluation

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

### Phase 4: PRESENT FINDINGS AND ASK WHAT TO FIX (REQUIRED — HARD STOP)

> 🛑 **HARD STOP POINT — READ THIS FIRST**
>
> This is the **ONLY** place in the entire review process where you must pause and wait for user input. After presenting findings, your job is DONE until the user responds.
>
> **EXECUTION FLOW (MUST follow this exact sequence):**
> ```
> Phase 3 (Line-by-line review) → Present Summary → call ask() → [STOP — WAIT] → (user responds) → Phase 5
>                                                              ↑
>                                                              └── YOU MUST STOP HERE. NOTHING ELSE.
> ```
>
> **PRE-FLIGHT CHECKLIST (Complete ALL before proceeding):**
> - [ ] I have presented the summary of findings ✅
> - [ ] I have called `ask()` with the fix-priority question ✅
> - [ ] I am NOT about to fix, suggest, or modify any code ✅
> - [ ] I am waiting for user input before doing anything else ✅
>
> **❌ NEVER do any of these:**
> - Present findings and then immediately start fixing issues
> - Say "What would you like to do?" in plain text instead of using `ask()`
> - Begin Phase 5 fixes before receiving a response from `ask()`
> - Assume the user wants everything fixed — let them choose
> - Continue reading code after presenting findings
> - Suggest fixes without waiting for user selection
>
> **✅ ALWAYS do this — IN EXACT ORDER:**
> 1. Present the summary (count and categorize all findings)
> 2. Call `ask()` with the exact question format shown below
> 3. STOP. Wait for the user's response.
>
> **This is a hard break in your execution flow.** You must not proceed to Phase 5 until `ask()` returns with a user answer.
>
> **VIOLATION DETECTION:** If you find yourself about to fix, suggest, or modify code after presenting findings, you have violated this rule. Go back and call `ask()` first.

#### Step 1: Count and categorize all findings

```typescript
// Tally your findings:
let countBlocking = 0;    // 🔴 Must fix before merge
let countImportant = 0;   // 🟡 Should fix, discuss if disagree
let countNit = 0;         // 🟢 Nice to have, not blocking
let countSuggestions = 0; // 💡 Alternative approaches
let countLearning = 0;    // 📚 Educational, no action needed
let countPraise = 0;      // 🎉 Good work, keep it up!
```

#### Step 2: Present a summary of findings FIRST

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

#### Step 3: CALL ask() IMMEDIATELY after the summary

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

#### Step 4: STOP and WAIT for user response

After calling `ask()`, your review is complete. Do nothing else. Do not start fixing. Do not suggest fixes. Wait for the user to respond, then proceed to Phase 5 based on their answer.

**If you skip this step or bypass ask(), you have violated a core rule of the review process.**

### Phase 5: Fix Issues (Based on User Choice)

**⚠️ You may ONLY enter this phase after `ask()` has returned a user response.** If you are here without having called `ask()` first, you have made an error — go back to Phase 4.

Fix issues ONE AT A TIME based on user's selection. For each issue:
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
