---
name: productivity-document-workflows
description: "Class-level umbrella for document productivity: OCR/PDF extraction and editing, PowerPoint deck creation/editing, and meeting-summary pipeline operations."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [productivity, documents, pdf, ocr, powerpoint, meetings]
    related_skills: [productivity-apis]
---

# Productivity Document Workflows

## Overview

Use this umbrella when the user needs work done on files or document-producing systems: extracting text from PDFs/scans, editing PDF text, creating or modifying PowerPoint decks, or operating a meeting-summary pipeline.

## When to Use

- User provides a PDF, scan, image-based document, or arXiv PDF and wants text extracted.
- User wants a PDF typo/title/date/content edited without rebuilding the whole document.
- User wants a `.pptx` read, summarized, created, cleaned, or modified.
- User asks about Teams meeting summaries, replaying meeting jobs, or Microsoft Graph subscription status.

## Decision Tree

| Input/request | Branch | Verify by |
|---|---|---|
| PDF/image needs text | OCR/extraction | markdown/text output exists and spot-checks against source |
| PDF needs small edit | PDF edit | edited PDF exists and page visually/textually checked |
| Presentation needs changes | PowerPoint | deck opens/validates; slide/text/media changes checked |
| Meeting pipeline task | Teams pipeline | CLI status/job/subscription output captured |

## Shared Document Rules

- Preserve originals; write outputs with clear suffixes.
- For OCR/extraction, choose lightweight extraction first, high-quality OCR when needed.
- For visual documents, verify with screenshots/thumbnails or extracted text, not just successful command exit.
- For office files, avoid corrupting package structure; validate by reopening or running a parser.
- Report exact output paths.

## Branch Notes

### OCR and Extraction

Use remote extraction when the URL is accessible and sufficient. For local PDFs, choose PyMuPDF for text PDFs and marker/OCR-style tools for scans or layout-sensitive documents.

### PDF Editing

Use focused PDF editing tools for small textual changes. For large layout rewrites, consider rebuilding from source instead.

### PowerPoint

Use package-aware scripts for reading and editing `.pptx` files. Keep slide design consistent; verify notes, media, and XML validity when modifying internals.

### Meeting Pipelines

Start with status/inspection commands, then replay or manage subscriptions only after confirming current state. Graph subscriptions expire and must be treated as time-sensitive.

## Absorbed Detailed Packages

Detailed original packages and support files are stored under `references/absorbed/<old-skill-name>/`:

- `references/absorbed/ocr-and-documents/SOURCE_SKILL.md`
- `references/absorbed/nano-pdf/SOURCE_SKILL.md`
- `references/absorbed/powerpoint/SOURCE_SKILL.md`
- `references/absorbed/teams-meeting-pipeline/SOURCE_SKILL.md`

## Verification Checklist

- [ ] Original files preserved.
- [ ] Output file paths reported.
- [ ] Visual/textual spot check completed for edited documents.
- [ ] Pipeline commands include real status/output evidence.
