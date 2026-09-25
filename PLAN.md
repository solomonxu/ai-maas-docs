# AI MaaS 垂直行业私有化微调平台 — 项目规划 (PLAN)

> 版本: v2.0（规划稿） 日期: 2026-09-25
> 定位: 私有化 / 混合部署的垂类模型微调与推理平台，做「基座无关的中间层」。
> 技术主线: Java 17/21 + Spring Boot 3.x + Spring Cloud Alibaba + COLA DDD + Vue 3 +
> Python 运行时组件 + K8s。
> 控制面通信: MVP 统一 WebSocket 长连接（Java ↔ Python 运行时）。
> 本文只做规划，不含代码；需拍板项见「六、需要拍板的问题」。

---

# 0.5 路线图修订说明（2026-09-25）

## 修订背景

经架构评审，现有 PLAN 覆盖了「能训能推」的核心链路，但在**生产化治理层**存在明显缺口：

- 缺少 **Model Registry**（当前 artifact 表只是对象存储元数据）
- 缺少 **Eval Harness**（评测工程而非单表）
- 缺少 **Guardrails 安全边界层**
- 缺少 **Inference Gateway**（路由/限流/缓存/审计）
- 缺少 **RAG/Knowledge Base** 支持
- 缺少 **Prompt Template Registry**
- 缺少 **Experiment Tracking**
- 缺少 **Cost/Quota/FinOps**
- 缺少 **Data Lineage**
- 缺少 **Scheduler 前置**

## 修订原则

- **不重构**现有架构，只做增量
- **不超前建设**，YAGNI 守住
- 兼容**私有化/离线部署**
- **三阶段目标不变**

## 三阶段增量清单

| 阶段 | 目标（不变） | 增量内容 | 关键原则 |
|---|---|---|---|
| M1 | 跑通单客户闭环 | eval baseline + guardrail-lite + artifact 血缘字段 | 不建新服务也能量化质量 |
| M2 | 多客户可复制 | inference-gateway / model-registry / eval-service / knowledge-service / prompt-registry / guardrail-service | 治理能力服务化 |
| M3 | 平台化复制 | experiment / lineage / template-market / FinOps | 规模化运营与生态 |

## 能力成熟度矩阵

| 能力 | M1 | M2 | M3 |
|---|---|---|---|
| 训练/压缩/推理 | ✅ | ✅ | ✅ |
| 评测门禁 | 轻量（单表 + golden set） | 服务化（回归集 + LLM-judge） | 持续回归 |
| 模型注册 | 字段级（artifact 表扩展） | 服务 + 审批 | 模型卡 + 谱系 |
| 推理网关 | 无 | ✅ 路由/限流/缓存/审计 | 多模型路由 |
| RAG | 不选 | ✅ 文档/chunk/embedding | 多知识域隔离 |
| Prompt 管理 | 写文件 | 版本化 | A/B 测试 + 回滚 |
| 安全护栏 | 长度/敏感词/PII | 服务化 + 审计 | 策略编排 |
| 血缘/实验 | 无 | 埋字段 | 全链路 |
| FinOps | 用量记录 | 配额/预算 | chargeback |

---

# 一、项目总览

## 1. 项目目标与非目标

### 1.1 目标
- **G1**：打通「垂类数据 → 微调 7B/14B → 评测 → 私有化推理」闭环，单客户/单行业/单场景可落地。
- **G2**：沉淀**基座无关**的训练/推理流水线与标准化接口（OpenAI 兼容为主）。
- **G3**：把项目制经验产品化为可复制平台（多租户、可观测、计费、离线部署）。
- **G4**：客户内网/混合云私有化部署，数据不出域。

### 1.2 非目标
- **N1**：不承诺从零训练千亿基座（仅微调/继续预训练/蒸馏/量化）。
- **N2**：不自研推理引擎内核，复用 vLLM / Ollama / TGI / TensorRT-LLM。
- **N3**：不自研通用标注工具（集成 LabelStudio，必要时轻量二开）。
- **N4**：不做公网多租户 SaaS（先私有化/混合）。
- **N5**：500B+ 仅作远期能力（分布式推理/蒸馏），MVP 不交付。

### 1.3 模型档位策略
| 档位 | 目标 | 阶段 |
|---|---|---|
| 7B / 14B | 标准化交付主力，LoRA/QLoRA 微调 | MVP |
| 70B（企业级） | 全参/继续预训练 + 多卡训练、张量并行推理 | 产品化 |
| 500B+ | 分布式推理 / 蒸馏压缩到可交付档位 | 复制期（探索） |

### 1.4 基座无关中间层：抽象边界（核心差异化）
- **基座适配层**：统一模型描述（tokenizer、chat_template、量化格式、上下文长度、商用许可）。
- **推理后端适配层**：`model-server` 内适配 vLLM / TGI / Ollama，对外统一 OpenAI 兼容协议。
- **数据/评测适配层**：统一 dataset schema 与评测口径，行业模板可插拔。
- 原则：基座与引擎只是「可替换实现」，业务服务只依赖接口。

## 2. 里程碑与交付物

### M0 · 验证期（0–2 月）
**目标**：端到端手工打通最小链路，证明可行。
- Java 侧：单体网关 + `base-model / dataset / training-orchestration` 三服务骨架 + `runtime-hub` WS 接入。
- Python 侧：`model-puller` + `model-trainer`(LoRA) + `model-server`(vLLM) 三组件，WS 连接。
- 部署：单机 docker-compose + 1 GPU。
- 判定：效果是否达标、成本是否可接受、能否进内网。

### M1 · MVP（2–5 月）
**目标**：产品化最小闭环，可交付首个试点客户。**模型发布必须有评测分数和 model card，否则不能标记为可交付。**
- 交付 4 服务：base-model、dataset、training-orchestration（含 runtime-hub）、artifact；Spring Cloud Gateway；Vue 3 基础管理台。
  ```text
  # 以下三项 M1 仅做轻量实现，不独立成服务：
  # - eval: golden set 50-200 条，training 出模型后自动跑，出分数
  # - guardrail: 输入长度限制 + 敏感词表 + PII 正则 + 输出 schema 校验，挂 model-server 前
  # - artifact: 表加 stage / parent_artifact_id / source_hash / eval_summary_json 字段
  ```
- 交付 4 运行时组件：puller / compressor / trainer / server（WS 控制面）。
- 集成：PostgreSQL、MinIO、Redis、Kafka/RocketMQ、Nacos、LabelStudio（原生）、vLLM。
- 部署：docker-compose 一键试用 + Helm 私有化；弱多租户 + 用量记录。
- 测试：JUnit5 + Testcontainers + ArchUnit + WS 协议契约测试 + pytest。

### M2 · 产品化（5–9 月）
**目标**：多客户可复制。**M2 是 LLMOps 中台期：补齐治理层服务，支撑多客户、多场景、可治理。**
- 补齐：serving、eval、compress、billing、scheduler 五服务；annotation 二开。
- **新增治理层服务**：
  - **inference-gateway**（推理流量治理：路由/限流/缓存/fallback/审计）
  - **model-registry**（模型版本管理/发布审批/model card）
  - **eval-service**（回归集管理/LLM-as-judge/评测门禁）
  - **knowledge-service**（RAG：文档解析/chunk/embedding/检索）
  - **prompt-registry**（Prompt 模板版本化/部署管理）
  - **guardrail-service**（从 lite 升级为独立服务：策略/规则/审计日志）
- 运行时拆分：`model-server` 增加 HTTP/gRPC 推理端点；`model-trainer` 接入 MQ 做长任务持久化；`model-server` 弹性伸缩。
- 多租户强化、RBAC、审计、配额、可观测（Prometheus/Grafana/Loki/OTel）。
- 离线安装包、升级回滚、License 控制；70B 训练/推理。
- **不做什么**：不碰多模态、AutoML、联邦学习、自研推理内核、公网 SaaS。

### M3 · 复制期（9–15 月）
**目标**：**M3 是治理+飞轮期：新行业客户 1-2 周完成接入，不写新代码只配模板。**
- 场景模板市场、多基座适配、GitOps 多集群、500B 蒸馏/分布式推理、商业计费闭环。
- **新增治理与飞轮能力**：
  - **experiment-service**（超参搜索/多次实验对比/自动报告）
  - **lineage-service**（全链路血缘：data→train→artifact→eval→deploy）
  - **template-market**（行业场景模板的发布/订阅/版本管理）
  - **FinOps**（chargeback/成本趋势/GPU 利用率优化）
  - **数据飞轮**（生产流量采样→进 eval 集→再训练）

## 3. 仓库划分建议

**结论：Monorepo（Maven 多模块 + Python 多 package + 前端 pnpm workspace）起步，M2 后按需拆仓。**

理由：M0–M1 服务边界与运行时协议尚不稳定，统一版本、统一 CI、跨语言协议同步成本低；Java 多模块与 Python 多 package 均可独立打包镜像部署。

```
maas/
├── pom.xml                        # 父 POM：dependencyManagement / 插件 / 版本统一
├── docs/                          # 架构、ADR、接口协议、部署手册
│   ├── adr/
│   ├── api/openapi/               # 聚合后的 OpenAPI 产物
│   ├── protocol/                  # WS 协议 JSON Schema（Java/Python 共享）
│   └── deploy/
├── maas-common/                   # Java 共享基座（无业务语义）
│   ├── common-core/               # 统一返回/异常/错误码/工具/TenantContext
│   ├── common-ddd/                # COLA 基类：Entity/VO/Aggregate/DomainEvent/Repository
│   ├── common-web/                # 全局异常、Swagger、租户拦截器、traceId
│   ├── common-mybatis/            # MyBatis-Plus 配置、租户插件、审计字段填充
│   ├── common-mq/                 # Kafka/RocketMQ 生产/消费封装、Outbox
│   ├── common-redis/              # 缓存、分布式锁、连接注册表
│   ├── common-minio/              # 对象存储客户端、预签名 URL
│   └── common-runtime-ws/         # WS 连接管理、协议模型、连接注册表
├── maas-gateway/                  # Spring Cloud Gateway：路由/鉴权/限流
├── maas-services/
│   ├── base-model-service/
│   ├── dataset-service/
│   ├── training-orchestration/    # MVP 含 runtime-hub
│   ├── artifact-service/
│   ├── serving-service/           # Phase 2
│   ├── eval-service/              # Phase 2
│   ├── compress-service/          # Phase 2
│   ├── billing-service/           # Phase 2
│   └── scheduler-service/         # Phase 2（吸收 runtime-hub）
├── runtimes/                      # Python 运行时组件
│   ├── common/                    # maas-runtime-sdk：连接/协议/存储/日志/指标
│   ├── model-puller/
│   ├── model-compressor/
│   ├── model-trainer/
│   └── model-server/
├── maas-web/                      # Vue 3 + TS + Vite 前端
├── deploy/
│   ├── docker-compose/
│   ├── helm/
│   ├── k8s/                       # 含 GPU Operator / NetworkPolicy
│   ├── nacos/
│   └── offline/                   # 离线包构建
└── .github/workflows/             # 或 GitLab CI
```

**M2 起可拆分**：`maas-web`、`deploy`、`runtimes/common`（发布内部 PyPI）、`maas-templates`（M3）。

---

# 二、后端架构规划

## 4. 每个微服务的 COLA 四层规划

### 4.0 落地形态
- 每个服务 = **1 个 Maven 模块**，内部按 COLA 分包（轻量，M0–M1 推荐）。
- 服务体量大或团队分离时，再升级为「每层一个 Maven 子模块」的重型 COLA。
- 分层依赖：`adapter → app → domain ← infrastructure`；**domain 不依赖任何框架**。

### 4.1 通用 COLA 模板
```
{service}/
├── pom.xml
└── src/main/java/com/maas/{svc}/
    ├── adapter/
    │   ├── web/            # Controller + Request/Response
    │   ├── ws/             # WebSocket Endpoint（仅 runtime-hub）
    │   ├── mq/             # MQ Listener
    │   └── rpc/            # Feign Client（调其他服务）
    ├── app/
    │   ├── service/        # XxxCommandService / XxxQueryService（应用服务，编排）
    │   ├── handler/        # WS 消息 Handler / 事件 Handler
    │   └── dto/            # Cmd/Query DTO + Assembler
    ├── domain/
    │   ├── entity/         # 聚合根/实体
    │   ├── vo/             # 值对象、枚举
    │   ├── service/        # 领域服务（纯业务规则）
    │   ├── repository/     # Repository 接口（端口）
    │   └── event/          # 领域事件定义
    ├── infrastructure/
    │   ├── persistence/    # Repository 实现（MyBatis-Plus）
    │   ├── rpc/            # 外部调用实现（runtime-hub 客户端、第三方）
    │   ├── config/         # 配置类
    │   ├── mq/             # 事件发送实现
    │   └── util/           # 本地工具
    └── bootstrap/{Svc}Application.java
```

### 4.2 base-model-service（基模管理）
- **adapter/web**：`BaseModelController`、`ModelLicenseController`、`ModelPullController`。
- **app/service**：`BaseModelCommandService`、`BaseModelQueryService`、`ModelPullAppService`（调 runtime-hub 下发 pull 任务）。
- **app/dto**：`BaseModelCreateCmd`、`ModelPullCmd`、`BaseModelAssembler`。
- **domain/entity**：`BaseModel`（聚合根）、`ModelVersion`、`ModelPullTask`。
- **domain/vo**：`ModelFamily`、`ParameterScale`、`LicenseType`、`QuantFormat`、`PullSource`。
- **domain/service**：`LicensePolicyService`（商用白名单校验）。
- **domain/repository**：`BaseModelRepository`、`ModelVersionRepository`、`ModelPullTaskRepository`。
- **domain/event**：`BaseModelRegisteredEvent`、`BaseModelStatusChangedEvent`、`ModelPullCompletedEvent`。
- **infrastructure/rpc**：`RuntimeHubClient`（下发 pull 任务）。
- **infrastructure/persistence**：Repository 实现 + Mapper + PO。

### 4.3 dataset-service（数据标注 / 数据集）
- **adapter/web**：`DatasetController`、`DatasetVersionController`、`AnnotationProjectController`。
- **adapter/mq**：`AnnotationCompletedListener`。
- **app/service**：`DatasetCommandService`、`DatasetVersionCommandService`、`AnnotationAppService`。
- **app/dto**：`DatasetCreateCmd`、`VersionUploadCmd`、`AnnotationProjectAssembler`。
- **domain/entity**：`Dataset`（聚合根）、`DatasetVersion`、`AnnotationTask`。
- **domain/vo**：`DataFormat`、`TaskType`、`DatasetStatus`。
- **domain/service**：`DatasetValidationService`（schema/行数/去重校验）。
- **domain/repository**：`DatasetRepository`、`DatasetVersionRepository`、`AnnotationTaskRepository`。
- **domain/event**：`DatasetVersionCreatedEvent`、`DatasetVersionValidatedEvent`、`AnnotationCompletedEvent`。
- **infrastructure/rpc**：`LabelStudioClient`、`BaseModelClient`（Feign）。
- **infrastructure/persistence**：Repository 实现 + Mapper + PO。

### 4.4 training-orchestration（训练编排 + runtime-hub）
- **adapter/web**：`TrainingJobController`、`TrainingRunController`、`RuntimeNodeController`。
- **adapter/ws**：`RuntimeWebSocketEndpoint`（连接握手/鉴权/心跳）、`RuntimeMessageDispatcher`。
- **adapter/mq**：`TrainingEngineEventListener`。
- **app/service**：`TrainingJobCommandService`、`TrainingJobQueryService`、`RuntimeHubAppService`、`TaskDispatchService`（选节点/下发/超时/重试）。
- **app/handler**：`TaskAckHandler`、`TaskProgressHandler`、`TaskResultHandler`、`TaskErrorHandler`、`NodeRegisterHandler`。
- **app/dto**：`TrainingJobCreateCmd`、`TaskAssignMsg`、`TaskProgressMsg`、`TrainingJobAssembler`。
- **domain/entity**：`TrainingJob`（聚合根）、`TrainingRun`、`TrainingEventLog`、`RuntimeNode`、`TaskDispatch`。
- **domain/vo**：`TrainingMethod`、`JobStatus`、`RuntimeType`（puller/compressor/trainer/server）、`NodeStatus`（online/offline/busy）、`TaskStatus`。
- **domain/service**：`TrainingJobStateMachine`、`HyperParamValidator`、`NodeSelectionPolicy`（MVP：单节点 FIFO）。
- **domain/repository**：`TrainingJobRepository`、`TrainingRunRepository`、`RuntimeNodeRepository`、`TaskDispatchRepository`。
- **domain/event**：`TrainingJobSubmittedEvent`、`TrainingJobSucceededEvent`、`TrainingJobFailedEvent`、`RuntimeNodeOfflineEvent`。
- **infrastructure/persistence / mq / config**：Repository 实现、事件发布、连接注册表配置。
- **infrastructure/redis**：`ConnectionRegistry`（nodeId → hub 实例 + topic）。

### 4.5 artifact-service（制品管理）
- **adapter/web**：`ArtifactController`、`ArtifactPromotionController`。
- **adapter/mq**：`TrainingSucceededListener`、`EvalCompletedListener`（M2）。
- **app/service**：`ArtifactCommandService`、`ArtifactQueryService`、`ArtifactLineageService`。
- **domain/entity**：`Artifact`（聚合根）、`ArtifactVersion`、`PromotionRecord`。
- **domain/vo**：`ArtifactType`（Adapter/Fused/Quantized）、`ArtifactStage`（DEV/STAGING/PROD）、`StorageUri`。
- **domain/service**：`PromotionPolicyService`（M1 空实现/人工）。
- **domain/repository**：`ArtifactRepository`、`PromotionRecordRepository`。
- **domain/event**：`ArtifactRegisteredEvent`、`ArtifactPromotedEvent`。
- **infrastructure/rpc**：MinIO 预签名/校验 checksum。

### 4.6 Phase 2 服务（骨架）
| 服务 | 领域核心 | 关键事件 |
|---|---|---|
| serving-service | `Endpoint`、`EndpointDeployment`、`ApiKey`、`InferenceInstance` | `EndpointReadyEvent`、`UsageRecordedEvent` |
| eval-service | `EvalTask`、`EvalDataset`、`EvalRun`、`EvalGate` | `EvalCompletedEvent`、`GatePassedEvent` |
| compress-service | `CompressionJob`、`CompressProfile` | `CompressSucceededEvent` |
| billing-service | `UsageRecord`、`Quota`、`BillingRule` | `QuotaExceededEvent`、`InvoiceIssuedEvent` |
| scheduler-service | `ResourcePool`、`GpuNode`、`ScheduleTask` | `ResourceAllocatedEvent`、`ResourceReleasedEvent` |

## 5. 跨服务通信方式：event-driven vs sync 选边

**结论：读用同步（Feign），状态与生命周期用异步（Kafka/RocketMQ），Java↔Python 运行时用 WebSocket，推理数据面留 HTTP/gRPC。**

| 场景 | 方式 | 理由 |
|---|---|---|
| 前端聚合查询（如任务详情带基座名） | **Sync Feign** | 需即时结果，缓存兜底 |
| 训练成功 → 登记制品 → 触发评测 | **Async Kafka/RocketMQ** | 长流程解耦、可重试、可回放 |
| 用量产生 → 计费 | **Async MQ** | 削峰、最终一致 |
| 状态变更通知 | **Async MQ** | 多订阅方 |
| Java → Python 运行时（拉取/压缩/训练/推理控制） | **WebSocket 长连接** | 内网穿透友好、单连接双向、MVP 简单 |
| Python 运行时 → Java（进度/结果/错误） | **同一 WebSocket** | 复用连接，无需额外回调端口 |
| model-server 推理数据面 | **HTTP（OpenAI 兼容）/ gRPC**（M2） | 低延迟、高吞吐 |

约定：
- 所有跨服务事件走 `maas-common/common-mq`，**禁止服务间直接读对方库**；同步调用只允许「聚合根查询」。
- WebSocket 连接状态是**有状态**的：多 hub 实例时用 Redis 连接注册表 + Pub/Sub 路由（见 §13）。
- Java 侧为任务状态的**唯一事实来源**；WS 仅作传输通道。

## 6. 服务批次：第一批 vs 第二批

### 第一批（MVP，4 个）
1. **base-model-service** — 基座登记 + 触发 model-puller 拉取。
2. **dataset-service** — 数据入口 + LabelStudio 集成。
3. **training-orchestration** — 核心编排，**并承载 runtime-hub**（WS 接入/任务下发/状态回收）。
4. **artifact-service** — 训练产物落库，闭合「训练→制品」链路。

### 第二批（Phase 2，5 个）
serving-service、eval-service、compress-service、billing-service、scheduler-service。
（M2 将 runtime-hub 从 training-orchestration 抽到 scheduler-service；annotation 二开并入 dataset-service。）

> MVP 收敛原则：先跑通「注册/拉取基座 → 导入数据 → 提交训练 → 产物登记 → 推理 endpoint」一条链。

## 7. 数据表归属（database-per-service，禁止跨服务 join）

| 服务 | Database | 核心表 |
|---|---|---|
| base-model-service | `maas_base_model` | `base_model`、`model_version`、`model_license`、`model_pull_task` |
| dataset-service | `maas_dataset` | `dataset`、`dataset_version`、`annotation_task` |
| training-orchestration | `maas_training` | `training_job`、`training_run`、`runtime_node`、`task_dispatch`、`training_event_log`、`outbox_event` |
| artifact-service | `maas_artifact` | `artifact`、`artifact_version`、`promotion_record`、`artifact_lineage` |
| （公共）| `maas_iam` | Phase2：`tenant`、`user`、`role`、`permission`、`api_key` |
| serving-service | `maas_serving` | Phase2：`inference_instance`、`endpoint`、`api_key` |
| eval-service | `maas_eval` | Phase2：`eval_task` |
| compress-service | `maas_compress` | Phase2：`compression_job` |
| billing-service | `maas_billing` | Phase2：`billing_record` |
| scheduler-service | `maas_scheduler` | Phase2：`schedule_task`、`resource_pool` |

- 每个服务独立 PostgreSQL database / schema / 账号，**跨服务只允许 ID 引用 + 事件同步**，绝不 join。
- 表均含：`id`(雪花/ULID)、`tenant_id`、`created_at`、`updated_at`、`created_by`、`deleted`。
- 事件可靠性：用 **Outbox 表**落地后再由 Relay 投递 MQ（Relay 用 `FOR UPDATE SKIP LOCKED` 防重复）。

## 8. 与 Python 运行时组件的交互边界

**结论：北向只经 API 网关；Python 运行时组件不直接对外，统一连到 Java 的 `runtime-hub`。**

- **北向（前端→Java）**：统一走 `maas-gateway`（鉴权/限流/路由）。
- **Java ↔ Python（控制面）**：所有运行时组件主动向 `runtime-hub`（MVP 在 training-orchestration）建立 **WebSocket 长连接**，地址 `wss://{host}/ws/runtime`。
  - 运维类任务（pull/compress/train）由对应业务服务发命令给 runtime-hub，hub 通过连接下发。
  - 运行时通过同一连接上报状态/进度/结果/日志/错误。
- **数据面**：模型/数据集/产物一律走 MinIO，消息只传 URI + checksum，不传大文件。
- **推理数据面（M2）**：`model-server` 额外暴露内网 HTTP（OpenAI 兼容）+ gRPC 端点（ClusterIP），由 serving-service/gateway 反代与鉴权；不直接对公网。
- **部署**：Python 组件以 K8s Deployment/StatefulSet 运行在 GPU 节点，Service 用 ClusterIP，NetworkPolicy 仅允许访问 hub 与 MinIO。

## 9. 多租户隔离方案

**结论：Phase 1 = `tenant_id` 行级隔离 + PostgreSQL RLS 兜底；Phase 2 = 大客户 schema 级隔离。混合模式。**

- **adapter 层**：`TenantContextFilter` 从 JWT/Header 解析 `tenant_id`，写入 `TenantContext`(TransmittableThreadLocal)；WS 握手时从 token 解析并绑定连接。
- **app 层**：只透传，不做租户判断，保持业务纯净。
- **domain 层**：**完全不感知租户**（租户是横切关注点，不进领域模型）。
- **infrastructure 层**：MyBatis-Plus `TenantLineInnerInterceptor` 自动注入 `tenant_id`；DB 开 RLS（`current_setting('app.tenant_id')`）兜底。
- **跨服务**：Feign 透传 `X-Tenant-Id`；MQ 事件体携带 `tenantId`；WS 信封携带 `tenantId`。
- **异步/定时任务**：显式设置 `TenantContext`，避免串租户。
- **私有化注意**：私有化部署通常单租户，tenant_id 可固定为默认值；字段与机制保留，便于 M2 平滑开启多租户。

---

# 三、运行时组件专项规划

## 10. 各组件职责边界与不做什么（YAGNI）

| 组件 | 做什么 | **明确不做（YAGNI）** |
|---|---|---|
| **model-puller** | 从 HF/ModelScope 拉取模型权重/配置；校验 checksum；上传 MinIO；回报元信息与进度 | 不做格式转换、不做量化、不做许可合规判断（由 Java `LicensePolicyService` 做）、不碰 GPU |
| **model-compressor** | 量化（AWQ/GPTQ）、剪枝、蒸馏；产出压缩制品并上传 MinIO | 不做从零训练、不做评测打分、不做 OCR/多模态 |
| **model-trainer** | 全参/LoRA/QLoRA/DPO 微调；断点续训；指标推送；产出 adapter/fused 制品 | 不做数据清洗/标注、不做调度决策（听 hub 派发）、不做推理 |
| **model-server** | 加载模型、在线推理、流式输出、健康检查；M2 弹性伸缩 | 不做训练/压缩、不做鉴权（网关/serving 负责）、不做计费（上报用量即可） |
| **runtime 公共** | 连接管理、协议编解码、心跳/重连、MinIO 封装、结构化日志、指标 | 不含任何业务语义 |

> 原则：运行时只做「算力执行器」，所有业务状态、权限、编排都在 Java 侧。

## 11. 统一 WebSocket 通信协议设计

### 11.1 连接与握手
- 地址：`wss://{host}/ws/runtime`
- 鉴权：连接后客户端发送首帧 `AUTH`（避免 token 出现在 URL/日志），Java 校验后回 `AUTH_ACK`；未鉴权连接 10s 内关闭。
- 连接绑定：`(runtimeType, nodeId)` 唯一；重复连接**踢旧连新**。
- 心跳：应用层 `PING/PONG`，30s 一次；90s 无心跳标记节点 `offline`。
- 重连：指数退避（1s→2s→…→max 30s），重连后发送 `NODE_REGISTER` 上报在途任务，由 Java 对账。

### 11.2 消息信封（JSON）
| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `type` | string | 是 | 消息类型 |
| `msgId` | string(uuid) | 是 | 消息唯一 ID，用于幂等与 ACK |
| `taskId` | string | 条件 | 任务类消息必填 |
| `runtimeType` | string | 是 | puller/compressor/trainer/server |
| `nodeId` | string | 是 | 节点唯一标识 |
| `tenantId` | long | 是 | 租户 ID |
| `timestamp` | long | 是 | 毫秒时间戳 |
| `traceId` | string | 否 | 链路追踪 ID |
| `payload` | object | 否 | 业务负载 |

### 11.3 消息类型
**运行时 → Java：**
| type | 说明 | payload 关键字段 |
|---|---|---|
| `AUTH` | 连接鉴权 | `token` |
| `NODE_REGISTER` | 节点注册/重连对账 | `capabilities`(gpu型号/数量/显存), `runningTasks[]` |
| `NODE_STATUS` | 资源状态上报 | `gpuUtil`, `memUsed`, `load`, `status` |
| `PING` / `PONG` | 心跳 | — |
| `TASK_ACK` | 任务受理确认 | `accepted`(bool), `reason` |
| `TASK_PROGRESS` | 进度 | `phase`, `progress`(0-100), `step`, `totalSteps`, `metrics{}`, `etaSeconds` |
| `TASK_LOG` | 日志分片 | `level`, `message`, `seq` |
| `TASK_RESULT` | 结果 | `status`, `outputUri`, `checksum`, `sizeBytes`, `metrics{}`, `durationMs` |
| `TASK_ERROR` | 错误 | `code`, `message`, `detail`, `retryable` |

**Java → 运行时：**
| type | 说明 | payload 关键字段 |
|---|---|---|
| `AUTH_ACK` | 鉴权结果 | `ok`, `serverTime`, `heartbeatInterval` |
| `TASK_ASSIGN` | 下发任务 | `component`, `spec{}`（见下）, `timeoutSeconds` |
| `TASK_CANCEL` | 取消任务 | `reason` |
| `PING` / `PONG` | 心跳 | — |
| `CONFIG_UPDATE` | 动态配置 | `params{}` |

### 11.4 `TASK_ASSIGN.spec` 按组件区分
| 组件 | spec 字段 |
|---|---|
| model-puller | `source`(hf/modelscope), `repoId`, `revision`, `files[]`, `targetUri`, `hfTokenRef`(引用Secret名，不下发明文) |
| model-compressor | `sourceUri`, `method`(awq/gptq/prune/distill), `targetQuant`, `calibrationUri`, `outputUri`, `config{}` |
| model-trainer | `baseModelUri`, `datasetUri`, `method`(lora/qlora/full/dpo), `hyperparams{}`, `outputUri`, `resumeFrom`, `tensorParallel`, `gpuCount` |
| model-server | `modelUri`, `engine`(vllm/tgi/ollama), `tensorParallel`, `maxModelLen`, `gpuMemoryUtil`, `port`, `replicas` |

### 11.5 错误码
| code | 含义 | 可重试 |
|---|---|---|
| 1000 | OK | — |
| 4001 | AUTH_FAILED | 否 |
| 4002 | TOKEN_EXPIRED | 否（重连换 token） |
| 4003 | RUNTIME_TYPE_MISMATCH | 否 |
| 4004 | NODE_ALREADY_CONNECTED | 否 |
| 4100 | INVALID_PAYLOAD | 否 |
| 4101 | TASK_NOT_FOUND | 否 |
| 4102 | TASK_ALREADY_RUNNING | 否 |
| 4103 | TASK_CANCELLED | 否 |
| 4200 | GPU_OOM | 是（降 batch/换卡） |
| 4201 | GPU_UNAVAILABLE | 是 |
| 4300 | MODEL_DOWNLOAD_FAILED | 是 |
| 4301 | MODEL_FORMAT_UNSUPPORTED | 否 |
| 4302 | CHECKSUM_MISMATCH | 是（重拉） |
| 4400 | TRAINING_FAILED | 是（重试/调参） |
| 4401 | DATASET_INVALID | 否 |
| 4500 | INFERENCE_LOAD_FAILED | 是 |
| 5000 | INTERNAL_ERROR | 是 |

### 11.6 可靠性与一致性
- **至少一次投递**：任务类消息需 `TASK_ACK`；未 ACK 超时（15s）重发，最多 3 次。
- **幂等**：以 `msgId` 去重；`taskId` 状态迁移由 Java 状态机幂等守卫（只允许合法迁移）。
- **状态对账**：节点重连上报 `runningTasks`，Java 以 DB 为准修正。
- **节点离线**：>90s 无心跳 → `offline`；在途任务按策略标记 `FAILED`(retryable=true) 并重新入队。
- **顺序**：单任务内消息按 `timestamp` 排序；跨任务可并发。

### 11.7 典型消息流
```
runtime              Java runtime-hub                业务服务
   │  --AUTH------------------>│
   │  <----------AUTH_ACK------│
   │  --NODE_REGISTER--------->│
   │                           │<-- 训练命令(training-orchestration)
   │  <--TASK_ASSIGN-----------│
   │  --TASK_ACK-------------->│
   │  --TASK_PROGRESS(period)->│  ── 转发/落库 ──► 更新 training_run
   │  --TASK_LOG-------------->│
   │  --TASK_RESULT----------->│  ── MQ: TrainingJobSucceededEvent ──► artifact-service
   │                           │
```

## 12. 各组件技术选型与目录结构建议

### 12.1 公共 SDK（`runtimes/common`，发布内部 wheel `maas-runtime-sdk`）
```
runtimes/common/
├── connector.py      # RuntimeConnector 基类：连接/重连/心跳/收发/ACK
├── protocol.py       # pydantic 消息模型 + 枚举（type/错误码）— 与 docs/protocol 对齐
├── config.py         # pydantic-settings：HUB_URL/TOKEN/MINIO/NODE_ID
├── storage.py        # MinIO/S3 封装：上传/下载/checksum/预签名
├── logging.py        # structlog + traceId 注入
├── metrics.py        # prometheus-client 指标
└── errors.py         # 异常 → 错误码映射
```
各组件复用 `RuntimeConnector`，只实现 `handler.py` 的任务派发。

### 12.2 组件选型
| 组件 | Python | 关键依赖 | 通信/存储 |
|---|---|---|---|
| model-puller | 3.11 | huggingface_hub, modelscope, boto3/minio, websockets, pydantic | WS + MinIO |
| model-compressor | 3.11 | torch, transformers, autoawq, auto-gptq, peft, datasets, accelerate | WS + MinIO |
| model-trainer | 3.11 | torch, transformers, peft, deepspeed, accelerate, bitsandbytes, datasets, safetensors | WS + MinIO |
| model-server | 3.11 | vllm, fastapi, uvicorn, transformers, websockets, prometheus-client | WS(控制) + HTTP(M2) + MinIO |
| 测试 | — | pytest, pytest-asyncio, jsonschema | 协议契约测试 |

### 12.3 目录结构（在用户给定基础上补充 common）
```
runtimes/
├── common/                # maas-runtime-sdk
├── model-puller/
│   ├── src/{main.py,connector.py,handler.py,pullers/,storage.py}
│   └── Dockerfile
├── model-compressor/
│   ├── src/{main.py,connector.py,handler.py,compressors/,storage.py}
│   └── Dockerfile
├── model-trainer/
│   ├── src/{main.py,connector.py,handler.py,trainers/,checkpoint.py,monitor.py}
│   └── Dockerfile
└── model-server/
    ├── src/{main.py,connector.py,inference.py,api.py,scaler.py}
    └── Dockerfile
```
> `connector.py` 建议改为直接从 `maas-runtime-sdk` 导入，避免四份复制。

## 13. 部署拓扑（K8s）

### 13.1 组件工作量与资源规格
| 组件 | 工作负载 | 副本 | CPU | 内存 | GPU | 节点 | 说明 |
|---|---|---|---|---|---|---|---|
| model-puller | Deployment | 1 | 2c | 4Gi | 无 | 海外 CPU 节点(HK/SG) | 需出网到 HF/ModelScope；本地缓存 PVC |
| model-compressor | Deployment/Job | 0–1 | 4–8c | 32Gi | 1×24GB | 内网 GPU 节点 | 按需/常驻 |
| model-trainer | StatefulSet/Deployment | 1/资源池 | 8–16c | 64–128Gi | 1–8 | 内网 GPU 节点(taint) | 本地 NVMe 存 checkpoint |
| model-server | Deployment + HPA(M2) | 1..N | 8c | 32Gi | 1 | 内网 GPU 节点 | HTTP 8000 + WS |
| runtime-hub | 随 training-orchestration | ≥1 | 2c | 4Gi | 无 | 内网 | 有状态，见 13.3 |

### 13.2 K8s 要点
- 命名空间：`maas-system`（Java）、`maas-runtime`（Python）。
- GPU：`nodeSelector: nvidia.com/gpu.present=true`、`resources.limits: nvidia.com/gpu: N`、tolerations 对应 taint；依赖 GPU Operator。
- NetworkPolicy：仅允许 `maas-runtime` → hub、MinIO、Nacos、MQ；禁止外网（puller 除外）。
- `runtime-hub` 的 Ingress/Service 必须把 `proxy_read_timeout`（Nginx）调大（如 3600s）以支持长连接；开 TLS。
- 模型缓存/checkpoint 用 PVC；MinIO 存最终制品。
- Secret：MinIO 凭据、HF token、WS bootstrap token，挂载为文件或 env，不硬编码。

### 13.3 有状态连接的路由方案（关键）
WebSocket 连接绑定在某个 hub 实例上。多副本时：
- `ConnectionRegistry`（Redis）：`nodeId → hubInstanceId`，附带 `tenantId/runtimeType`。
- 业务服务要下发任务时，通过 Redis Pub/Sub 发到持有连接的实例（或由 hub 集群内转发），避免"任务找不到连接"。
- 或 MVP 直接让 **runtime-hub 单副本**（简单），M2 再上注册表 + Pub/Sub。
- 连接优雅下线：滚动更新前发 `DRAIN`，等待在途任务结束或迁移。

## 14. 安全考虑

### 14.1 连接鉴权
- 启动引导：组件从 K8s Secret 读取**一次性 bootstrap token**，向 Java `POST /api/v1/runtime/auth/exchange` 换取**短期 JWT**（5–15min）；或 Java 直接签发绑定 `(runtimeType,nodeId,tenantId)` 的 token。
- WS 首帧 `AUTH` 携带 JWT；校验签名、过期、`runtimeType` 与连接声明一致。
- 一个 `nodeId` 同时仅一个连接（踢旧连新）；连接建立/断开写审计日志。
- token 过期策略：长时间连接由 Java 主动下发 `CONFIG_UPDATE` 或要求重连；重连走 rename。

### 14.2 敏感信息保护
- HF token、MinIO 凭据、API Key 一律放 K8s Secret/Vault；**协议 payload 只传 Secret 引用名**，不传明文。
- `model-server` 的 API Key 加密存储、日志脱敏；训练数据/model 只走 MinIO URI。
- 禁止在 `TASK_LOG`/错误 payload 中回传密钥或用户数据原文。

### 14.3 内网通信加密
- WS 一律 `wss`（Ingress TLS 终止或 mTLS）。
- 内网可选 mTLS（hub ↔ runtime），产品化阶段启用。
- NetworkPolicy 默认拒绝，最小放行；运行时容器非 root、只读 rootfs、限制 capabilities。
- 限流与防护：单连接最大帧 1MB、消息速率上限、任务白名单、`taskId` 越权校验（校验 tenant 归属）。

---

# 四、前端架构规划

## 15. 技术选型确认
| 项 | 选型 | 说明 |
|---|---|---|
| 框架 | Vue 3 + `<script setup>` | Composition API |
| 语言 | TypeScript（strict） | 全量类型 |
| 构建 | Vite | 快速 HMR |
| 状态 | Pinia | 按模块分 store |
| 路由 | Vue Router 4 | 动态路由 + 权限守卫 |
| UI | Element Plus（备选 Naive UI） | 企业后台生态成熟 |
| 请求 | axios + 统一封装 | 拦截器注入 token/tenant/traceId |
| 类型 | openapi-typescript 生成 | 后端 OpenAPI → TS 类型 |
| 图表 | ECharts | 训练曲线/用量 |
| 实时 | WebSocket/SSE（M2） | 训练进度实时推送，MVP 轮询 |

## 16. 前端目录结构
```
maas-web/
├── index.html
├── vite.config.ts
├── tsconfig.json
├── .env.development / .env.production
└── src/
    ├── main.ts
    ├── App.vue
    ├── api/                 # 按服务分文件，类型来自 openapi
    │   ├── client.ts        # axios 实例 + 拦截器
    │   ├── baseModel.ts
    │   ├── dataset.ts
    │   └── training.ts
    ├── types/
    │   └── openapi.d.ts
    ├── stores/              # Pinia
    │   ├── user.ts
    │   ├── tenant.ts
    │   └── runtime.ts       # 运行时节点列表/状态
    ├── router/
    │   ├── index.ts
    │   └── guards.ts
    ├── pages/
    │   ├── base-model/      # 列表/详情/注册/拉取
    │   ├── dataset/         # 数据集/版本/标注入口
    │   ├── training/        # 提交/列表/详情/日志/曲线
    │   ├── runtime/         # 运行时节点/任务下发状态
    │   └── artifact/        # 制品列表/晋级
    ├── components/
    ├── layouts/
    ├── composables/         # usePagination/usePolling/useRuntimeWs
    ├── utils/
    └── assets/
```

## 17. 前后端协作
- 后端聚合 OpenAPI：CI 用 `springdoc-openapi` 输出各服务 spec → 合并到 `docs/api/openapi/maas.openapi.json`。
- 前端 `pnpm gen:api` 调 `openapi-typescript` 生成 `src/types/openapi.d.ts`；`src/api/*.ts` 薄封装。
- `client.ts` 统一：baseURL、token、`X-Tenant-Id`、`X-Trace-Id`、错误码→提示、401 跳登录。
- 运行时状态前端**不直连 Python**，只读 Java 聚合接口（`/api/v1/runtime/nodes`、`/tasks/{id}`）。
- 破坏性变更由 CI 做 **schema diff 校验**，不一致则前端构建失败。

## 18. 前端部署
- **独立静态部署**：`Vue build → Nginx 镜像`，Phase 1 用 docker-compose 起 `maas-web` + `maas-gateway`。
- Nginx：静态资源 + `/api` 反代到 gateway（同源免 CORS）；若 M2 用 SSE/WS 推送，需配置长连接超时。
- 私有化：Helm chart 内 Nginx Ingress；离线包内置镜像。
- 不嵌入后端 jar（保持前后端独立发布）。

---

# 五、落地顺序与风险

## 19. 落地顺序

### 19.1 后端 + 运行时骨架顺序（M0）
1. 父 POM + `maas-common/*`（core/web/ddd/mybatis/minio/mq/redis/runtime-ws）。
2. `runtimes/common`（SDK）+ WS 协议模型（与 `docs/protocol` 同步）。
3. `maas-gateway` + Nacos 通路；`runtime-hub` WS 端点（先单副本）。
4. `base-model-service`（CRUD + 触发 pull）。
5. `dataset-service`（MinIO 上传 + 校验）。
6. `training-orchestration`（下发 train + 状态机 + 事件）。
7. `artifact-service`（训练成功事件 → 登记）。
8. `model-puller` / `model-trainer` / `model-server` 三个 Python 组件接入 WS。
9. `maas-web` 基础页面 + OpenAPI 生成。
10. `deploy/docker-compose` 一键起栈。

### 19.2 数据表顺序
1. `model_pull_task` / `base_model` → 2. `dataset`/`dataset_version` → 3. `runtime_node`/`task_dispatch`/`training_job`/`training_run`/`outbox_event` → 4. `artifact`/`artifact_version`/`promotion_record`。
（Flyway，每服务独立 migration 目录。）

### 19.3 接口顺序
1. `POST/GET /api/v1/base-models`、`.../pull`
2. `POST/GET /api/v1/datasets`、`.../versions`、`.../validate`
3. `POST/GET /api/v1/training-jobs`、`.../cancel`、`.../runs/{id}/logs`
4. `GET /api/v1/artifacts`、`.../promote`
5. `GET /api/v1/runtime/nodes`、`GET /api/v1/runtime/tasks/{id}`
6. **WS**：`GET /ws/runtime`（升级握手）
7. Phase2：`/v1/chat/completions`（OpenAI 兼容）、`/v1/evals`、`/v1/usage`

### 19.4 测试顺序
1. **领域单测**：状态机、校验、许可策略、节点选择。
2. **ArchUnit**：守护 COLA 分层依赖。
3. **Repository 集成测试**：Testcontainers(PostgreSQL) 验证租户插件/RLS。
4. **Controller 测试**：`@WebMvcTest` + MockMvc + 异常/鉴权。
5. **WS 协议契约测试**：Java 与 Python 共用 `docs/protocol` JSON Schema，双向校验消息。
6. **pytest**：Python 组件 handler 单测 + mock hub 的端到端。
7. **E2E smoke**：compose 起栈后跑「注册基座→pull→上传数据→提交训练(mock/真实小模型)→出制品→起 server」。
8. Phase2：真实 GPU 回归 + 压缩/评测对接。

### 19.5 CI 顺序
1. M0：`mvn verify`（单测 + ArchUnit）+ `pytest` + 前端 `lint + typecheck + build`。
2. M1：Java/Python 镜像构建推送 + compose 起栈 + API 集成 + WS 契约 + openapi diff。
3. M2：e2e + SAST/镜像扫描 + Helm 部署 staging + 制品签名。
4. M3：多环境 GitOps + 灰度/回滚。

### 19.6 部署顺序
1. M0：单机 docker-compose（pg/minio/redis/nacos/mq + 4 服务 + gateway + web + 运行时 `gpu` profile）。
2. M1：kind/k3s + Helm，MinIO/PG 内置，GPU Operator + vLLM GPU 节点。
3. M2：生产 K8s + 可观测栈 + 离线安装包。
4. M3：多集群 / 混合云 / 边缘推理。

### 19.7 docker-compose 服务清单
| 服务 | 用途 | profile |
|---|---|---|
| `postgres` | 各服务独立 database | 默认 |
| `minio` | 数据集/模型/制品对象存储 | 默认 |
| `redis` | 缓存、锁、连接注册表 | 默认 |
| `nacos` | 注册 + 配置中心 | 默认 |
| `rocketmq-namesrv`/`broker`（或 Kafka） | 领域事件 | 默认 |
| `maas-gateway` | API 网关 + WS 入口 | 默认 |
| `base-model-service`/`dataset-service`/`training-orchestration`/`artifact-service` | 第一批服务 | 默认 |
| `maas-web` | Nginx + 静态资源 | 默认 |
| `model-puller` | 模型拉取（可出网） | `puller` |
| `model-trainer`/`model-server` | GPU 运行时 | `gpu` |
| `model-compressor` | 压缩运行时 | `gpu` |
| `labelstudio` | 标注平台 | `annotation` |
| `prometheus`/`grafana`/`loki` | 可观测 | `observability` |

### 19.8 硬件基线（MVP POC vs 生产私有化）
| 阶段 | 硬件 | 用途 | 承诺 |
|---|---|---|---|
| 本地开发 / M0–M1 POC | AMD Ryzen AI MAX+ 395（128GB 统一内存） | 本地开发、7B/14B 推理、LoRA/QLoRA 验证 | **ROCm 实验级支持，不承诺生产 SLA** |
| 生产私有化（M2 起交付基线） | NVIDIA A100 80G / L20 / 4090（CUDA 12.4） | 企业级训练 / 推理交付 | 生产 SLA |

- 开发机为统一内存（无独立大显存）：7B 可跑，14B 需 QLoRA 低显存配置，**70B 不可行**。
- **ROCm 属实验级**：镜像/依赖与 CUDA 路径分离，避免污染生产镜像；vLLM/DeepSpeed 兼容性以 NVIDIA 为唯一验收基线。
- 生产镜像以 **CUDA 12.4** 为准；驱动/CUDA 矩阵为交付前置（见 20.1）。
- 交付给客户的 GPU 从 `A100 80G / L20 / 4090` 中选一档作为基线，其他型号不承诺。

## 20. 风险、待确认问题、MVP 范围

### 20.1 风险
| 风险 | 影响 | 缓解 |
|---|---|---|
| GPU 稀缺/成本高 | 交付延期 | 先 7B + LoRA/QLoRA；云上验证；按需调度 |
| 客户内网无外网 | 模型无法拉取 | 海外节点 pull 后经 MinIO/摆渡包导入；离线包 |
| 模型格式兼容（HF↔adapter↔vLLM↔GGUF） | 训练成功但推理失败/掉点 | 锁定少量已验证路径；制品固化 tokenizer/chat_template 版本 |
| WS 长连接在私有化网关被掐 | 任务中断 | 心跳 + 重连 + 断点续训；Ingress 调大超时；协议幂等 |
| hub 多副本连接路由 | 任务下发失败 | Redis 连接注册表 + Pub/Sub；MVP 单副本 |
| 运行时断线任务丢失 | 状态漂移 | Java 持有状态，重连对账，可重试错误码 |
| GPU 驱动/CUDA 版本不匹配 | 组件起不来 | 锁定已验证 OS/CUDA/驱动矩阵；镜像内置匹配 torch |
| 开发机 ROCm 与生产 CUDA 行为差异 | POC 通过但生产失败 | ROCm 仅作实验验证；NVIDIA CUDA 12.4 为唯一验收基线；镜像分离 |
| 数据合规/泄露 | 法务与信任 | 不出域、加密、审计、脱敏、RLS |
| 基座 License 商用限制 | 法律风险 | `LicensePolicyService` 白名单门禁 |
| 评测不可量化 | 无法验收 | 固定评测集 + 门禁 + 业务指标 |
| LabelStudio 二开成本 | 拖慢进度 | M1 原生，M2 再二开 |
| MQ/事件一致性 | 状态漂移 | Outbox + 幂等消费者 + 重试 |
| 租户串数据 | 安全事件 | 拦截器 + MyBatis 插件 + RLS 双保险 |
| 多租户/配额/计费无执行点 | 伪能力 | M1 明确单租户 + 仅记录，不承诺执行 |

### 20.2 最小可上线范围（MVP = M1）
- 文本单模态；7B/14B；LoRA/QLoRA。
- 4 服务：base-model / dataset / training-orchestration（含 runtime-hub）/ artifact。
- 运行时：puller / trainer / server（WS）；compressor 可在 M1 末或 M2。
- 数据：CSV/JSONL 导入 + 原生 LabelStudio。
- 训练：单机单/多卡，提交/监控/取消；单任务串行。
- 制品：版本化 + 人工晋级。
- 推理：vLLM OpenAI 兼容 endpoint，单 API Key。
- 部署：docker-compose 试用 + 私有化 Helm。
- 计费：仅用量记录，不出账单；权限：弱多租户（MVP 可单租户）。

## 21. 先做 / 不做（YAGNI）

### 先做（Now）
- 父 POM + `maas-common/*`、网关、Nacos、`runtime-hub` 单副本。
- `runtimes/common` SDK + WS 协议（含 JSON Schema）。
- 4 个第一批服务（COLA + Flyway + OpenAPI）。
- 3 个运行时：model-puller / model-trainer / model-server。
- PostgreSQL/MinIO/Redis/MQ/LabelStudio 集成。
- 核心测试：领域单测 + ArchUnit + Testcontainers + WS 契约 + pytest。
- Vue 3 基础页面 + OpenAPI 生成 TS。
- docker-compose 一键栈 + Helm 雏形 + 弱多租户 + 用量记录。

### 暂不做（YAGNI）
- 从零训练千亿基座；自研推理引擎/调度器/标注工具。
- 多模态、AutoML、联邦学习；GDPR/PII 自动脱敏引擎。
- 复杂计费/发票引擎（先记用量）。
- 强租户隔离（独立 DB/集群）、跨云联邦、模型市场、公网 SaaS、移动端、完整 BI。
- 服务网格（Istio）、CQRS 事件溯源全量、Saga/ProcessManager（先手工）。
- model-server 弹性伸缩、HTTP/gRPC 推理端点、trainer 接 MQ（均 M2）。
- 500B+ 分布式推理（M3 再评估）。
- 每层拆成独立 Maven 模块的重型 COLA（M2 视规模再拆）。
- hub 多副本连接路由（MVP 单副本，M2 再上注册表）。

---

# 六、需要拍板的问题（Decisions Needed）

> 拍板会议议程（30 分钟）。建议优先拍板**第一梯队**；**第二梯队**无异议则按推荐方案执行，有异议现场讨论。会后在下方「决议记录」填写结论并同步本文件。

## 第一梯队（必须先定）
| # | 问题 | 推荐方案 | 备选 | 决策者 | 结论 |
|---|---|---|---|---|---|
| 1 | 首个行业与场景 | 金融客服 / 法律文书 | 医疗 / 教育 | ______ | ______ |
| 2 | 首批服务数 | 4 服务（含 artifact-service） | 3 服务 | ______ | ______ |
| 3 | runtime-hub 归属 | MVP 放 training-orchestration | 提前建 scheduler-service | ______ | ______ |
| 4 | MQ 选型 | RocketMQ 5.x | Kafka | ______ | ______ |
| 8 | 首发基座模型 | Qwen2.5-7B/14B + Llama-3.1-8B | 其他 | ______ | ______ |
| 10 | GPU 型号与 CUDA | 生产：A100 80G / L20 / 4090 + CUDA 12.4；开发：Ryzen AI MAX+ 395（ROCm 实验级，不承诺 SLA） | 其他 | ______ | ______ |
| 15 | 团队与工期 | Java2 + Python2 + 前端1 + 算法1；M0 2 月 → M1 5 月 → M2 9 月 | 调整 | ______ | ______ |

## 第二梯队（可默认走推荐）
| # | 问题 | 推荐方案 | 是否有异议 | 结论 |
|---|---|---|---|---|
| 5 | 注册/配置中心 | Nacos | 是 / 否 | ______ |
| 6 | COLA 形态 | 单模块分包 | 是 / 否 | ______ |
| 7 | 租户方案 | M1 单租户 + 预留字段 | 是 / 否 | ______ |
| 9 | WS 协议 | AUTH 首帧 + 30s/90s 心跳 | 是 / 否 | ______ |
| 11 | 前端 UI | Element Plus | 是 / 否 | ______ |
| 12 | 交付形态 | 纯软件 + 可选一体机，支持离线 | 是 / 否 | ______ |
| 13 | 训练方式 | MVP 仅 LoRA/QLoRA | 是 / 否 | ______ |
| 14 | 仓库与 CI | Monorepo + GitHub Actions | 是 / 否 | ______ |

## 决议记录
- **第一梯队**：______（决策者）在 ______（日期）前给出最终决定。
- **第二梯队**：无异议则按推荐方案执行，有异议现场讨论。
- 会后更新本文件并标注决策结果。

### 硬件基线（已定）
- **MVP 开发 / POC**：AMD Ryzen AI MAX+ 395（128GB 统一内存），7B/14B 推理与 LoRA/QLoRA 验证；**ROCm 实验级支持，不承诺生产 SLA**。
- **生产私有化**：NVIDIA A100 80G / L20 / 4090（CUDA 12.4），M2 起作为交付基线。

> 前置建议优先拍板：**1、2、3、4、8、10、15**。

---

## 附：一句话总结
平台新增四个 Python 运行时组件（**model-puller / model-compressor / model-trainer / model-server**），MVP 阶段统一通过 **WebSocket 长连接**与 Java 后端通信，连接管理逻辑抽取为公共 SDK（`maas-runtime-sdk`）复用；Java 侧为状态唯一事实来源，运行时仅作算力执行器；产品化阶段再为 model-server 增加 HTTP/gRPC、为 model-trainer 接入 MQ。

---

# 附录：何时需要重画路线图

以下任一情况发生，需要重新评估三阶段划分：

1. 首个行业客户需求不是「微调模型」，而是「RAG + 知识库」 → **Knowledge Service 提前到 M1**
2. 客户行业为金融/政务/医疗 → **Guardrail / Audit / Model Card 提前到 M1 末**
3. 7B LoRA 效果不够，必须 70B 全参数微调 → **Scheduler / 多卡训练提前**
4. 销售承诺「多租户 SaaS」 → **私有化路线需分叉**
5. GPU 预算砍半 → **必须加推理缓存 / 小模型路由 / 成本门禁**
