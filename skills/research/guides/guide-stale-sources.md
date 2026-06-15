# Stale Sources Guide

Handling paywalled, outdated, and unverifiable sources during research.

## Table of Contents

- [Stale Sources](#stale-sources)
- [Paywalled Sources](#paywalled-sources)
- [Unverifiable Sources](#unverifiable-sources)
- [Archive Strategies](#archive-strategies)
- [Reporting Stale Findings](#reporting-stale-findings)

---

## Stale Sources

A source is stale when its information is likely outdated relative to the research question.

### Detecting Staleness

| Signal | Likely Stale | Likely Current |
|--------|-------------|----------------|
| Date | >2 years old for technical topics | <1 year old, or clearly versioned |
| Version | References old version without noting current | References current version or ranges |
| Links | Broken links, 404s | Working links to current resources |
| Content | Mentions features that no longer exist | Matches current behavior |
| Domain | Defunct project, archived repo | Active project, recent commits |

### What to Do with Stale Sources

1. **Note the date** — always include the publication date in your citation
2. **Check for updates** — search for newer versions of the same information
3. **Cross-reference** — verify against current documentation or sources
4. **Flag in report** — mark as "potentially outdated" if you can't verify

### When Stale Sources Are Still Useful

- Historical context ("how did X evolve?")
- Foundational concepts that don't change
- Academic papers (the methodology remains valid even if old)
- Version-specific questions ("how did this work in v2?")

## Paywalled Sources

### Detecting Paywalls

- Full text not available without subscription
- Abstract-only with "access denied" for full content
- Login required to view content
- "Behind paywall" message

### Workarounds (in order of preference)

1. **Abstract + citations** — the abstract often contains the key claims; the citations point to open sources
2. **Author's website / institutional repository** — authors often post preprints
3. **arXiv / bioRxiv / SSRN** — preprint servers for many fields
4. **Google Scholar** — may link to open-access versions
5. **Wayback Machine** — `web.archive.org` may have cached the content
6. **Contact the author** — note in report that direct contact might help

### Reporting Paywalled Sources

```markdown
## Source: [Title]

**Availability:** ❌ Paywalled (IEEE Xplore subscription required)
**Abstract:** [Summary of what the abstract says]
**Key claims (from abstract):** [What you can verify from the abstract]
**Open alternatives:** [Any open-access versions found]
**Impact on research:** [How the paywall affects your conclusions]
```

## Unverifiable Sources

A source is unverifiable when you cannot independently confirm its claims.

### Common Unverifiable Sources

| Type | Why Unverifiable | What to Do |
|------|-----------------|------------|
| Anonymous blog | No author to verify credentials | Cross-reference claims with known sources |
| Social media post | No editorial review, easily deleted | Treat as lead, not evidence. Find primary source. |
| Internal company doc | Not publicly accessible | Note as "internal source, not independently verified" |
| Oral statement | No written record | Note as "per [person], [date]" — low credibility |
| Deleted content | Wayback has no copy | Note as "source no longer available" |

### Handling Unverifiable Claims

1. **Never cite as fact** — mark clearly as unverified
2. **Try to find a primary source** — the unverifiable source may reference something you can verify
3. **Note the limitation** — explain in your report why the claim can't be verified
4. **Don't let it block research** — if it's a minor point, note and move on

## Archive Strategies

### Web Archives

| Service | Best For | URL Pattern |
|---------|----------|-------------|
| Wayback Machine | General web pages | `web.archive.org/web/<date>/<url>` |
| Google Cache | Recently changed pages | Google search → ⋮ → Cached |
| archive.today | Journalistic content | `archive.is/<url>` |

### Using Archives

```typescript
// Try wayback machine for a broken URL
fetch_content({ 
  url: "https://web.archive.org/web/20240101000000/https://example.com/original" 
})
```

### When Archives Don't Help

- Dynamic content (JS-rendered pages often don't archive well)
- Very recent content (may not be archived yet)
- Paywalled content (archives respect paywalls)
- API endpoints (return different data each call)

## Reporting Stale Findings

### In the Report

```markdown
## Findings

### [Topic]

**Finding:** [What you found]
**Source:** [Citation with date]
**Currency:** ⚠️ Source is from [date] — may be outdated. Cross-referenced with [current source] and [agrees / conflicts].

**Finding:** [What you found]
**Source:** [Citation]
**Currency:** ✅ Source is current (published [date], references [current version]).
```

### In the Gaps Section

```markdown
## Gaps

- **[Topic]** — Only found sources from [year]. Current behavior may differ. 
  Recommendation: Check [current documentation] for updates.
- **[Topic]** — Key source ([title]) is paywalled. Abstract suggests [claim], 
  but full methodology unavailable. Recommendation: Access via [institution / author].
```
