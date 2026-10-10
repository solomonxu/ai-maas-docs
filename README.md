# AI MaaS — 垂直行业私有化微调平台

> 基座无关的中间层 · 私有化部署 · COLA DDD 微服务架构
> 状态：**🚧 M0（开发中，非生产就绪）** ｜ 许可：[Apache-2.0](./LICENSE)

[简体中文](./README.md) | [English](./README.en.md)

面向金融/法律等垂直行业，提供**私有化部署**的大模型**微调与推理**平台。

## 为什么是 AI MaaS

- **基座无关（Base-model agnostic）**：可替换 Qwen / Llama / DeepSeek 等任意开源模型，**只需改配置、不改业务代码**。
- **私有化部署（Private/Offline）**：数据不出域，支持内网/离线环境，面向合规敏感场景。

## 架构概览

```
            ┌──────────────────────────────┐
            │        Vue 3 前端 (SPA)        │
            └───────────────┬──────────────┘
                     HTTP / SSE / WS
                            │
                 ┌──────────▼───────────┐
                 │ Spring Cloud Gateway │
                 └──────────┬───────────┘
                    REST │            │ REST/WS
        ┌────────────────▼──┐   ┌─────▼──────────────────────────┐
        │  管理面微服务       │   │        runtime-hub              │
        │ (Java · COLA DDD)  │   │  （WS 统一控制面 / 注册表）      │
        │ model / dataset /  │   └─────┬──────────────────────────┘
        │ training / artifact│         │ WebSocket
        │ serving / eval /   │   ┌─────▼──────────────────────────┐
        │ compress / ...      │   │  Python 运行时                  │
        └─────────┬──────────┘   │  puller / trainer / server /    │
                  │              │  scanner / compressor / ...     │
                  │              └─────┬──────────────────────────┘
                  └──────────┬─────────┘
                 ┌───────────▼───────────────────────────────┐
                 │ PostgreSQL · MinIO · Redis · RocketMQ · Nacos │
                 └────────────────────────────────────────────┘
```

## 技术栈

| 层 | 技术 |
|---|---|
| 后端 | Java 17 · Spring Boot 3.x · Spring Cloud Alibaba · **COLA DDD** |
| 运行时 | **Python 3.12**（CPU）· PyTorch · Transformers · vLLM · PEFT |
| 前端 | Vue 3 · TypeScript · Pinia · Vue Router · **自研组件（无第三方 UI 库）** |
| 存储 | PostgreSQL（database-per-service）· MinIO |
| 消息/注册 | RocketMQ · Nacos · Redis |
| 部署 | Docker Compose · Helm / K8s（规划中） |
| 契约 | WebSocket 消息 Schema · ADR · 引擎/配方清单 |

## 核心特性

- **统一模型抽象**：tokenizer / chat_template / 量化格式 / 上下文长度 的统一描述。
- **WS 统一控制面**：Java ↔ Python 运行时单连接双向（下发任务 + 回收状态/进度/日志），心跳/离线检测/指数退避重连，注册表支持多副本路由。
- **端到端链路**：模型拉取 → 收尾扫描（安全/结构） → 微调（CPU/GPU） → 评测 → 量化/压缩 → 推理服务。
- **COLA DDD 分层**：`domain` 零框架依赖，`adapter → app → domain ← infrastructure`。
- **M1+ 治理（规划）**：Model Registry · Eval Gate · Guardrail · Inference Gateway · RAG/Knowledge · Prompt Registry。

## 快速开始

> 需要 Docker 与 Docker Compose。默认使用本地主机端口；实际端口以 `.env` 为准。

```bash
git clone <your-fork-or-mirror-url> ai-maas
cd ai-maas/deploy/docker-compose
cp .env.example .env
docker compose up -d --build
```

启动后访问网关入口：`http://localhost:18080`（以 `.env` 中 `MANAGEMENT_*`/端口映射为准）。

更多：见 [快速开始](./docs/quickstart.md) 与 [部署文档](./docs/deployment.md)。

## 文档导航

| 文档 | 说明 |
|---|---|
| [docs/overview.md](./docs/overview.md) | 项目概述与定位 |
| [docs/architecture.md](./docs/architecture.md) | 架构详解（分层/控制面/数据） |
| [docs/quickstart.md](./docs/quickstart.md) | 快速开始 |
| [docs/development.md](./docs/development.md) | 开发指南与规范 |
| [docs/deployment.md](./docs/deployment.md) | 部署（Compose / GPU / 离线） |
| [docs/dev-log/](./docs/dev-log/) | 每日开发日志 |
| [PLAN.md](./PLAN.md) | 总体规划与路线图 |
| [docs/decisions/](./docs/decisions/) | 架构决策记录（ADR，逐步补充） |

## 路线图

| 阶段 | 目标 | 周期 |
|---|---|---|
| **M0** 验证期 | 端到端打通最小链路（开发中） | 0–2 月 |
| M1 MVP | 产品化最小闭环，可交付试点 | 2–5 月 |
| M2 产品化 | 多客户可复制，补齐治理层 | 5–9 月 |
| M3 复制期 | 模板市场 + 数据飞轮 | 9–15 月 |

## 贡献

见 [CONTRIBUTING.md](./CONTRIBUTING.md)。安全问题请遵循 [SECURITY.md](./SECURITY.md)（勿公开披露）。

## 许可证与免责声明

- 本**文档仓**采用 [Apache-2.0](./LICENSE)；代码仓遵循其自身许可（MIT）。
- 项目处于 **M0 开发中**，按 **“AS IS”** 提供，**不承诺生产可用性**；使用风险自负。
- 集成第三方**基座模型**时，请自行评估并遵守相应模型许可证与合规要求。
- 文中第三方名称/商标仅用于说明与兼容性描述，不构成背书。
