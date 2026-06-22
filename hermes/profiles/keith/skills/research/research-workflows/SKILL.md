---
name: research-workflows
description: "Class-level umbrella for research discovery, monitoring, market/data-source queries, knowledge-base building, and paper-writing workflows."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [research, arxiv, feeds, prediction-markets, knowledge-base, papers]
    related_skills: [ocr-and-documents]
---

# Research Workflows

## Overview

Use this umbrella for the class of tasks where the agent must discover sources, monitor feeds, query domain data, organize notes, or turn evidence into a research artifact. It unifies paper search, RSS/blog monitoring, prediction-market data, wiki-style knowledge bases, and ML paper writing.

## When to Use

- Search arXiv or pull paper metadata/full text.
- Monitor blogs, RSS, or Atom feeds for new content.
- Query Polymarket or similar public market/data APIs.
- Build or update an interlinked markdown knowledge base.
- Draft, revise, or prepare academic/ML papers for conference submission.

## Decision Tree

| Task | Branch | Output |
|---|---|---|
| Find papers | Literature discovery | citation list, abstracts, PDFs or markdown extracts |
| Track sources over time | Monitoring | feed list, scan results, changed/new item summary |
| Query live markets/data | Data-source query | market IDs, prices/orderbooks/history, timestamped results |
| Build a local corpus | Knowledge base | schema, linked notes, source index |
| Write a paper | Research writing | outline, LaTeX/template files, figures/tables/checklist |

## Shared Research Rules

- Prefer primary sources and preserve URLs/IDs/versions.
- Record query strings, dates, API endpoints, and filters.
- Separate extracted evidence from synthesis.
- Use local files for durable artifacts; report exact paths.
- For current data, timestamp results and avoid implying permanence.

## Branch Notes

### Literature Discovery

Use arXiv by keyword, author, category, or ID. For full papers, pull PDF/HTML and extract text before summarizing.

### Monitoring

Use feed/blog monitoring when the user cares about ongoing change rather than one-shot search. Keep source lists explicit and scan results reproducible.

### Prediction/Data Markets

For Polymarket-style queries, distinguish market metadata, outcome tokens, order books, prices, and historical time series. Parse double-encoded fields carefully.

### Knowledge Bases

For wiki-style work, start by reading the schema/index, then add small linked notes with provenance. Do not create disconnected dumps.

### Paper Writing

For conference papers, preserve template constraints, contribution framing, experiment evidence, citation workflow, reviewer checklists, and reproducibility notes.

## Absorbed Detailed Packages

Detailed original packages and support files are stored under `references/absorbed/<old-skill-name>/`:

- `references/absorbed/arxiv/SOURCE_SKILL.md`
- `references/absorbed/blogwatcher/SOURCE_SKILL.md`
- `references/absorbed/polymarket/SOURCE_SKILL.md`
- `references/absorbed/llm-wiki/SOURCE_SKILL.md`
- `references/absorbed/research-paper-writing/SOURCE_SKILL.md`

## Verification Checklist

- [ ] Source IDs/URLs and query parameters are preserved.
- [ ] Current/live data has a timestamp.
- [ ] Claims distinguish evidence from synthesis.
- [ ] Durable artifacts are saved with exact paths.
