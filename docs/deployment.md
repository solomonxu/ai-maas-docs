# 部署文档（Deployment）

[简体中文](./deployment.md) | [English](./deployment.en.md)

> OpenMaas 当前以 Docker Compose 为主，Helm/K8s 和离线包已具备基础；具体版本以代码仓库的 `deploy/` 为准。

## 部署形态

| 形态 | 说明 | 状态 |
|---|---|---|
| 单机 Compose | 管理面、运行时和依赖组件一体化 | 可用 |
| 分离部署 | 管理节点与算力节点分置 | 可用 |
| GPU 节点 | 训练/推理使用 ROCm 或 CUDA | 可用，需匹配节点与镜像 |
| Helm / K8s | 私有化集群部署 | 基础能力已具备 |
| 离线部署 | 无外网环境导入镜像与模型 | 已有离线包和脚本 |

## 单机 Compose

代码仓库公开后：

```bash
git clone <OpenMaas repository URL> ai-maas
cd ai-maas/deploy/docker-compose
cp .env.example .env
docker compose up -d --build
```

入口：

```text
Web:     http://localhost:5173
Gateway: http://localhost:18080
```

首次启动会构建或拉取较多镜像，耗时取决于网络和硬件。

## 分离部署

- 管理节点：执行 `deploy/docker-compose/scripts/deploy-management.sh`；
- 算力节点：复制 `deploy/docker-compose/runtime/.env.example` 为 `.env`，设置 `MANAGEMENT_HOST`；
- 执行 `deploy/docker-compose/scripts/deploy-runtime.sh`；
- 宿主机端口约定和映射见 `deploy/docker-compose/ports-mapping.md`。

## GPU 部署

- 训练和推理需要匹配节点架构、厂商和运行时；
- AMD 使用 ROCm 镜像，NVIDIA 使用 CUDA 镜像，Jetson 使用对应 arm64 镜像；
- 镜像、节点、模型和数据集需满足预检兼容性；
- 生产环境建议固定镜像 digest，并使用受控的镜像仓库。

## Helm / K8s

基础 Helm chart 位于代码仓库 `deploy/helm/maas/`。正式集群部署前需要准备：

- Kubernetes、Ingress、StorageClass；
- PostgreSQL、MinIO、Redis、Nacos（内置或外部）；
- GPU Operator / Device Plugin（如使用 GPU）；
- Secret、网络策略、TLS 和镜像拉取凭据。

具体 values 和验证命令以代码仓库 Helm README 为准。

## 离线部署

离线环境可使用预构建镜像包、离线模型包和本地 registry：

1. 在联网环境准备镜像和模型；
2. 通过 `docker save/load`、私有 registry 或离线介质传输；
3. 在目标环境登记模型和数据集；
4. 使用 Compose 或 Helm 启动。

## 环境变量

- 部署环境变量以代码仓库 `docs/ref/deploy-env.md` 为唯一事实源；
- `.env.example` 覆盖主要配置项；
- 不要提交真实密钥、Token 或生产口令；
- 生产环境应替换默认口令并启用 TLS、网络隔离和最小权限。

## 安全与运维建议

- 备份 PostgreSQL 与 MinIO；
- 监控 Postgres、MinIO、Redis、Nacos 和运行时组件；
- 为 GPU 任务设置租约、队列和容量上限；
- 为流水线配置预检、审批、预算和告警；
- 生产升级前先验证镜像、数据库迁移和回滚方案。

## 相关文档

- [快速开始](./quickstart.md)
- [架构](./architecture.md)
- [安全](../SECURITY.md)
