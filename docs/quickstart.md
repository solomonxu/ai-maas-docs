# 快速开始（Quickstart）

[简体中文](./quickstart.md) | [English](./quickstart.en.md)

> 前置：Docker 20+、Docker Compose v2。首次构建耗时较长。代码仓库公开前，以下命令中的仓库地址为占位符。

## 1. 获取代码

```bash
git clone <OpenMaas repository URL> ai-maas
cd ai-maas
```

## 2. 启动单机环境

```bash
cd deploy/docker-compose
cp .env.example .env
docker compose up -d --build
```

入口：

```text
Web:     http://localhost:5173
Gateway: http://localhost:18080
```

实际端口以 `.env` 和 compose 映射为准。

## 3. 首次使用建议顺序

1. 确认计算节点和运行时组件在线；
2. 登记基座模型并拉取；
3. 导入或登记数据集；
4. 创建训练任务，查看指标和日志；
5. 注册模型制品；
6. 创建推理服务，使用 OpenAI 兼容端点调用；
7. 按需运行评测、压缩和流水线。

## 4. 使用流水线

在 Web 中进入“训推流水线”：

```text
选择目标 → 选择资源 → 选择模板 → 预检 → 创建运行 → 启动
```

也可以先保存“流水线定义”，再在触发器页面引用，用于 Cron 或 Webhook 自动化。

## 5. 停止与清理

```bash
docker compose down       # 停止，保留数据
docker compose down -v    # 停止并删除数据，慎用
```

## 6. 常见问题

- 首次启动慢：镜像较多，耐心等待或使用离线包；
- 端口冲突：修改 `.env` 后重启；
- 模型拉取失败：检查网络、镜像站配置和 Token；
- GPU 不可用：检查节点架构、驱动、镜像和设备映射；
- 预检失败：先根据预检项修复资源或兼容性问题，再启动。

## 更多

- [部署](./deployment.md)
- [架构](./architecture.md)
- [开发](./development.md)
