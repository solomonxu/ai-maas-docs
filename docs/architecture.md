# 架构详解（Architecture）

[简体中文](./architecture.md) | [English](./architecture.en.md)

> M0 阶段架构，可能随迭代调整。

## 总览

```
Frontend (Vue 3)  ──HTTP/SSE/WS──►  Gateway  ──►  管理面微服务 (Java/COLA DDD)
                                        │
                                        └──WS──► runtime-hub ──WS──► Python 运行时
                                                     │
                               PostgreSQL · MinIO · Redis · RocketMQ · Nacos
```

## 分层（每个 Java 微服务）

- **adapter**：REST Controller、WS Endpoint、MQ Listener（入站/出站边界）。
- **app**：应用编排、事务边界、跨聚合协调（CMP/Command）。
- **domain**：实体/值对象/领域服务/状态机，**零框架依赖**。
- **infrastructure**：MyBatis-Plus、Redis、MinIO、外部客户端等实现。

依赖方向：`adapter → app → domain ← infrastructure`。由架构守护（ArchUnit）保证。

## 微服务概览（管理面）

| 服务 | 职责（示例） |
|---|---|
| `model`（base-model） | 基座模型登记/拉取/收尾扫描/变体资产 |
| `dataset` | 数据集登记/导入/扫描/子集/血缘 |
| `training`（training-orchestration） | 训练任务编排与生命周期 |
| `artifact` | 模型制品/晋级/血缘 |
| `serving` | 推理服务与端点 |
| `eval` | 评测任务与门禁 |
| `compress` | 量化/压缩任务 |
| `gateway` | 统一入口、鉴权、路由、限流 |
| `scheduler`（GPU） | GPU 资源可见性/分配（M1+） |
| `node` / `admin` / `assistant` / `image` | 节点编排、平台管理、在线助手、镜像服务 |

> 具体清单以代码仓 `maas-services/` 与规划为准。

## WebSocket 统一控制面

- Java ↔ Python 运行时**单连接双向**：下发任务指令 + 回传状态/进度/日志/结果。
- 连接注册表（Redis）支持**多副本**路由；心跳/离线检测/指数退避重连。
- 消息契约见 [../docs/protocol/](../docs/protocol/)（逐步补充）与代码仓 §协议 schema。

## 数据与存储

- **PostgreSQL**：database/schema-per-service；多租户 `tenant_id` 行级 + RLS 兜底（M1）。
- **MinIO**：模型/数据集/制品对象存储，预签名直链预览。
- **Redis**：会话/注册表/事件总线（SSE 跨副本广播）。
- **RocketMQ**：异步任务与解耦（逐步接入）。
- **Nacos**：注册与配置。

## 关键设计原则

- **契约先行**：WS/作业/镜像等消息有 JSON Schema 约束。
- **状态机驱动**：任务生命周期显式建模（如 ADR-0009 训练状态机）。
- **可观测**：结构化日志 + 指标（M1 完善）。
- **安全默认**：最小权限、密钥外部化、私有化优先。

## ADR

架构决策记录见 [decisions/](./decisions/)（逐步补充）。
