# Overview

[English](./overview.en.md) | [简体中文](./overview.md)

## One-liner

AI MaaS is a **base-model-agnostic**, **privately deployable** platform for **fine-tuning and serving** large language models for vertical industries (M0, under development).

## Positioning & differentiators

| Dimension | Description |
|---|---|
| Base-model agnostic | Unified model description + inference-backend adapters; swapping Qwen/Llama/DeepSeek requires **configuration only**, no business-code changes |
| Private | Data never leaves your domain; supports on-prem/air-gapped environments for compliance-sensitive industries |
| End-to-end | Pull → scan → fine-tune → evaluate → quantize/compress → serve |
| Engineering | COLA DDD microservices + unified WS control plane + decoupled frontend/backend |

## Target users

- Organizations needing **private** LLM capabilities (finance, legal, government, …);
- Teams that want to **quickly try and swap** multiple open base models;
- Builders who care about **compliance, observability, and governance**.

## Core concepts

- **Base Model**: original weights that can be fine-tuned/served (e.g., `Qwen/Qwen2.5-0.5B`).
- **Variant**: the minimal deployable unit (a single file or a set of files) with format/precision/quantization attributes; role is `model` or `component`.
- **Runtime**: Python-side executors (pull/train/serve/scan, …) controlled over WebSocket.
- **Artifact**: outputs from training/compression (LoRA, quantized weights, …) that can be registered and promoted.

## Non-goals (current stage)

- Not aiming for full pretraining (focus: fine-tuning/LoRA and inference);
- Does not bundle LLM weights (users pull/import and comply with their licenses);
- M0 does not include large-scale data cleaning/indexing or multi-tenant governance (M1/M2).

## Status & roadmap

See the [README roadmap](../README.en.md#roadmap) and [PLAN.md](../PLAN.md). Currently **M0**; features and APIs may change.
