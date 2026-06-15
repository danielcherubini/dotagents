# Web Research Strategy

Research strategy for web-based investigation — documentation, market analysis, best practices, competitive research, and general web content.

## Table of Contents

- [Multi-Query Approach](#multi-query-approach)
- [Documentation Research](#documentation-research)
- [Market Research](#market-research)
- [Source Evaluation](#source-evaluation)
- [Search Techniques](#search-techniques)

---

## Multi-Query Approach

Never rely on a single search query. Use 2-4 varied angles to get comprehensive coverage.

**Good (varied angles):**
```
1. "React vs Vue performance benchmarks 2024"
2. "React vs Vue developer experience comparison"
3. "React ecosystem size vs Vue ecosystem statistics"
```

**Bad (redundant):**
```
1. "React vs Vue"
2. "React vs Vue comparison"
3. "React vs Vue review"
```

**Vary by:**
- Scope (broad overview vs specific feature)
- Recency (latest vs historical context)
- Perspective (developer experience vs performance vs ecosystem)
- Source type (benchmarks vs discussions vs official docs)

## Documentation Research

Official documentation is the highest-credibility source for library/framework questions.

### Priority Order

1. **Official docs** — version-pinned URLs when possible
2. **Source code README** — often more up-to-date than docs
3. **RFC / design docs** — for understanding why decisions were made
4. **Changelog / release notes** — for version-specific behavior
5. **Official blog / announcements** — for roadmap and deprecation notices

### Version Pinning

Always note the version you're researching against. Behavior changes between versions.

```
# Good
React 18 useId hook — https://react.dev/reference/react/useId (v18.3)

# Bad
React useId hook — https://react.dev/reference/react/useId
```

### When Docs Are Missing or Unclear

1. Check the source code directly (use `researcher` to search the repo)
2. Check GitHub issues for discussions about the behavior
3. Check Stack Overflow for community interpretations
4. Note in your report that official docs are incomplete

## Market Research

For competitive analysis, trend research, and market evaluation.

### Competitive Analysis Framework

When comparing products, libraries, or services:

| Dimension | What to look for |
|-----------|-----------------|
| **Features** | Feature parity, unique capabilities, missing features |
| **Performance** | Benchmarks, real-world measurements, not claims |
| **Adoption** | GitHub stars, npm downloads, community size |
| **Maintenance** | Release frequency, issue response time, active contributors |
| **Licensing** | License type, commercial use restrictions |
| **Ecosystem** | Plugins, integrations, community tools |
| **Documentation** | Quality, completeness, examples |
| **Support** | Official support, community forums, paid options |

### Trend Research

When researching trends or "what's popular":

1. Use recency filters — trends change fast
2. Cross-reference multiple sources — don't trust a single metric
3. Distinguish between hype and adoption — GitHub stars ≠ production usage
4. Note the date of your research — trends are time-sensitive

### Source Credibility for Market Research

Market research has lower-credibility sources by default. Be extra careful:

- **Vendor blogs** — biased toward their product. Cross-reference with independent sources.
- **Paid reports** — may have conflicts of interest. Check methodology.
- **Community polls** — small sample sizes, self-selection bias.
- **Download stats** — can be inflated. Look at trends, not absolute numbers.

## Source Evaluation

Every web source needs credibility assessment. See [Source Credibility Guide](../sources/source-credibility.md) for detailed scoring.

### Quick Credibility Check

| Signal | Credible | Questionable |
|--------|----------|--------------|
| Author | Named expert, maintainer | Anonymous, unknown |
| Domain | Official site, well-known publication | Personal blog, unknown domain |
| Date | Recent (or clearly dated historical) | No date, or very old without context |
| Evidence | Links to sources, data, code | Opinion without backing |
| Tone | Measured, acknowledges limitations | Hyperbolic, "best ever", fear-mongering |

### Red Flags

- No author attribution
- No date (for time-sensitive topics)
- No sources cited
- Sponsored content not labeled as such
- Broken links to "evidence"
- Claims that contradict well-established facts without explanation

## Search Techniques

### For `web_search`

- Use 2-4 varied queries, not the same query repeated
- Use `recencyFilter` for time-sensitive topics
- Use `domainFilter` to limit to authoritative sources
- Use `includeContent: true` when you need full page content

### For `fetch_content`

- Use for specific URLs you want to analyze in depth
- Use `prompt` parameter for YouTube videos to focus analysis
- Use `timestamp` for specific moments in videos
- Batch multiple URLs with `urls: [...]` for parallel fetching

### For GitHub Repositories

- Use `fetch_content` with the repo URL to get file tree
- Use `web_search` with `domainFilter: ["github.com"]` for issue/PR search
- Always get the commit SHA for permalinks, not branch names
