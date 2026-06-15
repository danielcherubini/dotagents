# Research Report Template

Structured format for research deliverables. Scale sections to research depth.

## Table of Contents

- [Quick Report](#quick-report)
- [Standard Report](#standard-report)
- [Deep Report](#deep-report)

---

## Quick Report

For quick lookups (< 2 min, single fact).

```markdown
## Research: [Question]

**Finding:** [Direct answer]
**Source:** [Citation]
**Currency:** [Date note if relevant]
```

### Example

```markdown
## Research: What files handle authentication?

**Finding:** Authentication is handled by three files:
- `src/middleware/auth.ts` — JWT validation middleware
- `src/routes/auth.ts` — login/register API routes  
- `src/services/auth.ts` — password hashing and token generation

**Source:** Local codebase (explore subagent)
**Files:** src/middleware/auth.ts, src/routes/auth.ts, src/services/auth.ts
```

---

## Standard Report

For standard research (5-10 min, multiple sources).

```markdown
## Research: [Question]

### Executive Summary
[2-3 sentence overview of key findings]

### Findings

#### [Sub-topic 1]
[Finding with citations]

#### [Sub-topic 2]
[Finding with citations]

### Sources
1. [Full citation 1]
2. [Full citation 2]
3. [Full citation 3]

### Gaps
- [What remains unknown, if anything]
```

### Example

```markdown
## Research: React vs Vue for a new dashboard project

### Executive Summary
React has larger ecosystem and more job market demand. Vue has simpler 
learning curve and built-in reactivity. For a small team with no React 
experience, Vue may be faster to productive. For long-term hiring and 
ecosystem access, React is stronger.

### Findings

#### Ecosystem Size
React: 200K+ GitHub stars, 18M weekly npm downloads. 
Larger plugin ecosystem (state management, routing, UI libraries).
Source: https://github.com/facebook/react — "React" (212K stars)
Source: https://www.npmjs.com/package/react — 18.2M weekly downloads

Vue: 205K GitHub stars, 85K weekly npm downloads.
Smaller but cohesive ecosystem (official router, pinia, vueuse).
Source: https://github.com/vuejs/core — "Vue" (205K stars)
Source: https://www.npmjs.com/package/vue — 85K weekly downloads

#### Learning Curve
Vue's template syntax and single-file components are easier for 
HTML/CSS-background developers. React requires understanding JSX 
and the component model more deeply.
Source: State of JS 2023 survey — Vue rated #1 for "would use again" 
among beginners.

### Sources
1. https://github.com/facebook/react — "React" (212K stars)
2. https://github.com/vuejs/core — "Vue" (205K stars)
3. https://www.npmjs.com/package/react — 18.2M weekly downloads
4. https://www.npmjs.com/package/vue — 85K weekly downloads
5. https://2023.stateofjs.com — "State of JavaScript 2023"

### Gaps
- No direct performance benchmark between the two for dashboard-specific 
  workloads. Recommendation: prototype a representative component in both.
- Team's specific skill set not factored in. Recommendation: assess 
  team's HTML/CSS/JS proficiency before deciding.
```

---

## Deep Report

For deep research (15-30 min, comprehensive investigation).

```markdown
## Research: [Question]

### Executive Summary
[3-5 sentence overview of key findings and recommendations]

### Findings

#### [Research Question 1]
[Detailed finding with multiple citations]

#### [Research Question 2]
[Detailed finding with multiple citations]

#### [Research Question 3]
[Detailed finding with multiple citations]

### Evidence

| Claim | Source | Credibility | Notes |
|-------|--------|-------------|-------|
| [Claim] | [Citation] | [Score/assessment] | [Any caveats] |

### Unresolved Contradictions

#### [Contradiction Topic]
**Claim A:** [Quote or paraphrase]
Source: [Citation]

**Claim B:** [Quote or paraphrase]  
Source: [Citation]

**Analysis:** [Why this is a contradiction, what type]
**Impact:** [How this affects conclusions]

### Gaps

- **[Topic]** — [Why unavailable, how to continue]
- **[Topic]** — [Why unavailable, how to continue]

### Sources
1. [Full citation 1]
2. [Full citation 2]
3. [Full citation 3]
...

### Methodology
[Optional: describe research approach, search queries, repos cloned, etc.]
```
