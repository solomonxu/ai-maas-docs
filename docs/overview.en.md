# Overview

[English](./overview.en.md) | [简体中文](./overview.md)

## In One Sentence

OpenMaas is a self-hosted platform for model training, inference, and evaluation, with capabilities for base models, datasets, training, artifacts, inference, evaluation, compression, resource governance, and pipelines.

## Positioning

| Dimension | Description |
|---|---|
| Self-hosted | Data and models remain in your environment; suitable for private networks and regulated scenarios |
| Base-model agnostic | Unified model descriptions and inference adapters; swap base models without business-code changes |
| End-to-end | Pull → data → training → artifact → inference → evaluation → compression |
| Run semantics | Preflight snapshots, stage state machines, database leases, recovery, cost, and audit |
| Engineering | Java microservices, Python runtimes, Vue 3 frontend, COLA DDD boundaries |

## Target Users

- architects and platform engineers building self-hosted AI platforms;
- MLOps teams responsible for training, inference, and evaluation;
- organizations focused on resource governance, cost attribution, and audit;
- teams that want to evaluate or replace open base models quickly.

## Core Concepts

- **Base model:** source model weights that can be fine-tuned or served.
- **Variant:** the smallest deployable unit, with format, precision, and quantization attributes.
- **Runtime:** a Python execution component controlled through a WebSocket control plane.
- **Artifact:** a training or compression output that can be registered, promoted, and released.
- **Pipeline run:** a cross-service training-to-inference transaction with stages, preflight, events, and cost.

## Scope

Current scope:

```text
base models / datasets / training / artifacts / inference
/ evaluation / compression / pipelines / resource governance
```

Not goals for the first public version:

- implementing a foundation-model training framework from scratch;
- building an in-house inference engine;
- replacing professional data-labeling tools;
- exposing an arbitrary free-form DAG;
- providing a complete financial billing system.

## Status

OpenMaas is in open-source closeout. The public repository and documentation site are coming soon. The current version is pre-1.0; APIs and capabilities may still change.

## Related

- [Architecture](./architecture.en.md)
- [Quickstart](./quickstart.en.md)
- [Deployment](./deployment.en.md)
- [Development](./development.en.md)
- [Security](../SECURITY.en.md)
