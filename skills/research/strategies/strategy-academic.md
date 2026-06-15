# Academic Research Strategy

Research strategy for academic papers, citations, reproducibility, and scholarly investigation.

## Table of Contents

- [Finding Papers](#finding-papers)
- [Evaluating Papers](#evaluating-papers)
- [Citation Formats](#citation-formats)
- [Reproducibility](#reproducibility)
- [Common Pitfalls](#common-pitfalls)

---

## Finding Papers

### Search Sources (by credibility)

1. **DOI lookup** — if you have a DOI, go directly to the paper
2. **arXiv** — preprints for CS, physics, math (check for published version)
3. **Google Scholar** — broad academic search with citation tracking
4. **PubMed** — biomedical and life sciences
5. **IEEE Xplore / ACM Digital Library** — engineering and CS
6. **Semantic Scholar** — AI-powered academic search
7. **University repositories** — institutional archives

### Search Techniques

```
# Good — specific, includes key terms and year range
"transformer architecture attention mechanism 2023..2024"

# Good — author + topic
" Vaswani attention is all you need"

# Good — citation chain
"papers cited by 'BERT: Pre-training of Deep Bidirectional Transformers'"

# Bad — too vague
"machine learning papers"
```

### Following Citation Chains

1. **Forward citations** — who cited this paper? (shows impact, extensions)
2. **Backward citations** — what did this paper cite? (shows foundations)
3. **Key papers** — high-citation papers in a field are entry points
4. **Recent surveys** — survey papers provide field overviews

## Evaluating Papers

### Quick Credibility Check

| Signal | Credible | Questionable |
|--------|----------|--------------|
| Venue | Top-tier conference/journal | Unknown venue, predatory journal |
| Authors | Affiliated with universities/research labs | No affiliation, unknown authors |
| Citations | Cited by other reputable work | Self-citations only, or none |
| Peer review | Double-blind, top venue | No peer review, open access without review |
| Reproducibility | Code/data available, clear methodology | Vague methodology, no artifacts |

### Red Flags

- Predatory journal (check [Cabells Predatory Reports](https://cabells.com/) or community lists)
- No clear methodology section
- Claims that seem too good to be true without strong evidence
- No ablation studies for ML papers
- Results that can't be reproduced from described methodology
- Author conflicts of interest not disclosed

### Understanding a Paper

When researching a paper's claims:

1. **Abstract** — what do they claim?
2. **Introduction** — what problem, why does it matter?
3. **Methodology** — how did they test it?
4. **Results** — what did they find? (look at actual numbers, not just claims)
5. **Limitations** — what do they acknowledge?
6. **Code/Artifacts** — is implementation available?

## Citation Formats

### In-Report Citation

```markdown
# Format
DOI or URL — Authors (Year) — "Title" — Venue

# Examples
doi:10.18653/v1/D19-1379 — Vaswani et al. (2017) — "Attention Is All You Need" — NeurIPS
https://arxiv.org/abs/1810.04805 — Devlin et al. (2018) — "BERT: Pre-training of Deep Bidirectional Transformers" — arXiv preprint
```

### Permalink Construction

For academic papers, use persistent URLs:

| Source | Permalink Format |
|--------|-----------------|
| DOI | `https://doi.org/10.xxxx/xxxxx` |
| arXiv | `https://arxiv.org/abs/YYMM.NNNNN` |
| PubMed | `https://pubmed.ncbi.nlm.nih.gov/{PMID}/` |
| IEEE | `https://ieeexplore.ieee.org/document/{ID}` |
| ACM | `https://dl.acm.org/doi/10.1145/xxxx` |

Never use a PDF download link as the primary citation — use the persistent URL.

## Reproducibility

### Checking Reproducibility

1. **Code available?** — GitHub link in paper or supplementary materials
2. **Data available?** — Dataset link, or clear description of data collection
3. **Environment specified?** — Dependencies, random seeds, hardware
4. **Results match?** — Do reported numbers match running the code?

### Reporting Reproducibility Status

```markdown
## Reproducibility: [Paper Title]

- **Code:** ✅ Available at github.com/owner/repo
- **Data:** ✅ Dataset linked in paper (HuggingFace dataset card)
- **Environment:** ⚠️ Dependencies listed but no Docker/container
- **Results:** ❓ Not verified — code not run
```

## Common Pitfalls

### Cherry-Picking Results

Papers often highlight their best results. Look for:

- Full results tables (not just the best row)
- Statistical significance tests
- Comparison with strong baselines (not straw men)
- Results on standard benchmarks (not custom favorable ones)

### Hype vs. Substance

- "State of the art" claims — check the actual numbers, not just the claim
- "X% improvement" — over what baseline? On what metric?
- "First to do X" — check prior work, especially from adjacent fields

### Preprint vs. Published

- arXiv preprints are not peer-reviewed
- Check if a published version exists (may have corrections)
- Note in your report whether the source is preprint or published

### Temporal Context

- A paper's claims may be outdated by the time you read it
- Always check for more recent work on the same topic
- Note the publication date in your report
