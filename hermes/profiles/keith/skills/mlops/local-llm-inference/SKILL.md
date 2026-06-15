---
name: local-llm-inference
description: "Run local/open LLM inference with llama.cpp, vLLM, quantized models, OpenAI-compatible servers, batching, and production-serving tradeoffs."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [llm, inference, llama-cpp, vllm, gguf, quantization, serving]
    related_skills: [huggingface-hub]
---

# Local LLM Inference

## Overview

Use this umbrella for running open/local LLMs on user-controlled hardware. It covers both lightweight GGUF/llama.cpp workflows and high-throughput vLLM serving. Choose based on model format, hardware, concurrency, and whether the user needs an OpenAI-compatible API, batch inference, or quick local testing.

## When to Use

- Discover/download local models and GGUF quantizations.
- Run `llama.cpp` directly or as an OpenAI-compatible server.
- Serve transformer models with vLLM for throughput, batching, and production APIs.
- Compare quantization/performance tradeoffs.
- Debug local model loading, GPU memory, context length, or server issues.

## Decision Tree

- **llama.cpp:** best for GGUF files, CPU/small GPU use, quick local testing, low-dependency server mode.
- **vLLM:** best for high-throughput GPU serving, batching, OpenAI-compatible production endpoints, and transformer checkpoints.
- **Hugging Face Hub:** use for model discovery/download before either runtime.

## Shared Workflow

1. Identify hardware: GPU model/VRAM, CPU/RAM, CUDA/ROCm/Metal availability.
2. Choose model size and quantization based on memory budget.
3. Start with a small prompt smoke test.
4. If serving, verify `/v1/models` and a chat/completions request.
5. Benchmark only after correctness and stability are confirmed.

## llama.cpp Notes

- Prefer exact GGUF filenames from the Hub for reproducibility.
- Server mode is useful for OpenAI-compatible clients.
- Quantization choice dominates speed/quality/memory tradeoff.

## vLLM Notes

- Use for production-style API deployment and offline batch inference.
- Watch GPU memory utilization, max model length, tensor parallelism, and quantization support.
- Validate model-specific chat templates and tokenizer behavior.

## Pitfalls

- Loading a model format with the wrong runtime.
- Picking a context length that silently exceeds VRAM.
- Benchmarking cold-start or failed requests.
- Assuming all quantization formats are supported by both runtimes.

## Verification Checklist

- [ ] Hardware and runtime compatibility checked.
- [ ] Model path/revision/quantization recorded.
- [ ] Smoke generation succeeded.
- [ ] Server endpoint verified when serving.
- [ ] Performance claims backed by real measurements.
