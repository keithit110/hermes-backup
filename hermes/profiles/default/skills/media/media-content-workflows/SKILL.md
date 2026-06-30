---
name: media-content-workflows
description: "Class-level umbrella for media content tasks: YouTube transcript extraction, GIF search/download, audio/music workflows, and social-ready transformations."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [media, youtube, transcripts, gifs, audio, music, content]
    related_skills: [visual-design-workflows]
---

# Media Content Workflows

## Overview

Use this umbrella when the task starts from or produces media content: YouTube videos/transcripts, GIF search/download, audio/music generation or analysis, or repackaging media into summaries, threads, blogs, or shareable assets.

## When to Use

- User gives a YouTube URL and wants transcript, summary, chapters, thread, or blog draft.
- User wants a reaction GIF or a GIF downloaded from Tenor-style search.
- User asks for music/audio prompts, generation workflow, spectrogram analysis, or song refinement.
- User wants media transformed into another content format.

## Decision Tree

| Task | Branch | Verify by |
|---|---|---|
| YouTube transcript/content | Transcript pipeline | transcript text/JSON and metadata saved or displayed |
| GIF search/download | GIF retrieval | candidate URLs or downloaded GIF path exists |
| Music/audio creation | Audio workflow | prompt/config/audio path or analysis output exists |
| Repurpose media | Content transform | output format matches requested channel and constraints |

## Shared Rules

- Preserve source URLs and language/format choices.
- Prefer helper scripts for repeatable extraction/downloads.
- Verify downloaded media exists and is the expected format/size.
- For summaries, distinguish transcript facts from interpretation.
- For social formats, respect platform length and style constraints.

## Absorbed Detailed Packages

Detailed original packages and support files are stored under `references/absorbed/<old-skill-name>/`:

- `references/absorbed/youtube-content/SOURCE_SKILL.md`
- `references/absorbed/gif-search/SOURCE_SKILL.md`

## Verification Checklist

- [ ] Source media URL/query recorded.
- [ ] Output file path or final text is provided.
- [ ] Downloads/transcripts were checked before summarizing.
