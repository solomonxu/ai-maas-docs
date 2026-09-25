# AI MaaS — 垂直行业私有化微调平台

> 基座无关的中间层 · 私有化部署 · COLA DDD 微服务架构

## 项目定位

面向金融/法律等垂直行业，提供**私有化部署的大模型微调与推理平台**。  
核心差异化：**基座无关**——可替换 Qwen / Llama / DeepSeek 等任意开源模型，不改业务代码。

## 架构概览
[Vue 3 前端]

↓ HTTP/WS

[Spring Cloud Gateway]

↓

[base-model] [dataset] [training-orchestration] [artifact]  ← Java 微服务 (COLA DDD)

↑                              ↓ WS 控制面

[runtime-hub] ← WebSocket → [model-puller / model-trainer / model-server]  ← Python 运行时

↓

[MinIO / PostgreSQL / Redis / RocketMQ]

纯文本
## 技术栈

| 层 | 技术 |
|---|---|
| 后端 | Java 17 + Spring Boot 3.x + Spring Cloud Alibaba + COLA DDD |
| 运行时 | Python 3.11 + PyTorch + vLLM + Transformers |
| 前端 | Vue 3 + TypeScript + Element Plus |
| 数据库 | PostgreSQL（database-per-service） |
| 对象存储 | MinIO |
| 消息队列 | RocketMQ |
| 注册配置 | Nacos |
| 部署 | Docker Compose / Helm / K8s |

## 核心设计亮点

### 1. 基座无关中间层
- 统一模型描述（tokenizer / chat_template / 量化格式 / 上下文长度）
- 推理后端适配层（vLLM / TGI / Ollama），对外统一 OpenAI 兼容协议
- 替换基座只需改配置，不修改业务服务

### 2. WebSocket 统一控制面
- Java ↔ Python 运行时统一通过 WS 长连接通信
- 单连接双向：下发任务 + 回收状态/进度/日志
- 心跳 30s / 离线 90s 检测 / 指数退避重连
- 连接注册表（Redis）解决多副本路由

### 3. COLA DDD 分层
每个微服务按 adapter / app / domain / infrastructure 四层分包：
- **domain 层**：纯业务规则，零框架依赖
- **app 层**：应用编排，事务边界
- **adapter 层**：Controller / WS Endpoint / MQ Listener
- **infrastructure 层**：MyBatis-Plus / Redis / MinIO 实现

### 4. 生产化治理（M2 目标）
- Model Registry + Eval Gate + Guardrail Service
- Inference Gateway（路由/限流/缓存/审计）
- Knowledge Service（RAG）+ Prompt Registry
- Experiment Tracking + Data Lineage

## 三阶段路线

| 阶段 | 目标 | 周期 |
|---|---|---|
| M0 验证期 | 端到端手工打通最小链路 | 0–2 月 |
| M1 MVP | 产品化最小闭环，可交付试点客户 | 2–5 月 |
| M2 产品化 | 多客户可复制，补齐治理层 | 5–9 月 |
| M3 复制期 | 模板市场 + 数据飞轮 | 9–15 月 |

## 快速开始（Docker Compose）
bash

git clone https://gitee.com/solomonxu/ai-maas.git

cd ai-maas/deploy/docker-compose

docker compose --profile default up -d

访问 http://localhost:8080

> 完整文档见 [ai-maas-docs](https://github.com/solomonxu/ai-maas-docs)

## 项目状态

🚧 开发中（M0 阶段） — 欢迎 Issue 与讨论

## 许可

本项目仅供学习参考，商用需自行评估基座模型 License。