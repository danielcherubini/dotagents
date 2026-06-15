# When to Stop Guide

Sunk-cost prevention and stopping criteria for research. Know when you have enough.

## Table of Contents

- [Diminishing Returns](#diminishing-returns)
- [Stopping Criteria by Depth](#stopping-criteria-by-depth)
- [Sunk Cost Signals](#sunk-cost-signals)
- [When to Escalate](#when-to-escalate)

---

## Diminishing Returns

Research hits diminishing returns when additional effort yields proportionally less new information.

### Signs of Diminishing Returns

- **Repeating sources** — you're finding the same information from different angles
- **No new facts in 2+ searches** — recent queries haven't added meaningful findings
- **Going deeper on tangents** — you're researching questions that arose from research, not the original question
- **Speculation increasing** — you're filling gaps with inference rather than evidence
- **Time exceeding depth tier** — Quick taking >2 min, Standard >10 min, Deep >30 min

### What to Do

1. **Acknowledge the wall** — note where you stopped and why
2. **Report what you have** — partial findings are better than no findings
3. **Flag the gaps** — be explicit about what remains unknown
4. **Suggest next steps** — how could the user continue if needed?

## Stopping Criteria by Depth

### Quick (< 2 min)

Stop when:
- You have a direct answer with a citation
- You've checked 2 sources and they agree
- You've checked 2 sources and they disagree (report both)

### Standard (5-10 min)

Stop when:
- All research questions have answers (or are flagged as gaps)
- You've covered the primary sources for the topic
- Additional searches are yielding duplicates
- You've hit 3 distinct angles without new information

### Deep (15-30 min)

Stop when:
- All research questions have answers (or are flagged as gaps)
- You've covered primary + secondary sources
- You've checked for contradictions and resolved or flagged them
- The last 2 searches yielded no new information
- You've hit 30 minutes (hard cap — report what you have)

## Sunk Cost Signals

### 🚩 You've Been Researching for 30+ Minutes

**Action:** Stop. Report what you have. Flag remaining gaps. The user can decide to continue.

### 🚩 You've Dispatched 5+ Subagents

**Action:** Pause. Synthesize what you have so far. Present to user before dispatching more.

### 🚩 You're Researching a Question That Arose From Research

**Action:** Step back. Is this question necessary to answer the original question? If not, note it as a tangent and skip.

### 🚩 You're Trying to Verify a Minor Detail

**Action:** If the detail doesn't affect the main conclusions, note it as unverified and move on.

### 🚩 Sources Keep Contradicting Without Resolution

**Action:** After 3 unresolved contradictions on the same topic, flag the topic as "contested" and move on.

## When to Escalate

Escalate to the user (via the hard stop or a progress note) when:

| Situation | Action |
|-----------|--------|
| Research is taking longer than depth tier allows | Present partial findings, ask if user wants to continue |
| Core question can't be answered with available sources | Report the dead end, suggest alternative approaches |
| You need access to paid/privileged sources | Note the gap, explain what a paid source would provide |
| The question itself seems wrong or based on false premises | Present your understanding, ask user to clarify |
| You've hit the 30-minute hard cap | Report what you have, flag gaps |

### Escalation Format

```markdown
## Research Checkpoint

**Status:** [X of Y questions answered]

**Completed:**
- [Question 1] — ✅ Answered
- [Question 2] — ✅ Answered

**Blocked:**
- [Question 3] — ❌ No available sources. Requires [paid access / internal docs / etc.]

**Recommendation:** [Deliver current findings, or continue with specific focus]
```
