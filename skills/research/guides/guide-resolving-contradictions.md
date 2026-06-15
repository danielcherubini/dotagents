# Resolving Contradictions Guide

Protocol for handling conflicting evidence from multiple sources during research.

## Table of Contents

- [Detection](#detection)
- [Classification](#classification)
- [Resolution Protocol](#resolution-protocol)
- [Reporting Unresolved Contradictions](#reporting-unresolved-contradictions)
- [Examples](#examples)

---

## Detection

A contradiction exists when two or more sources make claims that cannot both be true under the same conditions.

### Signs of a Contradiction

- Source A says X, Source B says not-X
- Different numbers for the same metric
- Different recommendations for the same scenario
- Different behavior descriptions for the same function
- Different version requirements for compatibility

### Not a Contradiction

- Different opinions on preference ("I prefer X over Y")
- Different results from different experiments (different conditions)
- Different recommendations for different contexts
- Updates over time (old docs vs new docs — the newer one wins)

## Classification

Classify the contradiction to determine the resolution approach.

| Type | Description | Example |
|------|-------------|---------|
| **Definitional** | Sources use the same term to mean different things | "React Server Component" means different things in different contexts |
| **Temporal** | Sources reflect different points in time | API behavior changed between v1 and v2 |
| **Scope** | Sources cover different subsets of the problem | Benchmark A tests X, Benchmark B tests Y |
| **Measurement** | Sources use different metrics or methodologies | Performance measured differently (throughput vs latency) |
| **Factual** | Sources directly contradict on an observable fact | Library X does/does not support feature Y |

## Resolution Protocol

### Step 1: Pair the Claims Side-by-Side

```markdown
**Claim A:** "React 18 uses automatic batching for all state updates"
Source: React docs (react.dev), accessed 2024-01-15

**Claim B:** "React 18 only batches inside event handlers"
Source: Blog post by @dev123, 2023-06-01
```

### Step 2: Identify the Conflict Type

In the example above: **Temporal** — the blog post (2023-06) predates a React 18 update that expanded batching. The docs (accessed 2024-01) reflect the current behavior.

### Step 3: Apply the Credibility Hierarchy

Apply the default or domain-appropriate credibility hierarchy from [Source Credibility](../sources/source-credibility.md):

1. **Higher-tier source wins** — if Source A is official docs and Source B is a blog, defer to docs
2. **Note the discrepancy** — even when one source clearly wins, mention the contradiction

### Step 4: Handle Same-Tier Conflicts

When sources are at the same credibility tier:

1. **Present both** with full citations
2. **Explain the conflict** — why might they differ?
3. **Mark as unresolved** — don't silently pick one
4. **Suggest verification** — how could the user verify?

```markdown
## Unresolved Contradiction: Batching Behavior

**Claim A:** "React 18 uses automatic batching for all state updates"
Source: [React docs](https://react.dev) (official documentation)

**Claim B:** "React 18 only batches inside event handlers"  
Source: [Blog post](https://example.com) by @dev123 (community blog)

**Resolution:** The React docs are the higher-credibility source. The blog post likely reflects an earlier React 18 alpha behavior. Automatic batching was expanded in React 18.0.0 (released 2022-03-29).

**Verdict:** Claim A is correct for React 18.0.0+. Claim B was correct for earlier alphas.
```

### Step 5: Never Silently Discard

Even when you're confident about which source is correct:

- **Always mention** the contradictory source
- **Always explain** why you deferred to one over the other
- **Always cite** both sources

## Reporting Unresolved Contradictions

When a contradiction truly cannot be resolved (same-tier sources, no way to verify):

```markdown
## Unresolved Contradiction

**Topic:** [What the contradiction is about]

**Claim A:** "[Quote or paraphrase]"
Source: [Full citation]

**Claim B:** "[Quote or paraphrase]"
Source: [Full citation]

**Analysis:** [Why this is a contradiction, what type, why it can't be resolved]

**Impact:** [How this affects the research conclusions]

**Recommendation:** [How the user could verify, or how to proceed despite the uncertainty]
```

## Examples

### Example 1: Temporal (Resolved)

```markdown
**Claim A:** "TypeScript 5.0 introduces decorators"
Source: TypeScript handbook (official docs)

**Claim B:** "TypeScript doesn't support decorators"
Source: Stack Overflow answer from 2019

**Resolution:** Temporal. Claim B was correct at the time (2019) but decorators were 
added in TypeScript 2.3 (experimental) and standardized in 5.0. Claim A is current.
```

### Example 2: Definitional (Resolved)

```markdown
**Claim A:** "Vue 3 uses the Composition API"
Source: Vue docs — refers to the `setup()` function and composables pattern

**Claim B:** "Vue 3 uses the Options API"
Source: Tutorial — refers to `data()`, `methods`, `computed` objects

**Resolution:** Definitional. Both are true — Vue 3 supports both APIs. The docs 
describe the newer Composition API; the tutorial uses the legacy Options API.
```

### Example 3: Factual (Unresolved)

```markdown
**Claim A:** "Library X has 50K weekly downloads"
Source: npm trends (fetched 2024-01-15)

**Claim B:** "Library X has 30K weekly downloads"
Source: Library's README (last updated 2023-09-01)

**Resolution:** Both sources are credible. The README is stale (4 months old). 
npm trends is live data. Likely the README hasn't been updated.

**Verdict:** Claim A is more likely correct (live data vs stale snapshot).
**Recommendation:** Check npm directly for current numbers.
```
