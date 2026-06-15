---
name: audio-music-workflows
description: "Create, analyze, and refine music/audio workflows: songwriting, AI music prompts, AudioCraft/HeartMuLa generation, and spectrogram feature extraction."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [audio, music, songwriting, audiocraft, heartmula, spectrograms, generation]
    related_skills: []
---

# Audio and Music Workflows

## Overview

Use this umbrella for music and audio tasks spanning creative writing, AI music prompting, model-based generation, and audio feature visualization. Pick the path based on the requested outcome: lyrics/prompt craft, full song generation, local model generation, or analysis/visualization.

## When to Use

- Write or refine lyrics, song structure, rhyme, meter, emotional arc, or Suno-style prompts.
- Generate music/sound locally with AudioCraft/MusicGen/AudioGen.
- Run HeartMuLa for Suno-like generation from lyrics and tags.
- Produce spectrograms or audio features such as mel, chroma, or MFCC.

## Decision Tree

1. **Songwriting/prompting:** start with structure, genre, emotional arc, lyrics, and bracketed meta-tags.
2. **Hosted-style song generation:** use HeartMuLa when the user wants lyrics+tags-to-song and local hardware can support it.
3. **Research/local generation:** use AudioCraft for MusicGen/AudioGen text-to-music/text-to-sound experiments.
4. **Analysis/visualization:** use SongSee for spectrograms and features.

## Songwriting and AI Prompt Craft

- Choose a structure before filling sections.
- Keep syllable stress and singability in mind.
- Put performance/arrangement cues in bracketed tags when targeting AI music tools.
- Make genre/style fields concrete: instrumentation, tempo, vocal style, mood, production era.

## Generation Workflows

- Check hardware and Python compatibility first; audio models are dependency-sensitive.
- Use a venv and pin versions when the tool requires older Python/transformers.
- Generate short previews before long renders.
- Save prompts, seeds/configs, and output paths for reproducibility.

## Analysis Workflow

- Choose visualization type based on question: mel for timbre/energy, chroma for harmony, MFCC for compact features.
- Preserve sample rate and channel assumptions in the report.
- Export images/features with paths the user can inspect.

## Pitfalls

- Treating AI music tags as prose; concise bracketed performance cues work better.
- Ignoring dependency compatibility for local generation stacks.
- Running long audio renders before confirming prompt direction.
- Reporting audio analysis without sample rate/window assumptions.

## Verification Checklist

- [ ] Correct creative/generation/analysis path chosen.
- [ ] Environment/hardware checked for local models.
- [ ] Outputs saved with paths or media attachments.
- [ ] Prompts/configs preserved for reproducibility.
