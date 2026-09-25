# AI MaaS 平台 — 数据库与领域模型规划 (DB & Domain Model)

> 版本: v0.3（增量修订稿） 日期: 2026-09-25
> 范围: 仅数据库 schema 与领域模型，不含业务代码。
> 对齐: `PLAN.md` v2.0（Java/COLA + Python 运行时 + WebSocket 控制面）。
> 数据库: PostgreSQL 15+；迁移工具: Flyway；主键: BIGINT 雪花（可换 ULID）。
> 实体: tenant / base_model / dataset / compression_job / training_job / model_artifact /
> inference_instance / eval_task / billing_record / schedule_task。
> 说明: 本文聚焦上述 10 个核心实体；运行时控制面表（model_pull_task / runtime_node /
> task_dispatch）见 §7，不在核心 10 表内。

---

## 0. 总体约定

### 0.1 通用审计字段（除 `tenant` 外每张表都有）
| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | BIGINT | 主键，雪花 ID |
| `tenant_id` | BIGINT | 租户 ID，所有查询强制过滤 |
| `created_at` | TIMESTAMPTZ | 创建时间，默认 now() |
| `updated_at` | TIMESTAMPTZ | 更新时间 |
| `created_by` | BIGINT | 创建人 |
| `updated_by` | BIGINT | 更新人 |
| `deleted` | SMALLINT | 软删标记，0=正常 1=删除 |
| `version` | INT | 乐观锁版本号 |

### 0.2 物理部署与 FK 策略（重要）
本平台采用 **database-per-service**（见 `PLAN.md`），因此：

- **物理外键（DB 级 FK）仅在同一个服务的库内建立**。
- **跨服务引用一律为逻辑引用**：只存 ID，不建 FK，由应用层/领域事件保证一致性。
- 若 M0 用「单 PG 多 schema」先行，则同 schema 内可建物理 FK，跨 schema 仍用逻辑引用。

服务归属：
| 实体 | 归属服务 | 库 |
|---|---|---|
| `tenant` | IAM（Phase 2） | `maas_iam` |
| `base_model` | base-model-service | `maas_base_model` |
| `dataset` | dataset-service | `maas_dataset` |
| `training_job` | training-orchestration | `maas_training`（同时含运行时控制面表，见 §7） |
| `compression_job` | compress-service | `maas_compress` |
| `model_artifact` | artifact-service | `maas_artifact` |
| `inference_instance` | serving-service | `maas_serving` |
| `eval_task` | eval-service | `maas_eval` |
| `billing_record` | billing-service | `maas_billing` |
| `schedule_task` | scheduler-service | `maas_scheduler` |

> 结论：**这 10 张表之间几乎全部是「逻辑引用」关系**，不建跨服务物理 FK。

### 0.3 路线图增量修订（对齐 PLAN.md §0.5）
在不删除、不重构核心 10 表的前提下，本版新增治理层表结构，共三处：
| 处 | 内容 | 章节 |
|---|---|---|
| ① | `model_artifact` 补 2 个字段：`source_hash`、`eval_summary_json`（`stage`/`parent_artifact_id` 原表已有） | §1.6 |
| ② | M1 轻量治理表：eval（golden set + 结果）、guardrail（策略 + 日志） | §8 |
| ③ | M2 治理服务表（model-registry / inference-gateway / knowledge-service / prompt-registry）与 M3 治理服务表（experiment / lineage / template-market） | §9、§10 |

> 原则：**不新建服务即不建库**；M1 轻量表随现有服务库部署，M2/M3 拆为独立服务库；新增表索引与建表顺序见 §11。

---

## 1. 各表字段说明（中文）

### 1.1 `tenant` — 租户
| 字段 | 类型 | 说明 | 约束 |
|---|---|---|---|
| id | BIGINT | 租户 ID | PK |
| `code` | VARCHAR(64) | 租户唯一编码 | NOT NULL, UNIQUE |
| `name` | VARCHAR(128) | 租户名称 | NOT NULL |
| `type` | VARCHAR(16) | 类型: internal/enterprise/trial | NOT NULL |
| `status` | VARCHAR(16) | 状态: active/suspended/closed | NOT NULL |
| `isolation_mode` | VARCHAR(16) | 隔离模式: row/schema/database | NOT NULL, 默认 row |
| `schema_name` | VARCHAR(64) | schema/database 隔离时的实际名 | NULL |
| `contact_name` | VARCHAR(64) | 联系人 | NULL |
| `contact_email` | VARCHAR(128) | 联系邮箱 | NULL |
| `contact_phone` | VARCHAR(32) | 联系电话 | NULL |
| `quota_json` | JSONB | 配额快照（GPU/存储/并发） | NULL |
| `expired_at` | TIMESTAMPTZ | 到期时间 | NULL |
| created_at/updated_at/created_by/updated_by/deleted/version | — | 审计字段 | — |

> tenant 为全局表，**不含 tenant_id**。

### 1.2 `base_model` — 基模
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID（内置基座可用 0 表示全局） |
| `name` | VARCHAR(128) | 显示名 |
| `family` | VARCHAR(32) | 模型家族: Qwen/Llama/DeepSeek/GLM… |
| `provider` | VARCHAR(64) | 提供方 |
| `parameter_scale` | VARCHAR(16) | 档位: 7B/14B/70B/500B+ |
| `parameter_count` | BIGINT | 精确参数量 |
| `license_type` | VARCHAR(32) | 许可证: apache/mit/llama/qwen/commercial/custom |
| `license_commercial_ok` | BOOLEAN | 是否允许商用（白名单门禁） |
| `context_length` | INT | 上下文长度(tokens) |
| `tokenizer` | VARCHAR(64) | tokenizer 标识 |
| `chat_template` | TEXT | 对话模板 |
| `quant_format` | VARCHAR(16) | 量化格式: none/fp16/bf16/gptq/awq/gguf |
| `weight_format` | VARCHAR(16) | 权重格式: safetensors/bin/gguf |
| `model_uri` | VARCHAR(512) | 模型存储路径（MinIO/本地） |
| `size_bytes` | BIGINT | 模型体积 |
| `checksum` | VARCHAR(64) | 校验和(sha256) |
| `source` | VARCHAR(16) | 来源: builtin/uploaded/hf/import |
| `status` | VARCHAR(16) | 状态: registered/available/offline/deprecated |
| `description` | TEXT | 描述 |
| 审计字段 | — | 通用 |

### 1.3 `dataset` — 数据集
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `name` | VARCHAR(128) | 数据集名 |
| `task_type` | VARCHAR(16) | 任务: sft/dpo/pretrain/rm |
| `modality` | VARCHAR(16) | 模态: text/image/multi（MVP 仅 text） |
| `format` | VARCHAR(16) | 文件格式: jsonl/csv/parquet |
| `schema_json` | JSONB | 数据字段 schema 定义 |
| `version` | INT | 数据版本号 |
| `is_latest` | BOOLEAN | 是否最新版本 |
| `storage_uri` | VARCHAR(512) | 数据存储路径 |
| `row_count` | BIGINT | 样本行数 |
| `size_bytes` | BIGINT | 数据体积 |
| `checksum` | VARCHAR(64) | 校验和 |
| `label_studio_project_id` | BIGINT | LabelStudio 项目 ID |
| `annotation_status` | VARCHAR(16) | 标注状态: none/annotating/completed |
| `status` | VARCHAR(16) | 状态: created/uploading/available/validating/invalid/archived |
| `origin` | VARCHAR(16) | 来源: upload/labelstudio/import |
| `description` | TEXT | 描述 |
| 审计字段 | — | 通用 |

> MVP 单版本内联；多版本若需独立血缘，Phase 2 拆 `dataset_version` 表（YAGNI）。

### 1.4 `compression_job` — 模型压缩任务
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `name` | VARCHAR(128) | 任务名 |
| `source_model_id` | BIGINT | 逻辑引用 base_model.id |
| `source_artifact_id` | BIGINT | 逻辑引用 model_artifact.id（可空） |
| `method` | VARCHAR(16) | 方法: gptq/awq/gguf/pruning/distill |
| `target_quant` | VARCHAR(16) | 目标量化: int4/int8/fp16 |
| `calibration_dataset_id` | BIGINT | 逻辑引用 dataset.id（校准集） |
| `config_json` | JSONB | 压缩参数(bits/group_size…) |
| `resource_spec_json` | JSONB | GPU/节点规格 |
| `engine_job_id` | VARCHAR(64) | Python 引擎任务 ID |
| `output_artifact_id` | BIGINT | 逻辑引用 model_artifact.id（产物） |
| `status` | VARCHAR(16) | queued/running/succeeded/failed/cancelled |
| `error_message` | TEXT | 失败原因 |
| `started_at`/`finished_at` | TIMESTAMPTZ | 起止时间 |
| 审计字段 | — | 通用 |

### 1.5 `training_job` — 训练任务
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `name` | VARCHAR(128) | 任务名 |
| `base_model_id` | BIGINT | 逻辑引用 base_model.id |
| `dataset_id` | BIGINT | 逻辑引用 dataset.id |
| `method` | VARCHAR(16) | lora/qlora/full/dpo/pretrain |
| `config_json` | JSONB | 超参(lr/epochs/batch/lora_rank…) |
| `priority` | INT | 优先级，越大越高 |
| `resource_pool` | VARCHAR(64) | GPU 资源池 |
| `gpu_type` | VARCHAR(32) | GPU 型号 |
| `gpu_count` | INT | GPU 卡数 |
| `node_count` | INT | 节点数 |
| `schedule_task_id` | BIGINT | 逻辑引用 schedule_task.id |
| `engine_job_id` | VARCHAR(64) | Python 引擎任务 ID |
| `output_artifact_id` | BIGINT | 逻辑引用 model_artifact.id |
| `progress` | SMALLINT | 进度 0–100 |
| `metrics_json` | JSONB | 最终指标(loss/bleu…) |
| `retry_count` | INT | 重试次数 |
| `status` | VARCHAR(16) | 状态机见 §4 |
| `error_code`/`error_message` | VARCHAR/TEXT | 失败信息 |
| `submitted_at`/`started_at`/`finished_at` | TIMESTAMPTZ | 时间线 |
| 审计字段 | — | 通用 |

### 1.6 `model_artifact` — 模型制品
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `name` | VARCHAR(128) | 制品名 |
| `artifact_type` | VARCHAR(16) | adapter/fused/quantized/merged/base_copy |
| `source_type` | VARCHAR(16) | training/compression/import |
| `source_job_id` | BIGINT | 逻辑引用来源任务(多态，无 FK) |
| `base_model_id` | BIGINT | 逻辑引用 base_model.id |
| `parent_artifact_id` | BIGINT | 自引用，血缘父制品(可空) |
| `storage_uri` | VARCHAR(512) | 存储路径 |
| `format` | VARCHAR(16) | safetensors/gguf/bin |
| `size_bytes` | BIGINT | 体积 |
| `checksum` | VARCHAR(64) | 校验和 |
| `source_hash` | VARCHAR(64) | **M1 新增**：来源指纹（基座+数据+超参哈希，用于去重/谱系） |
| `stage` | VARCHAR(16) | 晋级阶段: DEV/STAGING/PROD（原表已有） |
| `eval_summary_json` | JSONB | **M1 新增**：最近一次评测摘要（分数/指标/门禁），由轻量 eval 回写 |
| `metrics_json` | JSONB | 指标快照 |
| `status` | VARCHAR(16) | registered/available/deprecated/deleted |
| `description` | TEXT | 描述 |
| 审计字段 | — | 通用 |

> **修订（对齐 PLAN.md §0.5）**：M1 为 `model_artifact` 新增 `source_hash`、`eval_summary_json`；`stage`、`parent_artifact_id` 原表已具备。「最近评测结果」M1 先由 `eval_summary_json` 冗余兜底，同时仍可查询 `eval_result`，**不建 eval FK**，避免循环依赖。

### 1.7 `inference_instance` — 推理实例
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `name` | VARCHAR(128) | 实例名 |
| `artifact_id` | BIGINT | 逻辑引用 model_artifact.id |
| `base_model_id` | BIGINT | 冗余，逻辑引用 base_model.id |
| `engine` | VARCHAR(16) | vllm/tgi/ollama/tensorrt |
| `engine_version` | VARCHAR(32) | 引擎版本 |
| `deployment_type` | VARCHAR(16) | docker/k8s/process |
| `endpoint_url` | VARCHAR(256) | 服务地址 |
| `api_key` | VARCHAR(256) | 访问密钥(加密存储) |
| `openai_compatible` | BOOLEAN | 是否 OpenAI 兼容 |
| `replicas`/`min_replicas`/`max_replicas` | INT | 副本数 |
| `gpu_type`/`gpu_count` | VARCHAR/INT | GPU 规格 |
| `node_name` | VARCHAR(64) | 节点名 |
| `resource_spec_json` | JSONB | 资源规格 |
| `config_json` | JSONB | 推理参数(max_tokens/tp_size…) |
| `health_status` | VARCHAR(16) | healthy/unhealthy/unknown |
| `last_health_at` | TIMESTAMPTZ | 最近健康检查 |
| `metrics_json` | JSONB | qps/latency/token 用量 |
| `status` | VARCHAR(16) | 状态机见 §4 |
| `started_at`/`stopped_at` | TIMESTAMPTZ | 起止时间 |
| 审计字段 | — | 通用 |

### 1.8 `eval_task` — 评测任务
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `name` | VARCHAR(128) | 任务名 |
| `artifact_id` | BIGINT | 逻辑引用 model_artifact.id（被测模型） |
| `inference_instance_id` | BIGINT | 逻辑引用 inference_instance.id（在线评测，可空） |
| `eval_dataset_id` | BIGINT | 逻辑引用 dataset.id（评测集） |
| `eval_type` | VARCHAR(16) | auto/human/ab/regression |
| `metrics_config_json` | JSONB | 评测指标配置 |
| `metrics_result_json` | JSONB | 评测结果明细 |
| `score` | NUMERIC(10,4) | 总分 |
| `gate_passed` | BOOLEAN | 发布门禁是否通过 |
| `result_uri` | VARCHAR(512) | 详细报告路径 |
| `engine_job_id` | VARCHAR(64) | 引擎任务 ID |
| `status` | VARCHAR(16) | 状态机见 §4 |
| `error_message` | TEXT | 失败原因 |
| `started_at`/`finished_at` | TIMESTAMPTZ | 起止时间 |
| 审计字段 | — | 通用 |

### 1.9 `billing_record` — 计费/用量记录
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `record_type` | VARCHAR(16) | training/inference/storage/annotation/compression |
| `source_type` | VARCHAR(32) | 来源对象类型 |
| `source_id` | BIGINT | 来源对象 ID（多态，无 FK） |
| `sku` | VARCHAR(32) | 计量项: gpu_hour/token/gb_day |
| `quantity` | NUMERIC(18,4) | 数量 |
| `unit` | VARCHAR(16) | 单位 |
| `unit_price` | NUMERIC(18,6) | 单价 |
| `amount` | NUMERIC(18,4) | 金额 |
| `currency` | VARCHAR(8) | 币种: CNY/USD |
| `resource_pool`/`gpu_type` | VARCHAR | 资源信息 |
| `occurred_at` | TIMESTAMPTZ | 发生时间 |
| `period` | VARCHAR(7) | 账期: yyyy-MM |
| `status` | VARCHAR(16) | pending/settled/void |
| `remark` | TEXT | 备注 |
| 审计字段 | — | 通用 |

### 1.10 `schedule_task` — 调度任务
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `name` | VARCHAR(128) | 任务名 |
| `task_type` | VARCHAR(16) | training/compression/eval/inference |
| `biz_type` | VARCHAR(32) | 业务类型（逻辑引用） |
| `biz_id` | BIGINT | 业务 ID（逻辑引用，无 FK） |
| `priority` | INT | 优先级 |
| `preemptible` | BOOLEAN | 是否可被抢占 |
| `queue_name` | VARCHAR(64) | 队列名 |
| `resource_pool` | VARCHAR(64) | 资源池 |
| `required_gpu_type`/`required_gpu_count` | VARCHAR/INT | 需求规格 |
| `allocated_resource_json` | JSONB | 实际分配结果 |
| `node_name` | VARCHAR(64) | 分配节点 |
| `status` | VARCHAR(24) | 状态机见 §4（含 WAITING/PREEMPTED） |
| `retry_count`/`max_retry` | INT | 重试控制 |
| `enqueue_at`/`scheduled_at`/`started_at`/`finished_at` | TIMESTAMPTZ | 时间线 |
| `error_message` | TEXT | 失败原因 |
| 审计字段 | — | 通用 |

---

## 2. 主外键关系（ER 描述）

### 2.1 逻辑 ER（领域视角）
```
tenant (1) ──< (N) 所有业务实体           [tenant_id，逻辑]

base_model (1) ──< (N) training_job        [training_job.base_model_id]
base_model (1) ──< (N) compression_job     [compression_job.source_model_id]
base_model (1) ──< (N) model_artifact      [model_artifact.base_model_id]

dataset (1) ──< (N) training_job           [training_job.dataset_id]
dataset (1) ──< (N) compression_job        [compression_job.calibration_dataset_id]
dataset (1) ──< (N) eval_task              [eval_task.eval_dataset_id]

model_artifact (1) ──< (N) inference_instance   [inference_instance.artifact_id]
model_artifact (1) ──< (N) eval_task            [eval_task.artifact_id]
model_artifact (1) ──< (N) model_artifact       [parent_artifact_id 自引用，血缘]

training_job  (1) ── o (0..1) model_artifact    [training_job.output_artifact_id]
compression_job (1) ─ o (0..1) model_artifact   [compression_job.output_artifact_id]

inference_instance (1) ──< (N) eval_task        [eval_task.inference_instance_id, 可空]

schedule_task (1) ── o (0..1) training_job      [training_job.schedule_task_id，逻辑]

billing_record ── 多态逻辑引用 source_type + source_id（无 FK）
schedule_task  ── 多态逻辑引用 biz_type    + biz_id   （无 FK）
```

### 2.2 物理 FK 判定
| 关系 | 是否建物理 FK | 原因 |
|---|---|---|
| 各表 → tenant | **否** | tenant 属 IAM，跨库 |
| job → base_model / dataset / artifact | **否** | 跨服务 |
| artifact → base_model | **否** | 跨服务 |
| artifact → artifact（血缘） | 同库，**可建**（否则不建） | 自引用 |
| inference_instance → artifact | **否** | 跨服务 |
| eval_task → artifact / dataset / instance | **否** | 跨服务 |
| training_job → schedule_task | **否** | 跨服务 |
| billing/schedule 多态引用 | **否** | 多态，禁止 FK |
| 同一服务内其他引用 | 可建 | 同库一致性 |

> **一致性策略**：跨服务引用由领域事件 + 定时对账补偿；`model_artifact` 通过 `source_type/source_job_id` 反查来源任务，不建 FK。

---

## 3. 必须加索引的字段

> 约定：`tenant_id` 作为复合索引**前导列**；所有状态/时间/外键类字段建索引；软删大表可考虑部分索引 `WHERE deleted = 0`。

### 3.1 `tenant`
- `UNIQUE(code)`
- `INDEX(status)`、`INDEX(isolation_mode)`

### 3.2 `base_model`
- `UNIQUE(tenant_id, name, weight_format)`（同名不同格式允许）
- `INDEX(tenant_id, family, parameter_scale)`
- `INDEX(tenant_id, status)`
- `INDEX(tenant_id, license_commercial_ok)`（许可筛选）
- `INDEX(checksum)`

### 3.3 `dataset`
- `UNIQUE(tenant_id, name, version)`
- `INDEX(tenant_id, task_type)`
- `INDEX(tenant_id, status)`
- `INDEX(tenant_id, is_latest) WHERE deleted = 0`
- `INDEX(label_studio_project_id)`

### 3.4 `compression_job`
- `INDEX(tenant_id, status)`
- `INDEX(tenant_id, created_at DESC)`
- `INDEX(source_model_id)`、`INDEX(source_artifact_id)`
- `INDEX(output_artifact_id)`、`INDEX(calibration_dataset_id)`
- `INDEX(engine_job_id)`

### 3.5 `training_job`
- `INDEX(tenant_id, status)`
- `INDEX(tenant_id, created_at DESC)`（列表默认排序）
- `INDEX(tenant_id, priority DESC, created_at)`（调度队列）
- `INDEX(base_model_id)`、`INDEX(dataset_id)`、`INDEX(output_artifact_id)`
- `INDEX(schedule_task_id)`、`INDEX(engine_job_id)`

### 3.6 `model_artifact`
- `INDEX(tenant_id, artifact_type)`
- `INDEX(tenant_id, stage)`
- `INDEX(tenant_id, status)`
- `INDEX(source_type, source_job_id)`（多态反查）
- `INDEX(base_model_id)`、`INDEX(parent_artifact_id)`
- `INDEX(checksum)`

### 3.7 `inference_instance`
- `UNIQUE(endpoint_url)`
- `INDEX(tenant_id, status)`
- `INDEX(artifact_id)`、`INDEX(base_model_id)`
- `INDEX(engine)`

### 3.8 `eval_task`
- `INDEX(tenant_id, status)`
- `INDEX(tenant_id, created_at DESC)`
- `INDEX(artifact_id)`、`INDEX(eval_dataset_id)`、`INDEX(inference_instance_id)`
- `INDEX(engine_job_id)`

### 3.9 `billing_record`
- `UNIQUE(tenant_id, source_type, source_id, sku, occurred_at)`（幂等去重）
- `INDEX(tenant_id, period)`（账期聚合）
- `INDEX(tenant_id, occurred_at)`（时间范围）
- `INDEX(record_type)`、`INDEX(source_type, source_id)`

### 3.10 `schedule_task`
- `INDEX(tenant_id, status, priority DESC)`（取队首）
- `INDEX(task_type, status)`
- `INDEX(queue_name, priority DESC)`
- `INDEX(biz_type, biz_id)`
- `INDEX(tenant_id, enqueue_at)`

---

## 4. 状态机

### 4.1 `training_job`
```
        submit              resource ok            engine start
DRAFT ─────────► PENDING ──────────────► QUEUED ──────────────► RUNNING
  │                │                       │                       │
  │                │ cancel                │ cancel                ├──► SUCCEEDED  (终态)
  │                ▼                       ▼                       ├──► FAILED ──► PENDING (retry)
  └────────────► CANCELLED ◄───────────────┘                       └──► CANCELLED  (终态)
```
| 迁移 | 触发 | 副作用事件 |
|---|---|---|
| DRAFT→PENDING | 提交任务 | `TrainingJobSubmittedEvent` |
| PENDING→QUEUED | 校验通过，等待资源 | — |
| QUEUED→RUNNING | 分配资源、引擎启动 | — |
| RUNNING→SUCCEEDED | 引擎回调成功 | `TrainingJobSucceededEvent` → artifact 登记 |
| RUNNING→FAILED | 引擎回调失败/超时 | `TrainingJobFailedEvent` |
| FAILED→PENDING | 手动/自动重试（retry_count++） | — |
| PENDING/QUEUED/RUNNING→CANCELLED | 取消 | `TrainingJobCancelledEvent` |

- 终态：SUCCEEDED、CANCELLED；FAILED 可重试。
- 约束：`retry_count < max_retry` 才允许 FAILED→PENDING。

### 4.2 `inference_instance`
```
CREATING ──► DEPLOYING ──► RUNNING ──► UPGRADING ──► RUNNING
                 │            │  ▲
                 │            │  └──── DEGRADED（健康检查失败）
                 │            │              │
                 ▼            ▼              ▼
               FAILED ◄───────┴─────────── STOPPED
                 │                            ▲
                 └──────── redeploy ──────────┘
```
| 迁移 | 触发 |
|---|---|
| CREATING→DEPLOYING | 记录创建，下发部署 |
| DEPLOYING→RUNNING | 健康检查通过 |
| DEPLOYING→FAILED | 部署失败/超时 |
| RUNNING→DEGRADED | 健康检查持续失败（保留流量/降级） |
| DEGRADED→RUNNING | 恢复健康 |
| RUNNING→UPGRADING→RUNNING | 滚动升级 |
| RUNNING/DEGRADED→STOPPED | 手动停止 |
| STOPPED→DEPLOYING | 重新启动 |
| FAILED→DEPLOYING | 重新部署 |

- 终态：无（可按需 STOPPED）；FAILED 可转 DEPLOYING。
- 事件：`EndpointReadyEvent`（RUNNING）、`EndpointStoppedEvent`、`UsageRecordedEvent`（周期性用量）。

### 4.3 `eval_task`
```
PENDING ──► QUEUED ──► RUNNING ──┬──► SUCCEEDED ──► (gate_passed = true/false)
   │           │          │       │
   │           │          │       └──► FAILED ──► PENDING (retry)
   │           │          └──────────► CANCELLED
   └───────────┴─────────────────────► CANCELLED
```
| 迁移 | 触发 | 副作用 |
|---|---|---|
| PENDING→QUEUED | 校验通过、进入队列 | — |
| QUEUED→RUNNING | 资源就绪、开始评测 | — |
| RUNNING→SUCCEEDED | 评测完成 | `EvalCompletedEvent`；计算 `gate_passed` |
| RUNNING→FAILED | 异常 | `EvalFailedEvent` |
| FAILED→PENDING | 重试 | — |
| 任意非终态→CANCELLED | 取消 | — |

- 终态：SUCCEEDED、CANCELLED；FAILED 可重试。
- `gate_passed` 是**结果标记**而非状态：`score >= 门禁阈值` 时为 true，供 artifact 晋级门禁消费。

### 4.4 补充：`schedule_task`（调度器用，供参考）
```
PENDING → WAITING → RESOURCE_ALLOCATED → RUNNING → SUCCEEDED
              │            │                  │
              │            └──► FAILED        └──► PREEMPTED ──► WAITING
              └──► CANCELLED
```

---

## 5. 多租户隔离方案

**结论：混合模式。Phase 1 = `tenant_id` 行级 + PostgreSQL RLS 兜底；Phase 2 = 大客户 schema 级隔离。**

| 阶段 | 方案 | 适用 |
|---|---|---|
| Phase 1 | 行级：全表带 `tenant_id` + MyBatis 插件自动注入 + PG RLS 兜底 | 全部租户（默认） |
| Phase 2 | schema 级：`tenant.isolation_mode='schema'`，独立 PG schema | 大客户/强隔离 |
| Phase 3 | database 级：独立 PG 实例 | 监管/超大户 |

落地要点：
- `tenant` 为全局表（无 tenant_id），是隔离路由的依据。
- 所有业务表 `tenant_id NOT NULL`，复合索引以 `tenant_id` 为前导列。
- 应用层：`adapter` 解析 JWT/Header 写入 `TenantContext`；`infrastructure` 层 MyBatis-Plus `TenantLineInnerInterceptor` 注入；DB 侧开 RLS（`current_setting('app.tenant_id')`）双保险。
- 跨服务：Feign 透传 `X-Tenant-Id`；MQ 事件体携带 `tenantId`；异步任务显式设置 `TenantContext`。
- WS 控制面：握手 JWT 解析 `tenantId` 并绑定连接，任务下发前校验 `taskId` 归属租户（见 §7.6）。
- 私有化默认单租户：`tenant_id` 可固定默认值，字段与机制保留，M2 平滑开启多租户。
- domain 层**不感知租户**（租户是横切关注点）。
- 大表可考虑 `PARTITION BY tenant_id`（Phase 3 再评估）。

---

## 6. 第一版建表 SQL 落地顺序

### 6.1 单库先行顺序（M0 docker-compose，单 PG 多 schema，可建物理 FK）
按依赖拓扑排序：
1. `maas_iam.tenant`（根，无依赖）
2. `maas_base_model.base_model`
3. `maas_dataset.dataset`
4. `maas_artifact.model_artifact`（自引用 parent_artifact_id）
5. `maas_training.training_job`（逻辑引用 2/3/4）
6. `maas_compress.compression_job`（逻辑引用 2/3/4）
7. `maas_serving.inference_instance`（逻辑引用 4）
8. `maas_eval.eval_task`（逻辑引用 3/4/7）
9. `maas_scheduler.schedule_task`（多态引用 5/6）
10. `maas_billing.billing_record`（多态引用 5/7，最后建）

> 顺序原则：**无依赖的根表先建；被引用方先于引用方；多态引用表最后建；避免循环依赖**（已通过移除 artifact→eval FK 打破环）。

### 6.2 服务化顺序（database-per-service，各服务独立迁移）
每个服务的 Flyway 迁移只建自己的表，**不建跨服务 FK**：
| 顺序 | 服务 | 迁移文件 | 表 |
|---|---|---|---|
| 1 | IAM | `V1__create_tenant.sql` | tenant |
| 2 | base-model | `V1__create_base_model.sql` | base_model |
| 3 | dataset | `V1__create_dataset.sql` | dataset |
| 4 | artifact | `V1__create_model_artifact.sql` | model_artifact |
| 5 | training | `V1__create_training_job.sql` | training_job |
| 6 | compress | `V1__create_compression_job.sql` | compression_job |
| 7 | serving | `V1__create_inference_instance.sql` | inference_instance |
| 8 | eval | `V1__create_eval_task.sql` | eval_task |
| 9 | scheduler | `V1__create_schedule_task.sql` | schedule_task |
| 10 | billing | `V1__create_billing_record.sql` | billing_record |

### 6.3 迁移规范
- 命名：`V{n}__{verb}_{table}.sql`；每个服务独立 `src/main/resources/db/migration`。
- 只前进不修改历史迁移；变更用新版本文件。
- 每个 `V1` 末尾同时建通用索引（§3）。
- 初始化数据：`tenant` 默认租户、`base_model` 内置基座（可选 `R__seed.sql`）。
- Phase 2 迁移：`dataset_version` 拆分、`training_run` 多尝试、IAM/role/permission 表。

### 6.4 运行时控制面表落地顺序（PLAN v2.0 新增，详见 §7）
| 顺序 | 服务 | 迁移文件 | 表 |
|---|---|---|---|
| 1 | base-model | `V1__create_base_model.sql`（同批） | model_pull_task |
| 2 | training | `V1__create_runtime_node.sql` | runtime_node |
| 3 | training | `V1__create_training_job.sql`（同批） | task_dispatch |

### 6.5 完整依赖拓扑（含运行时表）
`tenant` → `base_model`/`model_pull_task` → `dataset` → `model_artifact` →
`runtime_node` → `training_job`/`task_dispatch` → `compression_job` →
`inference_instance` → `eval_task` → `schedule_task` → `billing_record`。

---

## 7. 与运行时架构（PLAN v2.0）的对齐

> 本节说明核心 10 表如何与 Python 运行时 + WebSocket 控制面对接；这三张表**不属于**核心 10 实体，但同库共存。

### 7.1 运行时控制面表
**`model_pull_task`（属 base-model-service / `maas_base_model`）**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `base_model_id` | BIGINT | 逻辑引用 base_model.id |
| `source` | VARCHAR(16) | hf/modelscope |
| `repo_id` | VARCHAR(256) | 模型仓库 ID |
| `revision` | VARCHAR(64) | 版本/commit |
| `status` | VARCHAR(16) | pending/running/succeeded/failed/cancelled |
| `progress` | SMALLINT | 进度 0–100 |
| `error_message` | TEXT | 失败原因 |
| `started_at`/`finished_at` | TIMESTAMPTZ | 起止时间 |
| 审计字段 | — | 通用 |

**`runtime_node`（属 training-orchestration / `maas_training`）**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `node_id` | VARCHAR(64) | 运行时节点唯一标识（WS 绑定） |
| `runtime_type` | VARCHAR(16) | puller/compressor/trainer/server |
| `host` | VARCHAR(128) | 主机名/IP |
| `gpu_type` | VARCHAR(32) | GPU 型号 |
| `gpu_count` | INT | 卡数 |
| `gpu_mem_mb` | INT | 单卡显存 |
| `status` | VARCHAR(16) | online/offline/busy |
| `last_heartbeat_at` | TIMESTAMPTZ | 最近心跳（>90s 判离线） |
| `version` | VARCHAR(32) | 组件版本 |
| 审计字段 | — | 通用 |

**`task_dispatch`（属 training-orchestration / `maas_training`）**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `task_id` | VARCHAR(64) | WS 任务 ID（全局唯一） |
| `task_type` | VARCHAR(16) | pull/compress/train/infer |
| `biz_type`/`biz_id` | VARCHAR/ BIGINT | 业务对象（多态逻辑引用，无 FK） |
| `node_id` | VARCHAR(64) | 执行节点 |
| `status` | VARCHAR(16) | assigned/acked/running/succeeded/failed/cancelled/timeout |
| `progress` | SMALLINT | 进度 0–100 |
| `retry_count` | INT | 重试次数 |
| `assigned_at`/`acked_at`/`started_at`/`finished_at` | TIMESTAMPTZ | 时间线 |
| 审计字段 | — | 通用 |

### 7.2 关系
- `model_pull_task.base_model_id` → 逻辑引用 `base_model.id`（库内，可建物理 FK）。
- `task_dispatch.biz_id` → 多态逻辑引用 `training_job.id` / `compression_job.id` / `inference_instance.id`（跨库，不建 FK）。
- `runtime_node.node_id` ↔ WS 连接绑定；`task_dispatch.node_id` 引用当前执行节点。

### 7.3 `engine_job_id` 语义统一
原核心表中的 `engine_job_id` 在 v2.0 中**等价于 WS `taskId`**。建议：
- 保留字段名以兼容查询，但统一存 WS `taskId`；
- 或改名为 `task_id`，与 `task_dispatch.task_id` 对齐（推荐后者，避免歧义）。

### 7.4 状态机与 WS 生命周期对齐
| training_job 迁移 | WS 事件 |
|---|---|
| PENDING→QUEUED | hub 生成 taskId 并落 `task_dispatch` |
| QUEUED→RUNNING | 收到首个 `TASK_ACK(accepted)` 或 `TASK_PROGRESS` |
| RUNNING→SUCCEEDED | 收到 `TASK_RESULT(status=succeeded)` |
| RUNNING→FAILED | 收到 `TASK_ERROR`；或节点离线导致 `task_dispatch=timeout` |
| 任意→CANCELLED | 下发 `TASK_CANCEL` 后收到结果 |

- 节点离线（`runtime_node.status=offline`）→ 在途 `task_dispatch` 置 `timeout` → 对应 `training_job` 置 `FAILED(retryable=true)` 并可按策略重入队。
- 重连对账：`NODE_REGISTER.runningTasks[]` 与 `task_dispatch` 比对修正，避免状态漂移。

### 7.5 运行时表必加索引
- `model_pull_task`：`INDEX(tenant_id, status)`、`INDEX(base_model_id)`。
- `runtime_node`：`UNIQUE(node_id)`、`INDEX(runtime_type, status)`、`INDEX(last_heartbeat_at)`。
- `task_dispatch`：`UNIQUE(task_id)`、`INDEX(tenant_id, status)`、`INDEX(biz_type, biz_id)`、`INDEX(node_id)`。

### 7.6 租户补充
- WS 握手时从 JWT 解析 `tenantId` 并绑定连接；`task_dispatch`、`runtime_node` 均带 `tenant_id`。
- 私有化通常单租户：`tenant_id` 可固定默认值；机制保留，M2 平滑开启多租户。

---

## 8. M1 轻量治理表（eval + guardrail）

> M1 **不独立成服务**，表随现有库部署；M2 拆至 `maas_eval` / `maas_guardrail`。

### 8.1 `eval_set` — 金标评测集（golden set，50–200 条）
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `code` | VARCHAR(64) | 评测集编码 |
| `name` | VARCHAR(128) | 名称 |
| `task_type` | VARCHAR(16) | sft/dpo/qa/... |
| `version` | INT | 版本号 |
| `row_count` | INT | 样本数（建议 50–200） |
| `storage_uri` | VARCHAR(512) | JSONL 路径（MinIO） |
| `checksum` | VARCHAR(64) | 校验和 |
| `status` | VARCHAR(16) | active/archived |
| 审计字段 | — | 通用 |

### 8.2 `eval_result` — 轻量评测结果
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `artifact_id` | BIGINT | 逻辑引用 model_artifact.id |
| `eval_set_id` | BIGINT | 逻辑引用 eval_set.id |
| `eval_type` | VARCHAR(16) | golden/regression/human |
| `score` | NUMERIC(10,4) | 总分 |
| `metrics_json` | JSONB | 指标明细 |
| `passed` | BOOLEAN | 是否通过门禁阈值 |
| `detail_uri` | VARCHAR(512) | 明细报告路径 |
| `status` | VARCHAR(16) | pending/running/succeeded/failed |
| `ran_at` | TIMESTAMPTZ | 执行时间 |
| `error_message` | TEXT | 失败原因 |
| 审计字段 | — | 通用 |

> training 出模型后自动跑，结果回写 `model_artifact.eval_summary_json`。

### 8.3 `guardrail_policy` — 护栏策略（lite）
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `name` | VARCHAR(128) | 策略名 |
| `type` | VARCHAR(24) | length/sensitive_word/pii/output_schema |
| `scope` | VARCHAR(16) | input/output/both |
| `config_json` | JSONB | 规则配置（词表/正则/长度阈值/JSON Schema） |
| `action` | VARCHAR(16) | block/mask/warn |
| `priority` | INT | 优先级 |
| `enabled` | BOOLEAN | 是否启用 |
| 审计字段 | — | 通用 |

### 8.4 `guardrail_log` — 护栏命中日志
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `policy_id` | BIGINT | 逻辑引用 guardrail_policy.id |
| `endpoint_id` | BIGINT | 逻辑引用 inference_instance.id（可空） |
| `direction` | VARCHAR(16) | input/output |
| `action` | VARCHAR(16) | block/mask/pass |
| `matched` | BOOLEAN | 是否命中 |
| `snippet_masked` | TEXT | 脱敏后的片段 |
| `request_id` | VARCHAR(64) | 请求 ID |
| `created_at` | TIMESTAMPTZ | 时间 |
| 审计字段 | — | 通用 |

---

## 9. M2 治理层服务表

### 9.1 model-registry（`maas_model_registry`）
**`registered_model` — 注册模型（逻辑）**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `code` | VARCHAR(64) | 模型编码，UNIQUE(tenant_id, code) |
| `name` | VARCHAR(128) | 名称 |
| `task_type` | VARCHAR(16) | 任务类型 |
| `owner` | VARCHAR(64) | 负责人 |
| `description` | TEXT | 描述 |
| `status` | VARCHAR(16) | active/archived |
| 审计字段 | — | 通用 |

**`registry_model_version` — 注册版本 + model card**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `registered_model_id` | BIGINT | 逻辑引用 registered_model.id |
| `artifact_id` | BIGINT | 逻辑引用 model_artifact.id |
| `version` | INT | 版本号，UNIQUE(tenant_id, registered_model_id, version) |
| `stage` | VARCHAR(16) | dev/staging/prod/archived |
| `status` | VARCHAR(16) | draft/published/deprecated |
| `model_card_json` | JSONB | 模型卡（用途/数据/指标/限制） |
| `source_hash` | VARCHAR(64) | 来源指纹 |
| `eval_summary_json` | JSONB | 评测摘要 |
| `release_notes` | TEXT | 发布说明 |
| `published_at` | TIMESTAMPTZ | 发布时间 |
| 审计字段 | — | 通用 |

**`model_release_approval` — 发布审批**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `registry_model_version_id` | BIGINT | 逻辑引用注册版本 |
| `from_stage` / `to_stage` | VARCHAR(16) | 晋级前后阶段 |
| `status` | VARCHAR(16) | pending/approved/rejected |
| `requested_by` / `approved_by` | BIGINT | 申请人/审批人 |
| `reason` | TEXT | 原因 |
| `decided_at` | TIMESTAMPTZ | 审批时间 |
| 审计字段 | — | 通用 |

### 9.2 inference-gateway（`maas_gateway`）
**`inference_route` — 路由**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `name` | VARCHAR(128) | 路由名，UNIQUE(tenant_id, name) |
| `model_version_id` | BIGINT | 逻辑引用 registry_model_version.id |
| `upstreams_json` | JSONB | 后端列表 + 权重 |
| `strategy` | VARCHAR(24) | round_robin/weighted/least_latency |
| `fallback_json` | JSONB | 降级策略 |
| `cache_enabled` | BOOLEAN | 是否开缓存 |
| `cache_ttl_sec` | INT | 缓存 TTL |
| `status` | VARCHAR(16) | active/disabled |
| 审计字段 | — | 通用 |

**`rate_limit_policy` — 限流**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `scope` | VARCHAR(16) | route/api_key/tenant |
| `scope_id` | BIGINT | 作用对象 ID |
| `qps` | INT | QPS 上限 |
| `burst` | INT | 突发额度 |
| `token_quota` | BIGINT | token 配额（可空） |
| `window_sec` | INT | 窗口秒数 |
| `enabled` | BOOLEAN | 是否启用 |
| 审计字段 | — | 通用 |

**`gateway_api_key` — 网关 API Key**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `name` | VARCHAR(128) | 名称 |
| `key_hash` | VARCHAR(128) | 密钥哈希（不存明文），UNIQUE |
| `key_prefix` | VARCHAR(16) | 前缀（展示用） |
| `status` | VARCHAR(16) | active/revoked/expired |
| `expires_at` | TIMESTAMPTZ | 过期时间 |
| `allowed_routes_json` | JSONB | 授权路由 |
| 审计字段 | — | 通用 |

**`gateway_audit_log` — 网关审计**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `route_id` | BIGINT | 逻辑引用 inference_route.id |
| `api_key_id` | BIGINT | 逻辑引用 gateway_api_key.id |
| `request_id` | VARCHAR(64) | 请求 ID |
| `status_code` | INT | HTTP 状态码 |
| `latency_ms` | INT | 延迟 |
| `prompt_tokens` / `completion_tokens` | INT | token 用量 |
| `cached` | BOOLEAN | 是否命中缓存 |
| `fallback_used` | BOOLEAN | 是否走降级 |
| `created_at` | TIMESTAMPTZ | 时间 |

> 响应缓存放 Redis，不落表。

### 9.3 knowledge-service（`maas_knowledge`，需 pgvector）
**`knowledge_base`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `name` | VARCHAR(128) | 知识库名 |
| `embedding_model` | VARCHAR(128) | 向量模型 |
| `dimension` | INT | 向量维度 |
| `chunk_size` / `chunk_overlap` | INT | 切分参数 |
| `vector_store` | VARCHAR(16) | pgvector/milvus |
| `status` | VARCHAR(16) | active/building/failed |
| 审计字段 | — | 通用 |

**`document`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `knowledge_base_id` | BIGINT | 逻辑引用 knowledge_base.id |
| `name` | VARCHAR(256) | 文档名 |
| `source_uri` | VARCHAR(512) | 原始文件路径 |
| `mime_type` | VARCHAR(64) | MIME 类型 |
| `size_bytes` | BIGINT | 体积 |
| `checksum` | VARCHAR(64) | 校验和 |
| `status` | VARCHAR(16) | uploaded/parsing/indexed/failed |
| `parse_error` | TEXT | 解析错误 |
| 审计字段 | — | 通用 |

**`document_chunk`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `document_id` | BIGINT | 逻辑引用 document.id |
| `chunk_index` | INT | 分片序号 |
| `content` | TEXT | 分片内容 |
| `token_count` | INT | token 数 |
| `embedding` | vector(dim) | pgvector 向量（可空） |
| `vector_ref` | VARCHAR(128) | 外部向量库 ID（可空） |
| `metadata_json` | JSONB | 元数据（页码/标题等） |
| 审计字段 | — | 通用 |

**`retrieval_log`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `knowledge_base_id` | BIGINT | 逻辑引用 |
| `query_hash` | VARCHAR(64) | 查询指纹 |
| `top_k` | INT | 召回数 |
| `latency_ms` | INT | 延迟 |
| `created_at` | TIMESTAMPTZ | 时间 |

### 9.4 prompt-registry（`maas_prompt`）
**`prompt_template`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `code` | VARCHAR(64) | 模板编码，UNIQUE(tenant_id, code) |
| `name` | VARCHAR(128) | 名称 |
| `task_type` | VARCHAR(16) | 任务类型 |
| `description` | TEXT | 描述 |
| `current_version` | INT | 当前版本 |
| `status` | VARCHAR(16) | active/archived |
| 审计字段 | — | 通用 |

**`prompt_version`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `prompt_template_id` | BIGINT | 逻辑引用 |
| `version` | INT | 版本，UNIQUE(tenant_id, prompt_template_id, version) |
| `content` | TEXT | 模板内容 |
| `variables_json` | JSONB | 变量定义 |
| `model_hint` | VARCHAR(64) | 适配模型提示 |
| `status` | VARCHAR(16) | draft/published/archived |
| `published_at` | TIMESTAMPTZ | 发布时间 |
| 审计字段 | — | 通用 |

**`prompt_binding`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `prompt_template_id` | BIGINT | 逻辑引用 |
| `prompt_version_id` | BIGINT | 逻辑引用 |
| `target_type` | VARCHAR(16) | endpoint/route |
| `target_id` | BIGINT | 目标 ID |
| `deployed_at` | TIMESTAMPTZ | 部署时间 |
| 审计字段 | — | 通用 |

---

## 10. M3 治理层服务表

### 10.1 experiment-service（`maas_experiment`）
**`experiment`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `name` | VARCHAR(128) | 实验名 |
| `objective` | VARCHAR(64) | 优化目标（如 eval score） |
| `search_space_json` | JSONB | 超参搜索空间 |
| `strategy` | VARCHAR(16) | random/grid/bayes |
| `status` | VARCHAR(16) | pending/running/succeeded/failed |
| `best_run_id` | BIGINT | 最优 run |
| 审计字段 | — | 通用 |

**`experiment_run`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `experiment_id` | BIGINT | 逻辑引用 |
| `training_job_id` | BIGINT | 逻辑引用 training_job.id |
| `params_json` | JSONB | 本次参数 |
| `metrics_json` | JSONB | 指标 |
| `score` | NUMERIC(10,4) | 评分 |
| `status` | VARCHAR(16) | pending/running/succeeded/failed |
| 审计字段 | — | 通用 |

### 10.2 lineage-service（`maas_lineage`）
**`lineage_node`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `node_type` | VARCHAR(24) | dataset/base_model/artifact/eval/endpoint/prompt |
| `ref_id` | BIGINT | 业务对象 ID |
| `name` | VARCHAR(128) | 展示名 |
| `metadata_json` | JSONB | 元数据 |
| `created_at` | TIMESTAMPTZ | 创建时间 |

**`lineage_edge`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `from_node_id` | BIGINT | 起点节点 |
| `to_node_id` | BIGINT | 终点节点 |
| `relation` | VARCHAR(32) | derived_from/evaluated_by/deployed_to/trained_on/uses_prompt |
| `created_at` | TIMESTAMPTZ | 创建时间 |

### 10.3 template-market（`maas_template`）
**`scenario_template`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID（0=全局） |
| `code` | VARCHAR(64) | 模板编码，UNIQUE(tenant_id, code) |
| `name` | VARCHAR(128) | 名称 |
| `industry` | VARCHAR(64) | 行业 |
| `description` | TEXT | 描述 |
| `latest_version` | INT | 最新版本 |
| `status` | VARCHAR(16) | draft/published/deprecated |
| 审计字段 | — | 通用 |

**`template_version`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `scenario_template_id` | BIGINT | 逻辑引用 |
| `version` | INT | 版本，UNIQUE(tenant_id, scenario_template_id, version) |
| `manifest_json` | JSONB | 清单（数据 schema/prompt/eval set/训练配置） |
| `storage_uri` | VARCHAR(512) | 包路径 |
| `checksum` | VARCHAR(64) | 校验和 |
| `status` | VARCHAR(16) | draft/published/archived |
| `published_at` | TIMESTAMPTZ | 发布时间 |
| 审计字段 | — | 通用 |

**`template_subscription`**
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | PK |
| tenant_id | BIGINT | 租户 ID |
| `scenario_template_id` | BIGINT | 逻辑引用 |
| `template_version_id` | BIGINT | 逻辑引用 |
| `status` | VARCHAR(16) | subscribed/unsubscribed |
| `subscribed_at` | TIMESTAMPTZ | 订阅时间 |
| 审计字段 | — | 通用 |

---

## 11. 新增表索引与建表顺序

### 11.1 新增表必加索引
| 表 | 索引 |
|---|---|
| `eval_set` | `UNIQUE(tenant_id, code, version)`、`INDEX(tenant_id, status)` |
| `eval_result` | `INDEX(tenant_id, artifact_id)`、`INDEX(tenant_id, status)`、`INDEX(eval_set_id)` |
| `guardrail_policy` | `INDEX(tenant_id, enabled, priority)` |
| `guardrail_log` | `INDEX(tenant_id, created_at DESC)`、`INDEX(policy_id)`、`INDEX(request_id)` |
| `registered_model` | `UNIQUE(tenant_id, code)`、`INDEX(tenant_id, status)` |
| `registry_model_version` | `UNIQUE(tenant_id, registered_model_id, version)`、`INDEX(tenant_id, stage)`、`INDEX(artifact_id)` |
| `model_release_approval` | `INDEX(tenant_id, status)`、`INDEX(registry_model_version_id)` |
| `inference_route` | `UNIQUE(tenant_id, name)`、`INDEX(tenant_id, status)` |
| `rate_limit_policy` | `INDEX(tenant_id, scope, scope_id)` |
| `gateway_api_key` | `UNIQUE(key_hash)`、`INDEX(tenant_id, status)` |
| `gateway_audit_log` | `INDEX(tenant_id, created_at DESC)`、`INDEX(route_id)`、`INDEX(api_key_id)`、`INDEX(request_id)` |
| `knowledge_base` | `INDEX(tenant_id, status)` |
| `document` | `INDEX(tenant_id, knowledge_base_id)`、`INDEX(tenant_id, status)` |
| `document_chunk` | `INDEX(document_id, chunk_index)`、`INDEX(tenant_id)`、向量索引（pgvector ivfflat/hnsw） |
| `retrieval_log` | `INDEX(tenant_id, created_at DESC)` |
| `prompt_template` | `UNIQUE(tenant_id, code)`、`INDEX(tenant_id, status)` |
| `prompt_version` | `UNIQUE(tenant_id, prompt_template_id, version)`、`INDEX(tenant_id, status)` |
| `prompt_binding` | `INDEX(tenant_id, target_type, target_id)` |
| `experiment` | `INDEX(tenant_id, status)` |
| `experiment_run` | `INDEX(tenant_id, experiment_id)`、`INDEX(training_job_id)` |
| `lineage_node` | `INDEX(tenant_id, node_type, ref_id)` |
| `lineage_edge` | `INDEX(tenant_id, from_node_id)`、`INDEX(tenant_id, to_node_id)` |
| `scenario_template` | `UNIQUE(tenant_id, code)`、`INDEX(tenant_id, industry, status)` |
| `template_version` | `UNIQUE(tenant_id, scenario_template_id, version)` |
| `template_subscription` | `UNIQUE(tenant_id, scenario_template_id)`、`INDEX(tenant_id, status)` |

### 11.2 新增建表顺序
1. **M1（随现有库）**：`eval_set` → `eval_result`；`guardrail_policy` → `guardrail_log`。
2. **M2（独立库）**：
   - model-registry：`registered_model` → `registry_model_version` → `model_release_approval`
   - inference-gateway：`inference_route` → `rate_limit_policy` → `gateway_api_key` → `gateway_audit_log`
   - knowledge：`knowledge_base` → `document` → `document_chunk` → `retrieval_log`
   - prompt：`prompt_template` → `prompt_version` → `prompt_binding`
3. **M3（独立库）**：
   - experiment：`experiment` → `experiment_run`
   - lineage：`lineage_node` → `lineage_edge`
   - template：`scenario_template` → `template_version` → `template_subscription`

> 原则不变：跨服务只存逻辑引用，**不建物理 FK**；多态引用（`*_type` + `*_id`）不建 FK；新增库同样 `tenant_id` 行级 + RLS。

---

## 附：YAGNI 说明
- 不为 MVP 拆 `dataset_version` / `training_run` / `artifact_version` 独立表，用内联字段兜住。
- 不建跨服务物理 FK、不建多态 FK。
- 不做过早分库分表/分区，Phase 3 视数据量再评估。
- 全量 CQRS 事件溯源、Saga 编排表暂不引入。
- 治理层表按 M1/M2/M3 增量建，**不提前建**；M2/M3 表在未到阶段前不创建。
