# Architecture

[English](./architecture.en.md) | [简体中文](./architecture.md)

> M0 architecture; subject to change.

## Overview

```
Frontend (Vue 3)  ──HTTP/SSE/WS──►  Gateway  ──►  Control-plane services (Java/COLA DDD)
                                        │
                                        └──WS──► runtime-hub ──WS──► Python runtimes
                                                     │
                               PostgreSQL · MinIO · Redis · RocketMQ · Nacos
```

## Layering (per Java service)

- **adapter**: REST controllers, WS endpoints, MQ listeners.
- **app**: application orchestration, transaction boundaries, cross-aggregate coordination.
- **domain**: entities/value objects/domain services/state machines, **zero framework dependencies**.
- **infrastructure**: MyBatis-Plus, Redis, MinIO, external clients.

Dependency direction: `adapter → app → domain ← infrastructure`, enforced by ArchUnit.

## Services (control plane)

| Service | Responsibility (examples) |
|---|---|
| `model` (base-model) | Base-model registry/pull/post-pull scan/variant assets |
| `dataset` | Dataset registry/import/scan/subsets/lineage |
| `training` (training-orchestration) | Training job orchestration & lifecycle |
| `artifact` | Model artifacts / promotion / lineage |
| `serving` | Inference services and endpoints |
| `eval` | Evaluation jobs and gates |
| `compress` | Quantization/compression jobs |
| `gateway` | Unified entry, authn, routing, rate limiting |
| `scheduler` (GPU) | GPU visibility/allocation (M1+) |
| `node` / `admin` / `assistant` / `image` | Node orchestration, admin, assistant, image service |

> The authoritative list follows the code repository (`maas-services/`) and the plan.

## Unified WebSocket control plane

- **Single bidirectional** connection between Java and Python runtimes: dispatch task commands + return status/progress/logs/results.
- A Redis-backed **connection registry** enables **multi-replica** routing; heartbeat, offline detection, exponential backoff reconnect.
- Message contracts live under [protocol/](./protocol/) (incrementally added) and the code repo schemas.

## Data & storage

- **PostgreSQL**: database/schema-per-service; multi-tenant `tenant_id` row-level + RLS fallback (M1).
- **MinIO**: object storage for models/datasets/artifacts; presigned direct-link previews.
- **Redis**: sessions/registry/event bus (SSE cross-replica broadcast).
- **RocketMQ**: async tasks/decoupling (incrementally adopted).
- **Nacos**: registry & config.

## Key principles

- **Contract-first**: JSON Schemas for WS/job/image messages.
- **State-machine driven**: explicit task lifecycles (e.g., ADR-0009 training state machine).
- **Observability**: structured logs + metrics (M1).
- **Secure by default**: least privilege, externalized secrets, private-first.

## ADRs

Architecture Decision Records live in [decisions/](./decisions/) (incrementally added).
