# 快速开始（Quickstart）

> 前置：Docker 20+ 与 Docker Compose v2。首次构建耗时较长（镜像较多）。

## 1. 获取代码

```bash
# 使用你所在组织的代码仓地址（示例占位）
git clone <your-code-repo-url> ai-maas
cd ai-maas
```

## 2. 一键启动（Docker Compose）

```bash
cd deploy/docker-compose
cp .env.example .env          # 按需修改端口/密钥
docker compose up -d --build
```

- 统一入口（网关）：`http://localhost:18080`（端口以 `.env` 映射为准）。
- 数据卷与端口映射见 `deploy/docker-compose/ports-mapping.md`。

## 3. 首次使用（建议顺序）

1. **计算节点**：确认运行时组件在线（拉取器/训练器/推理等）。
2. **基座模型**：登记 → 拉取（HuggingFace / ModelScope）→ 收尾分析与扫描 → 变体就绪。
3. **数据集**：登记/导入 → 扫描与子集。
4. **训练**：创建微调任务（LoRA）→ 查看指标/日志 → 产出制品。
5. **推理**：从基座 + LoRA 创建推理服务，使用 OpenAI 兼容端点调用。
6. **评测/压缩**：按需运行评测门禁与量化压缩。

## 4. 停止 / 清理

```bash
docker compose down            # 停止（保留数据卷）
docker compose down -v         # 停止并删除数据（慎用）
```

## 5. 常见问题

- 首次构建慢：耐心等待；或使用离线镜像包（见 [deployment.md](./deployment.md)）。
- 端口冲突：修改 `.env` 中对应端口后重启。
- 拉取失败：检查网络/镜像站配置（源站、镜像端点、Token）。
- 更多：见 [deployment.md](./deployment.md) 与故障排查文档（逐步补充）。

## 下一步

- 开发：见 [development.md](./development.md)
- 架构：见 [architecture.md](./architecture.md)
