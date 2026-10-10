# AI MaaS — A Private, Base-Model-Agnostic Fine-Tuning Platform

> Base-model agnostic · Private/offline deployment · COLA DDD microservices
> Status: **🚧 M0 (under development, not production-ready)** ｜ License: [Apache-2.0](./LICENSE)

[English](./README.en.md) | [简体中文](./README.md)

AI MaaS is a **privately deployable** platform for **fine-tuning and serving** large language models, built for vertical industries such as finance and legal.

## Why AI MaaS

- **Base-model agnostic**: swap in any open model (Qwen / Llama / DeepSeek, …) via configuration only — no business-code changes.
- **Private / offline by design**: your data never leaves your domain; supports air-gapped and on-prem environments.

## Architecture Overview

```
            ┌──────────────────────────────┐
            │        Vue 3 SPA (Frontend)    │
            └───────────────┬──────────────┘
                     HTTP / SSE / WS
                            │
                 ┌──────────▼───────────┐
                 │ Spring Cloud Gateway │
                 └──────────┬───────────┘
                    REST │            │ REST/WS
        ┌────────────────▼──┐   ┌─────▼──────────────────────────┐
        │  Control-plane     │   │        runtime-hub              │
        │  services (Java)   │   │  (unified WS control plane)     │
        │  model / dataset / │   └─────┬──────────────────────────┘
        │  training / ...    │         │ WebSocket
        └─────────┬──────────┘   ┌─────▼──────────────────────────┐
                  │              │  Python runtimes                │
                  │              │  puller / trainer / server /    │
                  │              │  scanner / compressor / ...     │
                  │              └─────┬──────────────────────────┘
                  └──────────┬─────────┘
                 ┌───────────▼───────────────────────────────┐
                 │ PostgreSQL · MinIO · Redis · RocketMQ · Nacos │
                 └────────────────────────────────────────────┘
```

## Tech Stack

| Layer | Tech |
|---|---|
| Backend | Java 17 · Spring Boot 3.x · Spring Cloud Alibaba · **COLA DDD** |
| Runtimes | **Python 3.12** (CPU) · PyTorch · Transformers · vLLM · PEFT |
| Frontend | Vue 3 · TypeScript · Pinia · Vue Router · **in-house components (no third-party UI lib)** |
| Storage | PostgreSQL (database-per-service) · MinIO |
| Messaging/Registry | RocketMQ · Nacos · Redis |
| Deployment | Docker Compose · Helm/K8s (planned) |
| Contracts | WebSocket message schema · ADRs · engine/recipe manifests |

## Key Features

- **Unified model abstraction**: a single description for tokenizer / chat_template / quantization / context length.
- **Unified WS control plane**: single bidirectional connection between Java and Python runtimes (dispatch tasks, collect status/progress/logs) with heartbeat, offline detection, exponential backoff, and a registry for multi-replica routing.
- **End-to-end pipeline**: model pull → post-pull scan (security/structure) → fine-tuning (CPU/GPU) → evaluation → quantization/compression → inference serving.
- **COLA DDD layering**: `domain` has zero framework dependencies; `adapter → app → domain ← infrastructure`.
- **Governance (M1+, planned)**: Model Registry · Eval Gate · Guardrail · Inference Gateway · RAG/Knowledge · Prompt Registry.

## Quick Start

> Requires Docker and Docker Compose. Local host ports are used by default; actual ports follow your `.env`.

```bash
git clone <your-fork-or-mirror-url> ai-maas
cd ai-maas/deploy/docker-compose
cp .env.example .env
docker compose up -d --build
```

Then open the gateway entry point: `http://localhost:18080` (see `MANAGEMENT_*`/port mappings in `.env`).

See [Quickstart](./docs/quickstart.md) and [Deployment](./docs/deployment.md) for details.

## Documentation

| Doc | Description |
|---|---|
| [docs/overview.md](./docs/overview.md) | Project overview and positioning |
| [docs/architecture.md](./docs/architecture.md) | Architecture (layers, control plane, data) |
| [docs/quickstart.md](./docs/quickstart.md) | Quickstart |
| [docs/development.md](./docs/development.md) | Development guide and conventions |
| [docs/deployment.md](./docs/deployment.md) | Deployment (Compose / GPU / offline) |
| [docs/dev-log/](./docs/dev-log/) | Daily development log |
| [PLAN.md](./PLAN.md) | Master plan and roadmap |
| [docs/decisions/](./docs/decisions/) | Architecture Decision Records (ADR) |

> Most detailed documents are currently in Chinese; English translations are being added incrementally.

## Roadmap

| Phase | Goal | Timeline |
|---|---|---|
| **M0** Validation | End-to-end minimal pipeline working (in progress) | 0–2 months |
| M1 MVP | Productized minimal loop, pilot-ready | 2–5 months |
| M2 Productization | Multi-tenant reusable, governance layer | 5–9 months |
| M3 Scaling | Template marketplace + data flywheel | 9–15 months |

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md). For security issues, follow [SECURITY.md](./SECURITY.md) — please do not disclose publicly.

## License & Disclaimer

- This **documentation repository** is licensed under [Apache-2.0](./LICENSE); the code repository follows its own license (MIT).
- The project is in **M0** and provided **“AS IS”**, **without any production-readiness warranty**; use at your own risk.
- When integrating third-party **base models**, you are responsible for evaluating and complying with their licenses and regulations.
- Third-party names/trademarks are used for descriptive and compatibility purposes only and do not imply endorsement.
