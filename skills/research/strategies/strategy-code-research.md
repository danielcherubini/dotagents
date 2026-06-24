# Code Research Strategy

Research strategy for codebase exploration — understanding existing code, tracing execution paths, finding patterns, and investigating implementations.

## Table of Contents

- [Local Codebase Research](#local-codebase-research)
- [External Library Research](#external-library-research)
- [Code Tracing](#code-tracing)
- [Pattern Identification](#pattern-identification)
- [Subagent Usage](#subagent-usage)

---

## Local Codebase Research

Use the `explore` subagent for fast local lookups. It's cheap, fast, and read-only.

### When to Use `explore`

| Task | Agent |
|------|-------|
| "What files handle authentication?" | `explore` |
| "Find all usages of `useState`" | `explore` |
| "Where is the config loaded?" | `explore` |
| "What does `processOrder` do?" | `explore` |
| "How is error handling done?" | `explore` |

### Effective `explore` Tasks

Be specific in your task descriptions:

```
# Good — specific, answerable
"Find all files that import or reference 'auth' or 'authentication'. 
 Report file paths and relevant line numbers."

# Bad — too vague
"Tell me about auth"

# Good — scoped tracing
"Trace the execution path of the 'login' function. 
 Start from the API route handler and follow through to the database call.
 Report each file and function in the chain."

# Bad — open-ended
"Explore the codebase"
```

### Search Order for Local Research

1. **Start targeted** — search for specific function names, imports, keywords
2. **Broaden if needed** — if specific search finds nothing, try concept names
3. **Read key files** — once you locate relevant files, read them for context
4. **Trace dependencies** — follow imports and function calls to understand flow

## External Library Research

Use the `researcher` subagent for external library research. It combines web search with code analysis.

### Library Research Workflow

1. **Clone the repo** — `fetch_content` with the GitHub URL
2. **Read the README** — overview, installation, basic usage
3. **Search the source** — `grep` for the function or feature in question
4. **Read key files** — locate and read the implementation
5. **Get commit SHA** — for permalink construction
6. **Check issues/PRs** — for context on design decisions

### Understanding an Implementation

When researching how a library implements something:

1. **Find the entry point** — public API, exported function
2. **Trace the call chain** — follow function calls to understand the full path
3. **Identify key data structures** — what state is maintained, how it flows
4. **Note design patterns** — what patterns are used and why
5. **Check edge cases** — how does it handle errors, empty inputs, boundaries

### Constructing Permalinks

```bash
# Get the commit SHA
cd /tmp/pi-github-repos/owner/repo && git rev-parse HEAD

# Construct permalink
https://github.com/owner/repo/blob/<sha>/path/to/file#L10-L20
```

Always use full commit SHAs, not branch names. Branch links break when code changes.

## Code Tracing

### Tracing an Execution Path

1. **Start at the entry point** — API route, event handler, exported function
2. **Follow each call** — note file, function, line number
3. **Branch points** — note conditionals that affect the path
4. **Async boundaries** — note `await`, callbacks, event emitters
5. **External calls** — note API calls, database queries, file I/O

### Report Format for Tracing

```markdown
## Execution Path: `login`

1. `src/routes/auth.ts:15` — POST /login handler receives request
2. `src/services/auth.ts:42` — `authenticate(email, password)` called
3. `src/services/auth.ts:45` — validates input format
4. `src/services/auth.ts:52` — queries database for user by email
5. `src/services/auth.ts:58` — compares password hash with bcrypt
6. `src/services/auth.ts:65` — generates JWT token
7. `src/routes/auth.ts:22` — returns token in response
```

## Pattern Identification

### Finding Architectural Patterns

When researching "how is X done in this codebase":

1. **Search for the concept** — grep for keywords related to the pattern
2. **Find 2-3 examples** — don't stop at the first match
3. **Compare implementations** — are they consistent or ad-hoc?
4. **Note variations** — different approaches for different contexts
5. **Identify the convention** — what's the dominant pattern?

### Common Patterns to Look For

| Pattern | What to search for |
|---------|-------------------|
| Error handling | `try/catch`, `throw`, error types, error middleware |
| State management | state stores, context, reducers, signals |
| Data access | repository pattern, ORM usage, direct queries |
| Authentication | auth middleware, token validation, session management |
| Configuration | env vars, config files, feature flags |
| Testing | test files, mocks, fixtures, test utilities |

## Angle Design for the Main Agent

This strategy is read by the **main agent** when planning research angles. The main agent dispatches all subagents. Subagents do not re-read this strategy or dispatch their own children.

### `explore` angles — fast local lookup

Use for: finding files, tracing imports, reading specific paths. Cheap and fast.

```typescript
// Call each as a SEPARATE subagent tool call in one response:
subagent({ agent: "explore", task: "Find all files that reference 'database' or 'db' in src/. Report file paths and line numbers." })
subagent({ agent: "explore", task: "Trace the login() call chain from the API route to the DB call. Report each file:line." })
```

### `researcher` angles — external library or hybrid web+local

Use for: external library internals, web docs, or hybrid questions.

```typescript
// Call each as a SEPARATE subagent tool call in one response:
subagent({ agent: "researcher", task: "Find how TanStack Query implements stale time checking. Clone the repo, find the implementation, return permalinks." })
subagent({ agent: "researcher", task: "Search web for TypeScript error handling best practices. Return top 3 patterns with citations." })
```

**Each separate `subagent(...)` call in the same response runs in parallel and gets its own visible UI block. Never use `tasks:[...]` — that hides all children inside one block.**
