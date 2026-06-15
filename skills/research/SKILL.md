---
name: research
description: |
  Comprehensive research skill with phased workflow for web, code, academic,
  and video research. Uses researcher subagent for web and deep analysis,
  explore subagent for fast local lookups. Every claim backed by cited evidence.
  Use when: comparing libraries or frameworks, evaluating approaches or
  architectures, investigating a technology or topic, finding best practices,
  deep-diving into a subject, doing competitive analysis, or researching
  academic papers. Not for: simple file lookups (use explore directly),
  quick fact checks, or tasks requiring code changes.
---

# Research

Systematic research through phased investigation, evidence-backed findings, and structured reporting.

## When to Use This Skill

- Comparing libraries, frameworks, or technologies
- Evaluating architectural approaches or design patterns
- Investigating how something works under the hood
- Finding best practices for a domain
- Deep-diving into a topic before making decisions
- Competitive or market analysis
- Academic paper research
- Video tutorial or conference talk analysis
- Any question requiring multi-source investigation

## Core Principles

### 1. Evidence Over Opinion

Every claim must be backed by a source — URL, file path, permalink, or video timestamp. Distinguish clearly between:

- **Verified facts** — directly observable from a cited source
- **Reasonable inferences** — logical conclusions from multiple sources
- **Speculation** — educated guesses with no direct evidence

When evidence conflicts, present both sides with credibility assessment. Never silently discard a source — if you disagree with it, explain why.

### 2. Depth Matches the Question

| Depth | Time | When |
|-------|------|------|
| **Quick** | < 2 min | Simple lookup, "where is X?", single fact |
| **Standard** | 5-10 min | "How does X work?", approach comparison |
| **Deep** | 15-30 min | "Should we use X or Y?", architecture research |

Scale your investigation to the stakes of the decision. Don't do deep research for a throwaway question.

### 3. Subagent Selection — Use the Right Tool

| Question type | Use | Why |
|---------------|-----|-----|
| "What files/functions handle X?" | `explore` | Fast, cheap, local-only |
| "Find all usages of Y" | `explore` | grep/glob is all you need |
| "What's the best approach for X?" | `researcher` | Needs web + local |
| "How does library X implement Y?" | `researcher` | Needs web search + docs |
| "Compare A vs B" | `researcher` | Needs web benchmarks, discussions |
| "Deep dive into topic Z" | `researcher` | Multi-angle web + code |
| Hybrid (local files + web docs) | `researcher` | It handles both |

**Key rule:** `explore` for fast local lookups. `researcher` for everything else.

### 4. Citation Standards

Every code-related claim needs a permalink. Every web claim needs a URL.

| Source type | Format | Example |
|-------------|--------|---------|
| Open-source code | GitHub permalink with commit SHA | `github.com/owner/repo/blob/<sha>/path#L10-L20` |
| Private/local code | `repo@<commit>:path:line` or `repo:path:line` (at HEAD) | `myproject@abc123:src/main.ts:42` |
| Web | Full URL + title + date | `https://example.com — "Title" (2024-01-15)` |
| Academic | DOI or persistent URL + authors + year | `doi:10.1234/abc — Smith et al. (2023)` |
| Video | URL + timestamp | `youtube.com/watch?v=abc — 12:34` |
| Local files | Absolute path + line range | `/home/user/project/src/main.ts:10-20` |

**Always use full commit SHAs, not branch names.** Branch links break when code changes.

### 5. Credibility Hierarchy

Default ordering (highest to lowest):

1. **Official source code / docs** — the single source of truth
2. **Peer-reviewed papers / specs** — rigorously vetted
3. **Well-known experts / maintainers** — domain authority
4. **Community discussions** — Stack Overflow, GitHub issues, forums
5. **Blogs / tutorials** — verify against primary sources

**Domain-dependent shifts:**
- **Scientific claims:** papers > everything else
- **Performance benchmarks:** measured benchmarks > opinion
- **Developer experience:** community discussions may outrank maintainer blogs
- **Library usage:** source code > docs > community

Consult [Source Credibility Guide](sources/source-credibility.md) for detailed scoring.

### 6. Deliverable Format

The deliverable is a **structured report**, not a brief. Scale length to depth:

| Depth | Report Scope |
|-------|-------------|
| Quick | Findings + citation (1-2 paragraphs) |
| Standard | Summary + Findings + Sources (1-2 pages) |
| Deep | Full report: Summary, Findings, Evidence, Contradictions, Gaps |

See [Research Report Template](assets/research-report-template.md) for the full format.

## Research Workflow

### Phase 1: Classify (30s)

Before doing anything, classify the request:

1. **Research type:** web, code, academic, video, or hybrid
2. **Depth:** quick, standard, or deep
3. **Subagent(s):** which agent(s) handle which angles

Consult the appropriate strategy guide:
- [Web Research](strategies/strategy-web-research.md) — web, docs, market
- [Code Research](strategies/strategy-code-research.md) — codebase exploration
- [Academic Research](strategies/strategy-academic.md) — papers, citations
- [Video Research](strategies/strategy-video-research.md) — YouTube, talks

**Failure mode:** If classification is ambiguous, ask the user to clarify.

### Phase 2: Plan (1-3 min)

Define the research plan:

1. **Research questions** — 2-5 specific, answerable questions
2. **Sources** — where to find answers (repos, docs, search angles)
3. **Execution order** — parallel (independent angles) vs sequential (dependent)
4. **Subagent tasks** — concrete task descriptions for each dispatch

**Failure mode:** If the question is too vague to plan against, ask the user to narrow scope or pick 1-2 angles.

### Phase 3: Execute (5-20 min)

Dispatch subagent(s) with specific, well-scoped tasks:

```
# Quick — single dispatch
subagent({ agent: "explore", task: "Find all files related to X" })

# Standard — parallel independent angles
subagent({
  tasks: [
    { agent: "researcher", task: "Search web for X best practices" },
    { agent: "explore", task: "Find X implementation in codebase" }
  ]
})

# Deep — sequential with dependencies
subagent({ agent: "researcher", task: "Phase 1: ..." })
# ... use results to inform next dispatch ...
subagent({ agent: "researcher", task: "Phase 2: ..." })
```

**Progress checkpoints (Deep mode only):**
- Start: "Dispatching X subagents for Y angles..."
- Midpoint: "X of Y angles complete, covering [topics]..."
- End: "All subagents returned, moving to synthesis."

**Failure modes during execution:**
- Subagent returns nothing → retry with broader query; if still empty, note as gap
- Web search returns no results → check connectivity, try alternative queries, note gap
- Subagent times out → note timeout, try simpler query or skip angle and note gap

### Phase 4: Synthesize (3-10 min)

Combine findings from all sources into a coherent draft report:

1. **Organize findings** by research question, not by source
2. **Resolve contradictions** — follow the protocol in [Resolving Contradictions Guide](guides/guide-resolving-contradictions.md):
   - Pair conflicting claims side-by-side
   - Identify conflict type (definitional, temporal, scope, measurement)
   - Apply credibility hierarchy — defer to higher-tier, note discrepancy
   - Same-tier conflicts: present both with citations, mark unresolved
   - Never silently pick one and discard the other
3. **Build evidence chain** — every claim linked to its source
4. **Identify gaps** — what remains unanswered or uncertain
5. **Format as draft report** using [Research Report Template](assets/research-report-template.md)

**Failure mode:** If sources conflict irreconcilably, present both with citations and mark as unresolved. Consult [Stale Sources Guide](guides/guide-stale-sources.md) for paywalled or outdated sources.

### Phase 4.5: PRESENT DRAFT AND ASK WHAT TO DO (REQUIRED — HARD STOP)

> 🛑 **HARD STOP POINT — READ THIS FIRST**
>
> This is the **ONLY** place in the entire research process where you must pause and wait for user input. After presenting the draft report, your job is DONE until the user responds.
>
> **EXECUTION FLOW (MUST follow this exact sequence):**
> ```
> Phase 4 (Synthesize) → Present Draft Report → call ask() → [STOP — WAIT] → (user responds) → Phase 5
>                                                              ↑
>                                                              └── YOU MUST STOP HERE. NOTHING ELSE.
> ```
>
> **❌ NEVER do any of these:**
> - Present the draft and then immediately start fixing gaps
> - Begin Phase 5 before receiving a response from `ask()`
> - Assume the user wants everything researched — let them choose
> - Continue researching after presenting the draft
> - Suggest additional research without waiting for user selection
>
> **✅ ALWAYS do this — IN EXACT ORDER:**
> 1. Present the draft report (findings, evidence, contradictions, gaps)
> 2. Call `ask()` with the question format shown below
> 3. STOP. Wait for the user's response.

#### Step 1: Present the draft report

```markdown
## Research Draft

### Executive Summary
[Brief overview of key findings]

### Findings
[Organized by research question, with citations]

### Unresolved Contradictions
[If any — paired claims with citations]

### Gaps
[What remains unanswered]
```

#### Step 2: CALL ask() IMMEDIATELY after the draft

```typescript
ask({
  questions: [{
    id: "research-next",
    question: "Draft report ready. What would you like to do?",
    options: [
      { label: "Deliver as-is" },
      { label: "Targeted deep-dive on specific gaps" },
      { label: "Add a research dimension" },
      { label: "Reframe the question" }
    ],
    multi: true,
    description: `**Deliver as-is:** Present final report including gaps section\n**Targeted deep-dive:** Choose 1-3 specific gaps to close\n**Add dimension:** Add scope not in original plan\n**Reframe:** The question itself needs revision`
  }]
})
```

#### Step 3: STOP and WAIT

After calling `ask()`, your research is paused. Do nothing else. Wait for the user to respond, then proceed to Phase 5 based on their answer.

### Phase 5: Deliver (based on user choice)

**⚠️ You may ONLY enter this phase after `ask()` has returned a user response.**

| User Choice | Action |
|-------------|--------|
| **Deliver as-is** | Present final structured report with all sections |
| **Targeted deep-dive** | Loop back to Phase 3 for specific gaps, then re-synthesize (Phase 4) |
| **Add dimension** | Update plan (Phase 2), execute new angle (Phase 3), re-synthesize (Phase 4) |
| **Reframe** | Return to Phase 1 with new question |

The final report includes:
- Executive Summary
- Findings (organized by question, every claim cited)
- Evidence (source details, credibility assessment)
- Unresolved Contradictions (if any)
- Gaps / What Remains Unknown

Consult [When to Stop Guide](guides/guide-when-to-stop.md) if the research is spiraling or hitting diminishing returns.

## Failure Modes & Recovery

| Failure | Recovery |
|---------|----------|
| Classification ambiguous | Ask user to clarify: web, code, academic, or hybrid? |
| Question too vague to plan | Ask user to narrow scope or pick 1-2 angles |
| Subagent returns nothing | Retry with broader query; if still empty, note as gap in report |
| Web search returns no results | Check connectivity; try alternative queries; note gap |
| Sources conflict irreconcilably | Present both with citations, mark as unresolved contradiction |
| Source is paywalled or stale | Note in report; try archive (wayback, cached); flag as gap |
| User changes direction mid-research | Acknowledge, return to Phase 1 with new question |
| Depth misjudged (too shallow) | User catches at hard stop — they'll choose "dig deeper" |
| Depth misjudged (too deep) | Consult [When to Stop Guide](guides/guide-when-to-stop.md) |
| Subagent times out | Note timeout, try simpler query or skip angle and note gap |

## Quick Reference

- [Research Quality Checklist](checklists/checklist-research-quality.md) — pre-delivery verification
- [Source Credibility Guide](sources/source-credibility.md) — credibility scoring and red flags
- [Source Citation Guide](sources/source-citation.md) — citation formats by source type
- [Resolving Contradictions Guide](guides/guide-resolving-contradictions.md) — protocol for conflicting evidence
- [When to Stop Guide](guides/guide-when-to-stop.md) — sunk-cost prevention
- [Stale Sources Guide](guides/guide-stale-sources.md) — paywalled, outdated, unverifiable sources
- [Research Report Template](assets/research-report-template.md) — structured report format
