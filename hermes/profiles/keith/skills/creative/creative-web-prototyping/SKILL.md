---
name: creative-web-prototyping
description: "Design and prototype visual web artifacts: HTML mockups, architecture diagrams, popular design-system references, p5.js sketches, Pretext demos, and DESIGN.md specs."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [creative, web-design, prototypes, diagrams, p5js, pretext, design-systems]
    related_skills: []
---

# Creative Web Prototyping

## Overview

Use this umbrella when the deliverable is a visual browser-based artifact or design spec rather than production application logic. It unifies quick HTML mockups, high-polish landing pages, architecture diagrams, design-system references, p5.js sketches, Pretext demos, and DESIGN.md token/spec work.

## When to Use

- Create 2-3 throwaway HTML mockup variants for comparison.
- Build a polished one-off landing page, prototype, or deck-like browser artifact.
- Draw architecture/cloud/infra diagrams as SVG/HTML.
- Borrow visual language from real design systems and websites.
- Create p5.js generative/interactive sketches.
- Build Pretext browser demos.
- Author or validate DESIGN.md token specs.

## Decision Tree

- **Fast comparison:** sketch multiple simple variants first.
- **Polished artifact:** use a design-system reference and implement one focused HTML/CSS/JS file.
- **Architecture/infra:** use SVG/HTML diagram patterns with labels and dark-theme clarity.
- **Generative/interactivity:** use p5.js or Pretext depending on the requested interaction style.
- **Formal design tokens:** use DESIGN.md spec workflow.

## Workflow

1. Clarify artifact purpose, audience, aspect ratio, and delivery format.
2. Pick a visual reference or diagram/prototype modality.
3. Build a real artifact, not just a description.
4. Run it locally or render/screenshot when possible.
5. Iterate on visible issues and deliver the file/path/media.

## Pitfalls

- Overbuilding production architecture for a throwaway visual.
- Mixing too many design references in one artifact.
- Delivering code without visually checking it.
- Using canvas/WebGL when static HTML/SVG would communicate better.

## Verification Checklist

- [ ] Artifact exists on disk or is attached.
- [ ] Visual output was opened/rendered/screenshot-tested where possible.
- [ ] User can reuse the file without hidden dependencies.
- [ ] Design reference or modality is named in the summary.
