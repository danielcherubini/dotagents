# Quick Review Checklist

Use this as a fast reference during code reviews.

## Universal Checks

### Code Quality
- [ ] No duplicate code
- [ ] Clear variable and function names
- [ ] Functions are small and focused
- [ ] No dead code or commented-out blocks
- [ ] Error handling is appropriate

### Security
- [ ] No hardcoded secrets or API keys
- [ ] Input validation present
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (sanitized output)
- [ ] Authentication/authorization checks in place

### Testing
- [ ] Tests added for new functionality
- [ ] Edge cases covered
- [ ] Tests are deterministic (no flaky tests)
- [ ] Test names describe behavior, not implementation

### Performance
- [ ] No unnecessary allocations in hot paths
- [ ] Database queries optimized (no N+1)
- [ ] Large responses paginated
- [ ] Caching considered where appropriate

## Language-Specific Quick Checks

### TypeScript/JavaScript
- [ ] No `any` types (use `unknown` + type guards)
- [ ] Promise rejections handled
- [ ] No memory leaks (event listeners, timers cleaned up)

### Python
- [ ] Type annotations present
- [ ] No mutable default arguments
- [ ] Exceptions caught specifically
- [ ] Files closed with `with` statement

### Rust
- [ ] No unnecessary `.clone()` calls
- [ ] Every `unsafe` block has `SAFETY` comment
- [ ] Error handling uses `Result`, not `unwrap()` in production
- [ ] Lifetimes documented where needed

### React
- [ ] Hooks rules followed
- [ ] useEffect dependencies complete
- [ ] No unnecessary re-renders (React.memo, useMemo, useCallback)

## Severity Labels Guide

| Label | Meaning | Action Required |
|-------|---------|-----------------|
| 🔴 `[blocking]` | Must fix before merge | Block PR |
| 🟡 `[important]` | Should fix, discuss if disagree | Address or discuss |
| 🟢 `[nit]` | Nice to have, not blocking | Optional |
| 💡 `[suggestion]` | Alternative approach | Consider |
| 📚 `[learning]` | Educational comment | No action needed |
| 🎉 `[praise]` | Good work, keep it up! | None |
