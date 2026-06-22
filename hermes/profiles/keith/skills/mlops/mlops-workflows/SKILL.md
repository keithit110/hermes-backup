---
name: mlops-workflows
description: "Class-level umbrella for ML operations: model/dataset hub work, benchmarking, experiment tracking, local inference, model surgery, and model-specific operational recipes."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [mlops, huggingface, evaluation, inference, experiments, models]
    related_skills: [local-llm-inference]
---

# MLOps Workflows

## Overview

Use this umbrella when the work is operationalizing ML systems rather than explaining ML concepts: fetching or publishing artifacts, running evaluations, logging experiments, serving models, performing model surgery, or using a model-specific toolkit.

## When to Use

- Downloading, uploading, searching, or managing Hugging Face models/datasets/spaces.
- Running LLM benchmarks such as MMLU/GSM8K with lm-evaluation-harness.
- Tracking training runs, sweeps, artifacts, dashboards, or registries in Weights & Biases.
- Running local/open inference with llama.cpp, vLLM, quantized checkpoints, or OpenAI-compatible servers.
- Applying specialized inference/model-editing workflows such as refusal-vector abliteration.
- Operating model-specific tools such as Segment Anything.

## Decision Tree

| Task | Branch | Verify by |
|---|---|---|
| Find/download/upload artifacts | Hub operations | `hf` command succeeds; files/checksums present |
| Benchmark a model | Evaluation harness | results JSON/CSV exists; task/model/config recorded |
| Track experiments | W&B | run URL or local offline run directory exists |
| Serve a model | Inference | health check + sample inference succeeds |
| Modify model behavior | Model surgery | before/after eval and rollback checkpoint |
| Run CV segmentation | Model-specific recipes | masks/output files produced and visually/quantitatively checked |

## Shared Safety Rules

- Check disk, VRAM/RAM, and expected download sizes before pulling large checkpoints.
- Record exact model IDs, revisions, quantization formats, seeds, benchmark tasks, and command lines.
- Keep credentials out of logs; prefer environment variables and CLI auth stores.
- Run a tiny smoke test before long evaluations or training jobs.
- Save outputs in a predictable directory and report paths/URLs rather than vague success.

## Branch Notes

### Hugging Face Hub

Use the `hf` CLI for login, repo creation, upload/download, discussions, and metadata inspection. Prefer pinned revisions for reproducibility.

### Evaluation Harness

Use lm-evaluation-harness for standardized benchmarks. Start with a small task/limit to validate model loading and tokenizer behavior, then run the full benchmark.

### Weights & Biases

Initialize runs with project/entity names, log configs and metrics, and close runs cleanly. For offline or CI use, capture the sync directory and tell the user whether syncing is still needed.

### Local Inference

Choose server/runtime based on hardware and model format: llama.cpp for GGUF/CPU-or-small-GPU, vLLM for high-throughput GPU serving, or provider-compatible HTTP endpoints for integration testing.

### Model Surgery and Specialized Models

Treat model surgery and model-specific toolkits as high-risk operational procedures: snapshot inputs, run a minimal repro, document deltas, and keep rollback artifacts.

## Absorbed Detailed Packages

Detailed original packages and support files are stored under `references/absorbed/<old-skill-name>/`:

- `references/absorbed/huggingface-hub/SOURCE_SKILL.md`
- `references/absorbed/evaluating-llms-harness/SOURCE_SKILL.md`
- `references/absorbed/weights-and-biases/SOURCE_SKILL.md`
- `references/absorbed/obliteratus/SOURCE_SKILL.md`
- `references/absorbed/segment-anything-model/SOURCE_SKILL.md`

## Verification Checklist

- [ ] Captured model/dataset IDs and revisions.
- [ ] Ran a smoke test before expensive work.
- [ ] Preserved output paths, result files, URLs, or run IDs.
- [ ] Documented hardware/resource assumptions.
