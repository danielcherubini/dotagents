# Video Research Strategy

Research strategy for YouTube videos, conference talks, tutorials, and screen recordings.

## Table of Contents

- [When to Use Video Research](#when-to-use-video-research)
- [Full Video Analysis](#full-video-analysis)
- [Targeted Extraction](#targeted-extraction)
- [Citation Formats](#citation-formats)
- [Credibility Assessment](#credibility-assessment)

---

## When to Use Video Research

Video research is appropriate when:

- The user asks about a specific video or talk
- Visual demonstration is important (UI walkthroughs, live coding)
- Conference talks provide context not available in written form
- Tutorials show step-by-step processes
- The user wants to know what happens at a specific moment

## Full Video Analysis

### Extracting Content from Videos

Use `fetch_content` with the video URL:

```typescript
// Full extraction (transcript + visual descriptions)
fetch_content({ url: "https://youtube.com/watch?v=abc" })

// Ask a specific question about the video
fetch_content({
  url: "https://youtube.com/watch?v=abc",
  prompt: "What libraries are imported in this tutorial?"
})
```

### The `prompt` Parameter

Always use the `prompt` parameter for videos — it directs the AI to focus on what you care about:

```typescript
// Good — specific question
fetch_content({
  url: "https://youtube.com/watch?v=abc",
  prompt: "What configuration steps are shown for setting up Docker?"
})

// Bad — generic
fetch_content({ url: "https://youtube.com/watch?v=abc" })
```

## Targeted Extraction

### Single Frame at a Known Moment

When you know the approximate timestamp:

```typescript
// Single frame
fetch_content({
  url: "https://youtube.com/watch?v=abc",
  timestamp: "12:34"
})
```

### Range Scan for Visual Discovery

When you know the approximate area but not the exact moment:

```typescript
// Range extracts evenly-spaced frames
fetch_content({
  url: "https://youtube.com/watch?v=abc",
  timestamp: "12:30-14:00"
})

// Custom density
fetch_content({
  url: "https://youtube.com/watch?v=abc",
  timestamp: "12:30-14:00",
  frames: 6  // 6 frames across the range
})
```

### Whole-Video Sampling

Quick overview of the entire video:

```typescript
// Sample 6 frames across the whole video
fetch_content({
  url: "https://youtube.com/watch?v=abc",
  frames: 6
})
```

### Batch Multiple Videos

```typescript
// Same question across multiple videos
fetch_content({
  urls: [
    "https://youtube.com/watch?v=abc",
    "https://youtube.com/watch?v=def"
  ],
  prompt: "What packages are installed?"
})
```

## Citation Formats

### Video Citations

```markdown
# Format
Video Title — Channel/Author — Platform — URL — Timestamp (if specific claim)

# Examples
"React Conf 2024: The Future of Server Components" — Dan Abramov — YouTube — https://youtube.com/watch?v=abc — 12:34
"Building a Compiler from Scratch" — CompilerCon 2023 — YouTube — https://youtube.com/watch?v=def
```

### Timestamp Conventions

- Use timestamps for specific claims: `12:34` or `12:34-12:45`
- Use full video URL for general references
- Note when a timestamp is approximate

## Credibility Assessment

### Speaker Credibility

| Signal | Credible | Questionable |
|--------|----------|--------------|
| Speaker | Known expert, maintainer, researcher | Unknown, no credentials |
| Venue | Major conference (React Conf, QCon, etc.) | Personal channel, no curation |
| Content | Code-backed claims, demos | Opinion without evidence |
| Date | Recent (for technical topics) | Outdated without disclaimer |

### Video-Specific Red Flags

- Clickbait title vs. actual content mismatch
- No code repository linked
- Claims without demonstration
- Outdated content presented as current
- Sponsored content not clearly labeled

### Cross-Referencing Video Claims

Video content should be cross-referenced when possible:

1. **Code shown in video** — verify against actual repository
2. **Claims about performance** — verify with benchmarks
3. **API usage** — verify against official documentation
4. **Architecture decisions** — verify against RFC or design docs

Videos are a secondary source — always verify against primary sources when the claim is critical.
