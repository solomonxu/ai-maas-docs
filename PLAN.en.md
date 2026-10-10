# AI MaaS — Project Plan (PLAN, English Summary)

> Version: v2.0 (draft) · Date: 2026-09-25
> Positioning: a private/hybrid-deployable platform for vertical-domain model **fine-tuning and serving**, acting as a **base-model-agnostic middle layer**.
> Tech mainline: Java 17/21 + Spring Boot 3.x + Spring Cloud Alibaba + COLA DDD + Vue 3 + Python runtimes + K8s.
> Control-plane comms: unified **WebSocket** long connection (Java ↔ Python runtimes) in the MVP.
>
> **This is an English summary/structure mapping.** The authoritative full text is the Chinese [PLAN.md](./PLAN.md). Section numbers are kept aligned.

---

## 0.5 Roadmap revision (2026-09-25)

**Background.** The plan covered the core "train & serve" path but had governance gaps: Model Registry, Eval Harness, Guardrails, Inference Gateway, RAG/Knowledge Base, Prompt Registry, Experiment Tracking, Cost/Quota/FinOps, Data Lineage, Scheduler.

**Principles.** No re-architecture (incremental only); no over-engineering (YAGNI); keep private/offline compatibility; keep the three-phase goals.

**Increments by phase.**

| Phase | Goal (unchanged) | Increments | Key principle |
|---|---|---|---|
| M1 | Single-customer loop | eval baseline + guardrail-lite + artifact lineage fields | quantify quality without new services |
| M2 | Multi-customer reusable | inference-gateway / model-registry / eval-service / knowledge-service / prompt-registry / guardrail-service | govern via services |
| M3 | Platform scaling | experiment / lineage / template-market / FinOps | scale & ecosystem |

**Capability maturity matrix (M1→M3):** training/compress/serve ✅; eval gate → lightweight → service → continuous; model registry → fields → service+approval → model card+lineage; inference gateway → none → ✅ → multi-model routing; RAG → skip → ✅ → multi-domain; prompt mgmt → files → versioned → A/B; guardrails → length/sensitive/PII → service → policy orchestration; lineage/experiment → none → fields → full-chain; FinOps → usage → quota → chargeback.

---

# Part 1 — Project overview

## 1. Goals & non-goals
**Goals.** G1: end-to-end loop "vertical data → fine-tune 7B/14B → eval → private inference". G2: base-model-agnostic pipelines + standardized (OpenAI-compatible) APIs. G3: productize into a reusable platform (multi-tenant, observable, metered, offline). G4: on-prem/hybrid deployment with data staying in-domain.
**Non-goals.** No from-scratch trillion-scale base training (fine-tune/continued-pretrain/distill/quantize only); no in-house inference engine (reuse vLLM/Ollama/TGI/TensorRT-LLM); no in-house generic labeling tool (integrate LabelStudio); not a public multi-tenant SaaS first; 500B+ is long-term only.

**Model tiers.** 7B/14B (MVP, LoRA/QLoRA) · 70B (productization, full/continue + multi-GPU) · 500B+ (exploration).

**Base-model-agnostic abstraction (core differentiator).** Unified model description (tokenizer/chat_template/quantization/context/license); inference-backend adapter (vLLM/TGI/Ollama behind `model-server`, OpenAI-compatible outward); unified dataset/eval adapters with pluggable industry templates. Base model and engine are replaceable implementations; business services depend only on interfaces.

## 2. Milestones & deliverables
- **M0 (0–2 mo)**: manually prove the minimal end-to-end path. Java: gateway + `base-model`/`dataset`/`training-orchestration` skeletons + `runtime-hub` WS. Python: `model-puller` + `model-trainer`(LoRA) + `model-server`(vLLM). Deploy: single-node docker-compose + 1 GPU.
- **M1 (2–5 mo)**: productized minimal loop; first pilot customer. **No release without an eval score and a model card.** Deliver 4 services (base-model, dataset, training-orchestration incl. runtime-hub, artifact) + gateway + Vue console; 4 runtimes (puller/compressor/trainer/server). Integrate PG/MinIO/Redis/MQ/Nacos/LabelStudio/vLLM. Lightweight eval/guardrail implemented inside existing services (not separate).
- **M2 (5–9 mo)**: multi-customer reusable LLMOps. Add serving/eval/compress/billing/scheduler + governance services (inference-gateway, model-registry, eval-service, knowledge-service, prompt-registry, guardrail-service). Strengthen multi-tenant/RBAC/audit/quota/observability; offline packages, upgrades/rollback, 70B.
- **M3 (9–15 mo)**: governance + flywheel. Scenario template market, multi-base adapters, GitOps multi-cluster, 500B distill/distributed, billing; experiment-service/lineage-service/template-market/FinOps/data flywheel.

## 3. Repository layout
**Decision: monorepo** (Maven multi-module + Python packages + frontend) at M0–M1; split later if needed. `maas/` contains: `pom.xml`, `docs/` (adr/api/protocol/deploy), `maas-common/*` (core/ddd/web/mybatis/mq/redis/minio/runtime-ws), `maas-gateway/`, `maas-services/*`, `runtimes/*` (common SDK + puller/compressor/trainer/server), `maas-web/`, `deploy/` (docker-compose/helm/k8s/nacos/offline), `.github/workflows`.

---

# Part 2 — Backend architecture

## 4. COLA four-layer plan per service
**4.0 Form.** One Maven module per service with COLA packages (lightweight, M0–M1); upgrade to one module per layer only when large. Dependency: `adapter → app → domain ← infrastructure`; **domain has zero framework deps**.
**4.1 Template.** `adapter/{web,ws,mq,rpc}`, `app/{service,handler,dto}`, `domain/{entity,vo,service,repository,event}`, `infrastructure/{persistence,rpc,config,mq,util}`, `bootstrap/`.
**4.2 base-model-service.** Controllers (BaseModel/License/Pull); command/query/pull app services; entities BaseModel/ModelVersion/ModelPullTask; VOs (family/scale/license/quant/pull-source); LicensePolicyService; repositories; events; RuntimeHubClient; persistence.
**4.3 dataset-service.** Dataset/Version/Annotation controllers; MQ listener; entities Dataset/DatasetVersion/AnnotationTask; DatasetValidationService; LabelStudio/BaseModel clients; events.
**4.4 training-orchestration (incl. runtime-hub).** Controllers + WS endpoint/dispatcher + MQ listener; command/query/hub/dispatch services; handlers (Ack/Progress/Result/Error/NodeRegister); entities TrainingJob/TrainingRun/TrainingEventLog/RuntimeNode/TaskDispatch; state machine + validators + node selection; `ConnectionRegistry` (Redis).
**4.5 artifact-service.** Artifact/promotion controllers; MQ listeners; entities Artifact/ArtifactVersion/PromotionRecord; PromotionPolicyService (M1 manual); MinIO presign/checksum.
**4.6 Phase 2 services (skeleton).** serving/eval/compress/billing/scheduler with their domains and key events.

## 5. Cross-service communication
**Decision:** reads → sync (Feign); state/lifecycle → async (Kafka/RocketMQ); Java↔Python runtime → WebSocket; inference data plane → HTTP/gRPC. Conventions: all cross-service events via `common-mq`; **no direct DB access across services**; sync only for aggregate queries. WS connection state is stateful → Redis registry + Pub/Sub routing. Java is the single source of truth for task state; WS is transport only.

## 6. Service batches
**Batch 1 (MVP, 4):** base-model-service, dataset-service, training-orchestration (carries runtime-hub), artifact-service.
**Batch 2 (Phase 2, 5):** serving, eval, compress, billing, scheduler (M2 moves runtime-hub to scheduler; annotation folds into dataset).

## 7. Table ownership (database-per-service, no cross-service joins)
Per service DBs: `maas_base_model`, `maas_dataset`, `maas_training`, `maas_artifact`; Phase 2: `maas_iam`, `maas_serving`, `maas_eval`, `maas_compress`, `maas_billing`, `maas_scheduler`. All tables include `id/tenant_id/created_at/updated_at/created_by/deleted`. Event reliability via **Outbox** + relay (`FOR UPDATE SKIP LOCKED`).

## 8. Runtime interaction boundary
Northbound only via gateway; Python runtimes never face outward — they connect to Java `runtime-hub` over `wss://{host}/ws/runtime`. Operations (pull/compress/train) are commands from business services → hub → runtime. Data plane uses MinIO (messages carry URI + checksum only). M2: `model-server` exposes internal HTTP (OpenAI-compatible) + gRPC, proxied/authenticated by serving/gateway. Runtimes run on GPU nodes via Deployment/StatefulSet with NetworkPolicy.

## 9. Multi-tenancy
**Phase 1:** `tenant_id` row-level + PostgreSQL RLS fallback. **Phase 2:** schema-level isolation for large customers (hybrid). Adapter parses tenant; app passes through; **domain is tenant-unaware**; infrastructure injects tenant (MyBatis interceptor) + RLS. Cross-service via `X-Tenant-Id`/event/WS envelope. Private deployment may fix a default tenant but keep the mechanism.

---

# Part 3 — Runtime components

## 10. Responsibilities & YAGNI
| Component | Does | Explicitly does NOT |
|---|---|---|
| model-puller | Pull weights/config from HF/ModelScope; checksum; upload MinIO; report meta/progress | No format convert/quantize/license decisions/GPU |
| model-compressor | Quantize (AWQ/GPTQ)/prune/distill; output compressed artifacts | No training/eval/multimodal |
| model-trainer | Full/LoRA/QLoRA/DPO fine-tune; resume; metrics; adapter/fused artifacts | No data cleaning/labeling/scheduling/inference |
| model-server | Load model, online inference, streaming, health; M2 autoscale | No training/compression/auth/billing (report usage only) |
| runtime common | Connection, protocol codec, heartbeat/reconnect, MinIO wrapper, structured logs, metrics | No business semantics |

## 11. Unified WebSocket protocol
**11.1 Handshake.** `wss://{host}/ws/runtime`; client sends first-frame `AUTH` (avoid token in URL/logs) → `AUTH_ACK`; unauthenticated connections closed within 10s. Binding unique by `(runtimeType,nodeId)`; duplicate → kick old; app-level `PING/PONG` every 30s, offline after 90s; exponential backoff reconnect (1s→30s) and `NODE_REGISTER` with running tasks for reconciliation.
**11.2 Envelope (JSON).** `type, msgId(uuid), taskId?, runtimeType, nodeId, tenantId, timestamp, traceId?, payload?`.
**11.3 Message types.** Runtime→Java: AUTH, NODE_REGISTER, NODE_STATUS, PING/PONG, TASK_ACK, TASK_PROGRESS, TASK_LOG, TASK_RESULT, TASK_ERROR. Java→Runtime: AUTH_ACK, TASK_ASSIGN, TASK_CANCEL, PING/PONG, CONFIG_UPDATE.
**11.4 `TASK_ASSIGN.spec` per component:** puller (source/repoId/revision/files/targetUri/hfTokenRef) · compressor (sourceUri/method/targetQuant/calibrationUri/outputUri/config) · trainer (baseModelUri/datasetUri/method/hyperparams/outputUri/resumeFrom/tensorParallel/gpuCount) · server (modelUri/engine/tensorParallel/maxModelLen/gpuMemoryUtil/port/replicas).
**11.5 Error codes.** 1000 OK; 4001 AUTH_FAILED; 4002 TOKEN_EXPIRED; 4003 RUNTIME_TYPE_MISMATCH; 4004 NODE_ALREADY_CONNECTED; 4100 INVALID_PAYLOAD; 4101 TASK_NOT_FOUND; 4102 TASK_ALREADY_RUNNING; 4103 TASK_CANCELLED; 4200 GPU_OOM(retry); 4201 GPU_UNAVAILABLE(retry); 4300 MODEL_DOWNLOAD_FAILED(retry); 4301 MODEL_FORMAT_UNSUPPORTED; 4302 CHECKSUM_MISMATCH(retry); 4400 TRAINING_FAILED(retry); 4401 DATASET_INVALID; 4500 INFERENCE_LOAD_FAILED(retry); 5000 INTERNAL_ERROR(retry).
**11.6 Reliability.** At-least-once (TASK_ACK; resend ≤3× on 15s timeout); idempotency by `msgId`; Java state machine guards transitions; reconnect reconciles `runningTasks` against DB; offline >90s → mark in-flight FAILED(retryable) and re-queue; per-task ordering by timestamp.
**11.7 Typical flow:** AUTH → AUTH_ACK → NODE_REGISTER → TASK_ASSIGN ← business command → TASK_ACK → TASK_PROGRESS/TASK_LOG → TASK_RESULT → MQ event → artifact-service.

## 12. Component tech & directory
**12.1 Common SDK** (`runtimes/common`, internal wheel `maas-runtime-sdk`): connector, protocol (pydantic, aligned with `docs/protocol`), config, storage (MinIO), logging (structlog+traceId), metrics (prometheus-client), errors. Components reuse `RuntimeConnector` and only implement `handler.py`.
**12.2 Selection:** puller (huggingface_hub/modelscope/boto3/websockets/pydantic) · compressor (torch/transformers/autoawq/auto-gptq/peft/datasets/accelerate) · trainer (torch/transformers/peft/deepspeed/accelerate/bitsandbytes/datasets/safetensors) · server (vllm/fastapi/uvicorn/transformers/websockets/prometheus-client) · tests (pytest/pytest-asyncio/jsonschema).
**12.3 Layout:** `runtimes/{common, model-puller, model-compressor, model-trainer, model-server}` each with `src/` + Dockerfile.

## 13. Deployment topology (K8s)
**13.1 Workloads/resources:** puller (Deployment, 2c/4Gi, egress to HF, overseas CPU node) · compressor (0–1, 4–8c/32Gi/1×24GB) · trainer (8–16c/64–128Gi/1–8 GPU, taint, NVMe checkpoints) · server (Deployment+HPA, 8c/32Gi/1 GPU) · runtime-hub (≥1, 2c/4Gi).
**13.2 K8s notes:** namespaces `maas-system`/`maas-runtime`; GPU via nodeSelector/limits/tolerations + GPU Operator; NetworkPolicy allows only runtime→hub/MinIO/Nacos/MQ; large `proxy_read_timeout` (e.g. 3600s) + TLS; PVCs for cache/checkpoints; Secrets for MinIO/HF/bootstrap.
**13.3 Stateful connection routing (key).** Redis `ConnectionRegistry` (`nodeId → hubInstanceId`) + Pub/Sub to the owning instance; or single-replica hub for MVP; graceful `DRAIN` before rolling updates.

## 14. Security
**14.1 Auth.** One-time bootstrap token → short-lived JWT (5–15min); WS first-frame `AUTH`; single connection per nodeId; audit connect/disconnect; token expiry handled via CONFIG_UPDATE/reconnect.
**14.2 Sensitive data.** HF token/MinIO creds/API keys in Secret/Vault; **payload references secrets by name only**; API keys encrypted + logs redacted; never echo secrets/user data in TASK_LOG/errors.
**14.3 Internal encryption.** `wss` always; optional mTLS; default-deny NetworkPolicy; non-root/read-only rootfs/limited caps; max frame 1MB, rate limits, task allowlist, `taskId` tenant-ownership checks.

---

# Part 4 — Frontend

## 15. Tech selection
Vue 3 (`<script setup>`) · TypeScript (strict) · Vite · Pinia · Vue Router 4 · UI: Element Plus (alt Naive UI) — *note: the current codebase uses in-house components; this plan item is historical* · axios wrapper (token/tenant/traceId) · openapi-typescript · ECharts · WS/SSE (M2).

## 16. Frontend layout
`maas-web/` with `src/{api,types,stores,router,pages,components,layouts,composables,utils,assets}`; pages: base-model/dataset/training/runtime/artifact.

## 17. FE/BE collaboration
Aggregate OpenAPI in CI (`springdoc`) → `docs/api/openapi/maas.openapi.json`; frontend generates TS types; `client.ts` injects baseURL/token/tenant/trace and maps error codes; runtime status read via Java aggregate APIs (not direct to Python); CI **schema diff** fails on breaking changes.

## 18. Frontend deployment
Static Nginx image; Phase 1 via docker-compose (`maas-web` + `maas-gateway`); Nginx proxies `/api` (same-origin); Helm Ingress for private deployment; long-connection timeouts if WS/SSE; no embedding into backend jars.

---

# Part 5 — Execution order & risks

## 19. Execution order
**19.1 Backend+runtime (M0):** parent POM + `maas-common/*` → `runtimes/common` SDK + WS protocol → gateway+Nacos + hub WS (single replica) → base-model → dataset → training-orchestration → artifact → 3 runtimes → web + OpenAPI → compose.
**19.2 Tables:** model_pull_task/base_model → dataset/dataset_version → runtime_node/task_dispatch/training_job/training_run/outbox_event → artifact/artifact_version/promotion_record (Flyway per service).
**19.3 APIs:** base-models/pull → datasets/versions/validate → training-jobs/cancel/logs → artifacts/promote → runtime/nodes,tasks → WS `/ws/runtime` → Phase 2 `/v1/chat/completions`, `/v1/evals`, `/v1/usage`.
**19.4 Testing:** domain unit → ArchUnit → Testcontainers (tenant/RLS) → `@WebMvcTest` → WS contract (shared JSON Schema) → pytest → E2E smoke → Phase 2 real-GPU.
**19.5 CI:** M0 `mvn verify`+`pytest`+frontend lint/typecheck/build; M1 images + compose + API/WS contract + openapi diff; M2 e2e + SAST + Helm staging + signing; M3 GitOps + canary/rollback.
**19.6 Deploy order:** M0 single-node compose; M1 kind/k3s + Helm + GPU Operator/vLLM; M2 prod K8s + observability + offline; M3 multi-cluster.
**19.7 Compose services:** postgres/minio/redis/nacos/mq/gateway + 4 services + web (default); puller (profile), trainer/server/compressor (gpu), labelstudio (annotation), prometheus/grafana/loki (observability).
**19.8 Hardware baseline:** dev/M0–M1 POC — AMD Ryzen AI MAX+ 395 (128GB unified; 7B/14B inference, LoRA/QLoRA; **ROCm experimental, no production SLA**); production (M2+) — NVIDIA A100 80G / L20 / 4090 (CUDA 12.4). Production acceptance baseline is NVIDIA CUDA 12.4; ROCm is experimental only.

## 20. Risks & MVP scope
**20.1 Risks** (impact → mitigation): GPU scarcity/cost → 7B+LoRA/cloud; no egress in customer intranet → overseas pull + MinIO transfer/offline pack; format compatibility → lock verified paths, pin tokenizer/chat_template; WS dropped by private gateway → heartbeat/reconnect/resume + larger timeouts + idempotency; hub multi-replica routing → Redis registry (MVP single replica); runtime disconnection → Java holds state + reconciliation; driver/CUDA mismatch → lock matrix + matching torch in image; ROCm vs CUDA divergence → NVIDIA-only acceptance; data compliance → no-egress/encryption/audit/RLS; base license limits → LicensePolicyService allowlist; unquantifiable eval → fixed sets + gates + business metrics; MQ consistency → Outbox + idempotent consumers; tenant leakage → interceptor+plugin+RLS; quota/billing without enforcement → M1 single-tenant + record-only.
**20.2 MVP (=M1) scope:** text single-modal, 7B/14B, LoRA/QLoRA; 4 services; runtimes puller/trainer/server (compressor at M1-end/M2); CSV/JSONL import + native LabelStudio; single-node train submit/monitor/cancel (serial); versioned artifacts + manual promotion; vLLM OpenAI-compatible endpoint + single API key; compose + private Helm; usage record only, weak multi-tenant.

## 21. Now vs YAGNI
**Now:** parent POM + `maas-common/*`, gateway, Nacos, single-replica hub; `runtimes/common` SDK + WS protocol; 4 services; 3 runtimes; PG/MinIO/Redis/MQ/LabelStudio; core tests; Vue base pages + OpenAPI TS; compose + Helm skeleton + weak multi-tenant + usage.
**YAGNI (later):** trillion-scale pretraining; in-house engine/scheduler/labeling; multimodal/AutoML/federated; complex billing engine; strong isolation/cross-cloud/model market/public SaaS/mobile/full BI; service mesh/full CQRS/Saga; server autoscale/HTTP-gRPC endpoints/trainer-on-MQ (M2); 500B+ (M3); heavy per-layer COLA; hub multi-replica routing (M2).

---

# Part 6 — Decisions needed
**Tier 1 (must decide):** (1) first industry/scenario — recommended fintech CS / legal docs; (2) first batch service count — 4; (3) runtime-hub ownership — training-orchestration in MVP; (4) MQ — RocketMQ 5.x; (8) launch base models — Qwen2.5-7B/14B + Llama-3.1-8B; (10) GPU/CUDA — prod A100/L20/4090 + CUDA 12.4, dev Ryzen AI MAX+ 395 (ROCm experimental); (15) team/timeline — Java2+Python2+FE1+Algo1, M0 2 → M1 5 → M2 9 months.
**Tier 2 (default unless objected):** (5) Nacos; (6) single-module COLA; (7) M1 single-tenant + reserved fields; (9) WS AUTH-first + 30s/90s heartbeat; (11) frontend UI Element Plus (historical; current: in-house); (12) software + optional appliance with offline; (13) MVP LoRA/QLoRA only; (14) monorepo + GitHub Actions.

**Hardware baseline (decided):** dev/POC AMD Ryzen AI MAX+ 395 (ROCm experimental, no SLA); production NVIDIA A100 80G / L20 / 4090 (CUDA 12.4) from M2.

## Appendix — when to redraw the roadmap
(1) First customer needs RAG/knowledge rather than fine-tuning → move Knowledge Service to M1. (2) Industry is finance/gov/medical → move Guardrail/Audit/Model Card to M1-end. (3) 7B LoRA insufficient, need 70B full-parameter → advance Scheduler/multi-GPU. (4) Sales promises multi-tenant SaaS → fork the private path. (5) GPU budget halved → add inference cache/small-model routing/cost gate.

---
> This English file is a **summary/structure mapping**. For the authoritative and most current details, see the Chinese [PLAN.md](./PLAN.md).
