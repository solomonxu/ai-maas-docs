# AI MaaS — Database & Domain Model Plan (English Summary)

> Version: v0.3 (incremental revision) · Date: 2026-09-25
> Scope: database schema & domain model only (no business code).
> Aligned with `PLAN.md` v2.0 (Java/COLA + Python runtimes + WebSocket control plane).
> Database: PostgreSQL 15+ · Migration: Flyway · PK: BIGINT snowflake (ULID-swappable).
> Core entities: tenant / base_model / dataset / compression_job / training_job / model_artifact / inference_instance / eval_task / billing_record / schedule_task.
>
> **This is an English summary/structure mapping.** Authoritative column-level details are in the Chinese [DB_SCHEMA_PLAN.md](./DB_SCHEMA_PLAN.md). Section numbers are kept aligned.

---

## 0. General conventions

**0.1 Audit fields (every table except `tenant`):** `id` (BIGINT PK, snowflake), `tenant_id` (forced filter on all queries), `created_at`, `updated_at`, `created_by`, `updated_by`, `deleted` (0/1 soft delete), `version` (optimistic lock).

**0.2 Physical deployment & FK policy (important).** The platform uses **database-per-service**:
- DB-level **physical FKs only within the same service's database**.
- **Cross-service references are logical** (store ID only, no FK); consistency via application/domain events.
- If M0 starts with a single PG / multiple schemas, same-schema FKs are allowed; cross-schema stays logical.

Service ownership: `tenant`→IAM(`maas_iam`, Phase2) · `base_model`→base-model(`maas_base_model`) · `dataset`→dataset(`maas_dataset`) · `training_job`→training-orchestration(`maas_training`, also holds runtime control-plane tables §7) · `compression_job`→compress(`maas_compress`) · `model_artifact`→artifact(`maas_artifact`) · `inference_instance`→serving(`maas_serving`) · `eval_task`→eval(`maas_eval`) · `billing_record`→billing(`maas_billing`) · `schedule_task`→scheduler(`maas_scheduler`). The 10 core tables are almost entirely **logical references**.

**0.3 Incremental revision (aligned with PLAN.md §0.5).** Without deleting/refactoring the core 10 tables, adds governance structures in three places: ① `model_artifact` +2 fields (`source_hash`, `eval_summary_json`; `stage`/`parent_artifact_id` already exist) — §1.6; ② M1 lightweight governance tables (eval golden set + results; guardrail policy + log) — §8; ③ M2/M3 governance service tables — §9/§10. Principle: **no new service → no new DB**; new-table indexes/creation order in §11.

## 1. Core table descriptions (columns in Chinese doc)
- **1.1 `tenant` — Tenant** (global; no `tenant_id`): `code` (unique), `name`, `type` (internal/enterprise/trial), `status` (active/suspended/closed), `isolation_mode` (row/schema/database), `schema_name`, contacts, `quota_json` (GPU/storage/concurrency), `expired_at`.
- **1.2 `base_model` — Base model**: display name, family (Qwen/Llama/DeepSeek/GLM…), provider, parameter scale/count, license type & commercialization allowlist, context length, tokenizer, chat_template, quant/weight format, `model_uri` (MinIO/local), size, sha256 checksum, source (builtin/uploaded/hf/import), status (registered/available/offline/deprecated).
- **1.3 `dataset` — Dataset** (+ versions/annotation): data format, task type, status, validation results.
- **1.4 `compression_job` — Compression job**: method (awq/gptq/prune/distill), target quantization, calibration/output URIs, status.
- **1.5 `training_job` — Training job**: base model / dataset references, method (lora/qlora/full/dpo), hyperparams, checkpoints, metrics, status/state.
- **1.6 `model_artifact` — Model artifact**: type (Adapter/Fused/Quantized), stage (DEV/STAGING/PROD), parent artifact, storage URI, `source_hash`, `eval_summary_json`.
- **1.7 `inference_instance` — Inference instance**: model/artifact references, engine (vllm/tgi/ollama), tensor parallel, max model length, GPU memory util, port, replicas, status.
- **1.8 `eval_task` — Evaluation task**: eval set, metrics config, thresholds/gate, result summary.
- **1.9 `billing_record` — Usage/billing record**: usage metrics, quota, billing rule reference.
- **1.10 `schedule_task` — Scheduling task** (reference): resource pool, GPU node, allocation, status.

## 2. ER & physical FK
**2.1 Logical ER (domain view):** tenant 1—N everything; base_model 1—N dataset? no — base_model 1—N training_job/compression_job/inference_instance; dataset 1—N training_job; training_job 1—N model_artifact; model_artifact 1—N inference_instance; model_artifact 1—N eval_task. **2.2 Physical FK:** only within the same service DB; cross-service logical.

## 3. Required indexes
Per table, index the high-selectivity/foreign keys and common filters: `tenant`(code unique), `base_model`(tenant+family/status/source), `dataset`(tenant+status), `compression_job`/`training_job`(tenant+status, +base_model_id/dataset_id), `model_artifact`(tenant+base_model_id, +stage), `inference_instance`(tenant+status, +endpoint), `eval_task`(tenant+status, +artifact_id), `billing_record`(tenant+created_at), `schedule_task`(tenant+status, +resource). All partial with `deleted=0`.

## 4. State machines
- **4.1 `training_job`:** explicit lifecycle (submitted → queued → running → succeeded/failed/cancelled/abandoned; pause/resume; retry). Authoritative in Java.
- **4.2 `inference_instance`:** creating/deploying/running/upgrading/degraded/failed/stopped/cold.
- **4.3 `eval_task`:** queued/running/succeeded/failed/cancelled.
- **4.4 `schedule_task`:** (reference for scheduler) pending/allocated/running/released/failed.

## 5. Multi-tenancy
Phase 1: `tenant_id` row-level + PostgreSQL RLS fallback. Phase 2: schema/database-level isolation for large customers. `tenant.isolation_mode` selects the mode. Domain stays tenant-unaware.

## 6. Table creation order (v1)
**6.1 Single-DB/schema first (M0 compose, physical FK allowed):** tenant → base_model → dataset/dataset_version → training_job/runtime_node/task_dispatch → model_artifact → … **6.2 Service-oriented (database-per-service, independent Flyway).** **6.3 Migration conventions.** **6.4 Runtime control-plane tables (see §7).** **6.5 Full dependency topology incl. runtime tables.**

## 7. Alignment with the runtime architecture (PLAN v2.0)
**7.1 Runtime control-plane tables:** `model_pull_task`, `runtime_node`, `task_dispatch` (+ events/outbox) held by training-orchestration (`maas_training`). **7.2 Relations.** **7.3 `engine_job_id` semantics unified** (the runtime's job id). **7.4 State machines aligned with WS lifecycle.** **7.5 Required indexes on runtime tables.** **7.6 Tenant notes.**

## 8. M1 lightweight governance tables (eval + guardrail)
- **8.1 `eval_set`** — golden set (50–200 items).
- **8.2 `eval_result`** — lightweight eval results.
- **8.3 `guardrail_policy`** — guardrail policy (lite): length/sensitive-words/PII/output-schema.
- **8.4 `guardrail_log`** — guardrail hits log.

## 9. M2 governance service tables
- **9.1 model-registry** (`maas_model_registry`): versions/approval/model card.
- **9.2 inference-gateway** (`maas_gateway`): routes/rate-limit policy/api keys/audit.
- **9.3 knowledge-service** (`maas_knowledge`, requires **pgvector**): documents/chunks/embeddings.
- **9.4 prompt-registry** (`maas_prompt`): prompt templates/versions/deployments.

## 10. M3 governance service tables
- **10.1 experiment-service** (`maas_experiment`): experiments/runs.
- **10.2 lineage-service** (`maas_lineage`): data→train→artifact→eval→deploy lineage.
- **10.3 template-market** (`maas_template`): industry templates publish/subscribe/versioning.

## 11. New-table indexes & creation order
**11.1 Required indexes** for the governance tables. **11.2 Creation order** for the governance tables.

## Appendix — YAGNI notes
Avoid over-modeling: no premature multi-DB splitting, no full CQRS/event-sourcing tables in MVP, no unnecessary joins across services.

---
> English summary; authoritative column-level details in the Chinese [DB_SCHEMA_PLAN.md](./DB_SCHEMA_PLAN.md).
