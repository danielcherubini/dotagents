# Code Review Best Practices

Comprehensive guide to conducting effective code reviews that improve code quality without slowing down development. Based on 2025 research and industry best practices.

## Table of Contents

- [Review Mindset](#review-mindset)
- [Before You Review](#before-you-review)
- [During the Review](#during-the-review)
- [After the Review](#after-the-review)
- [Common Pitfalls](#common-pitfalls)
- [Review Metrics](#review-metrics)

---

## Review Mindset

### Goals of Code Review

1. **Improve code quality** - Catch bugs, suggest improvements
2. **Share knowledge** - Spread understanding across the team
3. **Enforce standards** - Ensure consistency with team conventions
4. **Mentor developers** - Help junior developers learn best practices
5. **Reduce bus factor** - Multiple people understand each part of the codebase

### What Code Review Is NOT

- A performance evaluation tool
- An opportunity to nitpick formatting (use linters)
- A chance to rewrite code to your preference
- A gatekeeping mechanism to block progress

### Effective Feedback Principles

**Be specific and actionable:**
```markdown
❌ "This is wrong."
✅ "This could cause a race condition when multiple users access simultaneously. Consider using a mutex here."
```

**Focus on the code, not the person:**
```markdown
❌ "You forgot to handle errors."
✅ "This function doesn't handle network errors. Should we add error handling?"
```

**Be balanced:**
```markdown
❌ Only finding problems
✅ "Great approach with the strategy pattern here! One suggestion: consider adding input validation on line 42."
```

---

## Before You Review

### 1. Understand the Context

- Read the PR description carefully
- Check linked issues and requirements
- Understand the business goal
- Note any architectural decisions mentioned

### 2. Assess PR Size

| Size | Lines of Code | Recommendation |
|------|--------------|----------------|
| Small | < 100 lines | Full review |
| Medium | 100-400 lines | Full review |
| Large | 400-800 lines | Focus on critical paths, suggest splitting |
| Very Large | > 800 lines | Request split into smaller PRs |

### 3. Check CI/CD Status

- Are tests passing?
- Is linting clean?
- Are there any build warnings?

---

## During the Review

### High-Level Checks First

1. **Does it solve the problem?** - Verify against requirements
2. **Is the design sound?** - Architecture, patterns, scalability
3. **Are tests adequate?** - Coverage, edge cases, meaningful assertions
4. **Is documentation updated?** - README, inline comments, API docs

### Line-by-Line Review

For each change, consider:

#### Correctness
- Logic errors or off-by-one mistakes
- Edge case handling (empty inputs, null values, etc.)
- Error handling completeness
- Concurrency issues (race conditions, deadlocks)

#### Security
- Input validation and sanitization
- SQL injection prevention
- XSS prevention
- Authentication/authorization checks
- Sensitive data handling

#### Performance
- Algorithmic complexity
- Database query efficiency (N+1 queries?)
- Memory usage patterns
- Unnecessary allocations or copies

#### Maintainability
- Clear naming conventions
- Single responsibility principle
- DRY violations
- Comment quality and relevance

---

## After the Review

### Summarize Findings

Group issues by severity:

```markdown
## Summary

🔴 **Blocking (2):**
1. SQL injection vulnerability in user search
2. Race condition in concurrent payment processing

🟡 **Important (3):**
1. Missing error handling in API endpoint
2. N+1 query in user listing
3. Test coverage below threshold

🟢 **Nit (5):**
1-5. Minor style and naming suggestions...

💡 **Suggestions (2):**
1. Consider using strategy pattern for payment processing
2. Add input validation middleware
```

### Follow Up

- Address reviewer comments promptly
- Ask clarifying questions when needed
- Update PR based on feedback
- Celebrate good reviews!

---

## Common Pitfalls

### For Reviewers

| Pitfall | Solution |
|---------|----------|
| Reviewing too much at once | Break large PRs, review incrementally |
| Focusing only on style | Use linters for formatting, focus on logic |
| Being too critical | Balance criticism with praise |
| Ignoring the big picture | Start with architecture before line details |
| Taking too long to respond | Set SLAs (e.g., 24-hour review time) |

### For Authors

| Pitfall | Solution |
|---------|----------|
| Submitting large PRs | Keep PRs under 400 lines when possible |
| Ignoring reviewer feedback | Address all comments, even if disagreeing |
| Not writing tests | Tests are part of the PR, not optional |
| Rushing reviews | Quality over speed for code review |

---

## Review Metrics

### Track These (But Don't Game Them)

| Metric | Purpose | Healthy Range |
|--------|---------|---------------|
| PR size | Manage review effort | < 400 lines |
| Review time | Measure team responsiveness | < 24 hours |
| Comments per PR | Identify complex changes | Varies by complexity |
| Re-review rate | Quality of initial review | < 30% |
| First-time approval rate | Author preparation quality | > 50% |

### Remember

Metrics should help improve the process, not punish individuals. Focus on trends and systemic improvements.

---

## Reference Resources

- [Google's Code Review Guide](https://google.github.io/eng-practices/review/)
- [GitHub's Pull Request Best Practices](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-pull-requests)
- [Stripe's Code Review Guidelines](https://stripe.com/blog/code-review-at-stripe)
