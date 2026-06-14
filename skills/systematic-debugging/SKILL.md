---
name: systematic-debugging
description: Use when encountering any bug, test failure, or unexpected behavior, before proposing fixes
---

# Systematic Debugging

Find root cause before attempting fixes. No guessing.

## Process

### 1. Investigate
- Read error messages and stack traces completely
- Reproduce the issue consistently
- Check recent changes (git diff, recent commits)
- Trace the data flow to find where it breaks

### 2. Analyze
- Find working examples of similar code in the codebase
- Compare working vs broken — list every difference
- Check dependencies and assumptions

### 3. Hypothesize and test
- Form one specific hypothesis: "X causes Y because Z"
- Make the smallest possible change to test it
- One variable at a time
- Didn't work? New hypothesis, don't pile fixes

### 4. Fix
- Write a failing test that reproduces the bug
- Fix the root cause (not the symptom)
- Verify the fix and no regressions
- If 3+ fixes failed: stop and question the architecture

## Red flags — stop and go back to step 1
- "Quick fix for now"
- "Just try changing X"
- Proposing fixes before tracing data flow
- Each fix reveals a new problem elsewhere
