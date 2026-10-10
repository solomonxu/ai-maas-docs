# 部署文档（Deployment）

[简体中文](./deployment.md) | [English](./deployment.en.md)

> 当前聚焦 Docker Compose；Helm/K8s/离线为规划中（逐步补充）。

## 部署形态

| 形态 | 说明 | 状态 |
|---|---|---|
| 单机 Compose | 管理面 + 运行时 + 依赖组件一体化 | 可用（M0） |
| 分离部署 | 管理节点 / 算力节点分置 | 可用（M0） |
| GPU 节点 | 训练/推理使用 GPU（ROCm/CUDA） | 逐步完善 |
| 离线部署 | 无外网环境，镜像与模型经离线包导入 | 规划中 |
| Helm / K8s | 集群化与弹性 | 规划中 |

## 单机 Compose

```bash
cd deploy/docker-compose
cp .env.example .env
docker compose up -d --build
```

- 入口：网关 `http://localhost:18080`（以 `.env` 为准）。
- 端口映射与数据落盘位置见 `ports-mapping.md`。
- 依赖组件：PostgreSQL / MinIO / Nacos / Redis / RocketMQ。

## 分离部署（管理面 / 算力节点）

- 管理节点：`bash deploy/docker-compose/scripts/deploy-management.sh`
- 算力节点：复制 `deploy/docker-compose/runtime/.env.example` 为 `.env`，设置 `MANAGEMENT_HOST` 后
  `bash deploy/docker-compose/scripts/deploy-runtime.sh`
- 宿主端口统一 +10000（示例约定），详见 `ports-mapping.md`。

## GPU 部署

- 使用 GPU 覆盖文件（如 `docker-compose.gpu.yml`）与对应基础镜像。
- 训练/推理引擎镜像与节点 `arch/vendor` 需匹配（见引擎清单与多架构构建文档）。

## 环境变量

- **唯一事实源**：部署环境变量规范文档（代码仓 `docs/ref/deploy-env.md`）。
- `.env.example` 覆盖全部被引用的变量；请勿提交真实密钥。

## 离线部署（规划）

- 预拉取基础镜像与运行时镜像，`docker save/load` 或私有 registry 导入；
- 模型/数据集离线导入（摆渡包）→ 登记/扫描；
- 详见后续离线部署手册。

## 安全与运维建议

- 更换默认口令与密钥；启用 TLS/网络隔离；最小权限运行。
- 备份 PostgreSQL 与 MinIO；规划升级/回滚与对象生命周期。
- 生产环境请在后端可达的诊断端口与健康检查上接入监控告警。
