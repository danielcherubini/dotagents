# Source Citation Guide

Citation formats for every source type, with examples and permalink construction.

## Table of Contents

- [Code Sources](#code-sources)
- [Web Sources](#web-sources)
- [Academic Sources](#academic-sources)
- [Video Sources](#video-sources)
- [Local Files](#local-files)
- [General Rules](#general-rules)

---

## Code Sources

### Open-Source Repositories (GitHub)

**Format:** `https://github.com/<owner>/<repo>/blob/<commit-sha>/<path>#L<start>-L<end>`

**Getting the commit SHA:**
```bash
cd /tmp/pi-github-repos/owner/repo && git rev-parse HEAD
# or for a specific tag:
gh api repos/owner/repo/git/refs/tags/v1.0.0 --jq '.object.sha'
```

**Examples:**
```
https://github.com/facebook/react/blob/9d7d9e0df8704b8d3f96b85e0fdbe6f65e067628/packages/react/src/ReactHooks.ts#L42-L50
https://github.com/torvalds/linux/blob/abc123/drivers/net/ethernet/example.c#L100-L120
```

**Rules:**
- Always use full commit SHAs, never branch names
- Include line ranges for specific claims
- For file-level references, omit `#L` suffix

### GitLab / Bitbucket

**Format:** `https://gitlab.com/<group>/<repo>/-/blob/<commit-sha>/<path>#L<start>-L<end>`

**Rules:** Same as GitHub — full SHA, line ranges, never branches.

### Private / Local Repositories

**Format:** `repo@<commit>:<path>:<line>` or `repo:<path>:<line>` (at HEAD)

**Examples:**
```
myproject@abc123def:src/auth/login.ts:42
myproject:config/database.ts:10-20
```

**Rules:**
- Use short commit hash (7+ chars) for specific revisions
- Omit commit hash for "at HEAD" references
- Include line ranges for specific claims

### Code Snippets (Gists, Pastes)

**Format:** URL + note that it's a snippet, not a maintained repo

**Example:**
```
https://gist.github.com/user/abc123 — "Example implementation" (gist, not maintained)
```

## Web Sources

### General Web Pages

**Format:** `URL — "Title" (Date)`

**Examples:**
```
https://react.dev/reference/react/useId — "useId" (accessed 2024-01-15)
https://stackoverflow.com/questions/12345 — "How to implement X in TypeScript" (2023-06-01)
```

**Rules:**
- Include access date for pages that may change
- Include publication date for articles/blogs
- Include page title for context

### Documentation Sites

**Format:** `URL — "Title" (Version, Date)`

**Examples:**
```
https://docs.rs/tokio/1.35.0/tokio/ — "tokio v1.35.0" (rustdocs, 2024-01-01)
https://nodejs.org/api/fs.html — "fs - File System" (Node.js 20.x)
```

**Rules:**
- Always include version when available
- Prefer version-pinned URLs over root URLs

### Stack Overflow / Q&A Sites

**Format:** `URL — "Question Title" (Answer date, Author)`

**Example:**
```
https://stackoverflow.com/a/12345 — "How to implement X" (2023-06-01, @user123)
```

## Academic Sources

### DOI-Based

**Format:** `doi:<doi> — Authors (Year) — "Title" — Venue`

**Example:**
```
doi:10.18653/v1/D19-1379 — Vaswani et al. (2017) — "Attention Is All You Need" — NeurIPS 2017
```

### arXiv

**Format:** `https://arxiv.org/abs/<id> — Authors (Year) — "Title"`

**Example:**
```
https://arxiv.org/abs/1810.04805 — Devlin et al. (2018) — "BERT: Pre-training of Deep Bidirectional Transformers"
```

### Other Academic Databases

**Format:** `Persistent URL — Authors (Year) — "Title" — Journal/Conference`

**Example:**
```
https://pubmed.ncbi.nlm.nih.gov/12345678/ — Smith et al. (2023) — "Study Title" — Nature Medicine
```

**Rules:**
- Always use persistent URLs (DOI, arXiv ID, PubMed ID)
- Never use PDF download links as primary citation
- Note whether source is preprint or published

## Video Sources

### YouTube

**Format:** `"Title" — Channel — YouTube — URL — Timestamp`

**Examples:**
```
"React Conf 2024: The Future of Server Components" — React — YouTube — https://youtube.com/watch?v=abc — 12:34
"Building a Compiler from Scratch" — CompilerCon — YouTube — https://youtube.com/watch?v=def
```

**Rules:**
- Include timestamp for specific claims
- Omit timestamp for general references
- Note channel name for credibility context

### Conference Talks (Non-YouTube)

**Format:** `"Title" — Speaker — Conference — URL — Timestamp`

**Example:**
```
"Keynote: The Future of Web" — Jane Doe — CSS Conf Aus — https://vimeo.com/abc — 0:45:00
```

## Local Files

### Project Files

**Format:** `absolute/path/to/file:line-range`

**Examples:**
```
/home/user/project/src/auth/login.ts:42-50
/home/user/project/config/database.ts:10
```

**Rules:**
- Use absolute paths
- Include line ranges for specific claims
- For file-level references, single line number or no line number

### Generated / Temporary Files

**Format:** `path — description (generated, not persistent)`

**Example:**
```
/tmp/analysis/output.json — benchmark results (generated during research, not persistent)
```

## General Rules

1. **Every claim needs a citation** — no uncited claims in the report
2. **Use permalinks for code** — commit SHAs, not branches
3. **Include dates** — for time-sensitive topics, always note when the source was published or accessed
4. **Be consistent** — use the same format throughout the report
5. **Prefer persistent URLs** — DOIs, permalinks, archive links over generic URLs
6. **Note accessibility** — mark paywalled or unavailable sources
