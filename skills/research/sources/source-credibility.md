# Source Credibility Guide

Credibility scoring, red flags, and verification strategies for research sources.

## Table of Contents

- [Credibility Scoring](#credibility-scoring)
- [Default Hierarchy](#default-hierarchy)
- [Domain-Specific Hierarchies](#domain-specific-hierarchies)
- [Red Flags](#red-flags)
- [Verification Strategies](#verification-strategies)

---

## Credibility Scoring

Rate each source on a 1-5 scale across four dimensions. Multiply for a composite score (max 25).

### Dimension 1: Authority (1-5)

| Score | Description |
|-------|-------------|
| 5 | Official source — maintainers, authors, documentation owners |
| 4 | Recognized expert — known contributor, published author, speaker at major venues |
| 3 | Competent source — named author with relevant background |
| 2 | Unknown source — no clear credentials, but not obviously unreliable |
| 1 | Questionable source — anonymous, known for misinformation, clear conflict of interest |

### Dimension 2: Evidence (1-5)

| Score | Description |
|-------|-------------|
| 5 | Primary evidence — source code, measured benchmarks, peer-reviewed data |
| 4 | Strong secondary — cites primary sources, shows methodology |
| 3 | Moderate — some evidence but gaps in methodology or data |
| 2 | Weak — opinion with minimal backing, anecdotal |
| 1 | None — pure opinion, no evidence provided |

### Dimension 3: Currency (1-5)

| Score | Description |
|-------|-------------|
| 5 | Current — published within relevant timeframe for the topic |
| 4 | Recent — slightly older but still relevant |
| 3 | Dated — older but foundational (concepts still apply) |
| 2 | Stale — likely outdated for the topic |
| 1 | Obsolete — superseded by newer information |

### Dimension 4: Independence (1-5)

| Score | Description |
|-------|-------------|
| 5 | Independent — no conflict of interest |
| 4 | Mostly independent — minor conflicts disclosed |
| 3 | Some bias — sponsored content labeled, or clear perspective |
| 2 | Biased — vendor content, advocacy, undisclosed sponsorship |
| 1 | Compromised — paid promotion disguised as independent, astroturfing |

### Example Scoring

```
Source: React official docs on useId hook
- Authority: 5 (official docs)
- Evidence: 5 (links to source code, shows examples)
- Currency: 5 (updated for React 18)
- Independence: 4 (Meta maintains React, but docs are factual)
- Composite: 5 × 5 × 5 × 4 = 500 (out of 625 max)

Source: Blog post "Why React is Dead" by unknown author
- Authority: 2 (unknown author)
- Evidence: 2 (opinion with some examples)
- Currency: 3 (recent but topic is opinion)
- Independence: 2 (may be promoting alternative)
- Composite: 2 × 2 × 3 × 2 = 24 (out of 625 max)
```

## Default Hierarchy

Default credibility ordering (highest to lowest):

1. **Official source code / docs** — the single source of truth
2. **Peer-reviewed papers / specs** — rigorously vetted
3. **Well-known experts / maintainers** — domain authority
4. **Community discussions** — Stack Overflow, GitHub issues, forums
5. **Blogs / tutorials** — verify against primary sources

## Domain-Specific Hierarchies

### Library / Framework Usage
1. Source code (highest)
2. Official documentation
3. Maintainer statements (GitHub, Twitter)
4. Community discussions
5. Tutorials / blogs (lowest)

### Performance Questions
1. Measured benchmarks (with methodology)
1. Source code analysis (algorithmic complexity)
3. Expert measurements
4. Anecdotal reports
5. Vendor claims (lowest)

### Scientific / Medical Claims
1. Peer-reviewed meta-analyses (highest)
2. Peer-reviewed primary studies
3. Preprints (arXiv, bioRxiv)
4. Expert commentary
5. Popular science articles (lowest)

### Developer Experience / Opinion
1. Large-scale surveys (State of JS, etc.)
2. Community discussions (broad participation)
3. Maintainer perspectives
4. Individual blog posts
5. Social media hot takes (lowest)

### Security Questions
1. CVE databases, official security advisories (highest)
2. Peer-reviewed security research
3. Security audit reports
4. Community vulnerability reports
5. Blog posts about security (lowest)

## Red Flags

### Author Red Flags
- No name or pseudonym with no verifiable identity
- Claims expertise but no evidence of it
- Known for sensationalism or misinformation
- Clear conflict of interest not disclosed

### Content Red Flags
- No sources cited for factual claims
- Broken links to "evidence"
- Claims that contradict well-established facts without explanation
- Hyperbolic language ("best ever", "revolutionary", "everyone is wrong")
- Fear-mongering without data
- Cherry-picked data (shows only supporting evidence)

### Publication Red Flags
- No date (for time-sensitive topics)
- Predatory journal (check Cabells or community lists)
- Domain looks suspicious (misspellings of known sites)
- Sponsored content not labeled
- Site has no "about" or editorial standards

## Verification Strategies

### Cross-Reference

Always check at least 2 independent sources for important claims:

```
Claim: "Library X is faster than Library Y"
Source A: Blog post with benchmark → Score: 24/625
Source B: Independent benchmark repo → Score: 100/625
Source C: Library X's own benchmark → Score: 60/625 (bias)

Verdict: Source B is most credible. Source C confirms but is biased. 
Source A is too weak to rely on.
```

### Trace to Primary Source

When a source cites another source:

1. Find the primary source
2. Read it directly (don't trust the summary)
3. Check if the citation is taken out of context
4. Use the primary source in your report

### Check for Updates

For time-sensitive topics:

1. Search for newer versions of the same information
2. Check changelogs for behavior changes
3. Check GitHub issues for corrections
4. Note the date of your research in the report
