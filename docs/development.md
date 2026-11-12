# 开发指南（Development）

[简体中文](./development.md) | [English](./development.en.md)

> OpenMaas 由代码仓和文档仓组成。代码仓即将公开；本文档仓负责产品文档、架构、协议和开发日志。

## 技术栈

- 后端：Java 17、Spring Boot 3.x、Spring Cloud Alibaba、COLA DDD；
- 运行时：Python 3.12、PyTorch、Transformers、vLLM、PEFT；
- 前端：Vue 3、TypeScript、Pinia、Vue Router、自研组件；
- 数据与基础设施：PostgreSQL、MinIO、Redis、Nacos；
- 部署：Docker Compose、Helm / K8s、离线包。

## 工程约定

- 分层：`adapter → app → domain ← infrastructure`；`domain` 零框架依赖；
- 统一响应体 `Result<T>`，REST 前缀 `/api/v1`，错误码集中定义；
- boxed `Long` ID 序列化为字符串，避免前端精度丢失；
- 类、接口、字段和配置项使用中文注释；数据库字段注释落库；
- 多租户：`tenant_id` 行级 + 拦截器 + RLS 兜底；
- 前端多语种同步，`npm run i18n:audit` 必须通过。

## 代码仓库

代码仓库公开后，按以下方式开发：

```bash
git clone <OpenMaas repository URL> ai-maas
cd ai-maas
```

Java：

```bash
mvn -q clean compile -pl <module> -am
mvn -q test
mvn -q test -Dtest="*ArchitectureTest" -DfailIfNoTests=false
```

Python：

```bash
pip install -e 'runtimes/common[test]'
pip install -e 'runtimes/<component>[test]'
pytest runtimes/common runtimes/<component>
```

前端：

```bash
cd maas-web
npm install
npm run typecheck
npm run build
npm run i18n:audit
```

## 文档仓库

本仓使用 Markdown 维护。约定：

- 中文为主，英文版本保持同步；
- 架构图优先使用 Mermaid 或 SVG；图片放在 `images/`；
- 开发日志由每日脚本生成草稿，维护者审阅后提交；
- 提交前扫描敏感信息：内部 IP、凭据、Token、私钥和客户信息。

## 提交规范

```text
<type>(<scope>): <subject>
```

`type`：`feat | fix | docs | refactor | test | chore | perf | style`。一个提交/PR 聚焦单一意图，并在描述中包含验证方式。

## 文档贡献

- 修正术语和翻译时保持中英文一一对应；
- 不在公开文档中写入内部主机名、IP、凭据和个人敏感信息；
- 新功能先更新架构/部署/开发文档，再补充开发日志；
- 涉及外部模型、数据集和镜像时，注明许可证来源。

## 相关文档

- [架构](./architecture.md)
- [部署](./deployment.md)
- [贡献指南](../CONTRIBUTING.md)
- [安全](../SECURITY.md)
