---
name: test-driven-development
description: Use when implementing any feature or bugfix, before writing implementation code
---

# Test-Driven Development

Write the test first. Watch it fail. Write minimal code to pass. Refactor.

## The Cycle

1. **RED** — Write one failing test for the behavior you want
2. **Verify RED** — Run it, confirm it fails for the right reason
3. **GREEN** — Write the simplest code to make it pass
4. **Verify GREEN** — Run it, confirm all tests pass
5. **REFACTOR** — Clean up, keep tests green
6. **Repeat** for next behavior

## Rules

- No production code without a failing test first
- Write code before test? Delete it. Start over.
- One behavior per test, clear name, real code (not mocks)
- Minimal implementation — don't over-engineer

## When stuck

| Problem | Solution |
|---------|----------|
| Don't know how to test | Write the API you wish existed |
| Test too complicated | Design too complicated — simplify |
| Must mock everything | Code too coupled — use dependency injection |
