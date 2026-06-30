---
name: visual-design-workflows
description: "Class-level umbrella for visual/creative artifacts: ASCII art, diagrams, infographics, web prototypes, animation/video pipelines, ComfyUI, and TouchDesigner."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [creative, visual-design, diagrams, infographics, ascii, animation, generative-media]
    related_skills: [creative-web-prototyping]
---

# Visual Design Workflows

## Overview

Use this umbrella when the user asks for a visual artifact or visual production pipeline: ASCII art, Excalidraw diagrams, infographics, web mockups, Manim animations, ComfyUI generations, or TouchDesigner/MCP live visuals.

## When to Use

- Text-to-visual artifacts such as ASCII banners, boxes, image-to-ASCII, or terminal art.
- Architecture/flow/sequence diagrams in hand-drawn Excalidraw JSON.
- Infographics with layout/style systems and structured content.
- Browser-based visual prototypes, p5.js sketches, or design-system mockups.
- Math/algorithm/paper-explainer animations.
- Generative image/video/audio workflows through ComfyUI.
- Real-time visuals or installations through TouchDesigner MCP.

## Decision Tree

| Desired artifact | Branch | Verify by |
|---|---|---|
| Terminal/text art | ASCII art | rendered text/image output visible |
| Diagram | Excalidraw | valid scene JSON + visual layout inspection |
| Infographic | Infographic system | structured content + chosen layout/style |
| Web visual | Web prototype | browser screenshot or local page check |
| Educational animation | Manim | rendered frame/video and script path |
| Generative media pipeline | ComfyUI | workflow JSON, server health, output image/video |
| Live interactive visuals | TouchDesigner | MCP tool success + network/operator inspection |

## Shared Creative Rules

- Ask for or infer medium, audience, aspect ratio, style, and delivery format.
- Prefer concrete artifacts over descriptions: save files and report paths/URLs.
- Verify visually when possible via screenshots, thumbnails, or rendered previews.
- Keep prompts/specs modular so the user can iterate style without rebuilding content.
- Preserve source project files, not just exports.

## Branch Notes

### ASCII and Text Art

Use local text-art tools for deterministic banners and remote/API options only when needed. For image-to-ASCII, verify output dimensions and contrast.

### Diagrams and Infographics

For diagrams, focus on semantic layout, readable labels, and valid scene data. For infographics, first convert content into structured claims/sections, then choose layout and style deliberately.

### Animation and Generative Pipelines

For Manim, ComfyUI, and TouchDesigner, treat the workflow/project file as the source of truth. Run setup/health checks before long renders or live operations.

## Absorbed Detailed Packages

Detailed original packages and support files are stored under `references/absorbed/<old-skill-name>/`:

- `references/absorbed/ascii-art/SOURCE_SKILL.md`
- `references/absorbed/ascii-video/SOURCE_SKILL.md`
- `references/absorbed/baoyu-infographic/SOURCE_SKILL.md`
- `references/absorbed/excalidraw/SOURCE_SKILL.md`
- `references/absorbed/manim-video/SOURCE_SKILL.md`
- `references/absorbed/comfyui/SOURCE_SKILL.md`
- `references/absorbed/touchdesigner-mcp/SOURCE_SKILL.md`

## Verification Checklist

- [ ] Artifact source and export paths are reported.
- [ ] Visual output was inspected when possible.
- [ ] Style/layout choices match the request.
- [ ] Heavy toolchains were health-checked before expensive runs.
