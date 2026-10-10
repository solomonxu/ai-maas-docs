# 开发指南（Development）

> 本仓为**纯文档仓**；代码在私有主仓开发。以下为通用工程规范摘要。

## 技术栈

- 后端：Java 17 · Spring Boot 3.x · Spring Cloud Alibaba · COLA DDD
- 运行时：Python 3.12（CPU）
- 前端：Vue 3 · TypeScript · Pinia · Vue Router（**自研组件，不引入第三方 UI 库**）
- 存储：PostgreSQL · MinIO · Redis · RocketMQ · Nacos

## 工程约定（要点）

- **分层**：`adapter → app → domain ← infrastructure`；`domain` 零框架依赖，架构守护（ArchUnit）。
- **响应体**：统一 `Result<T>`；REST 前缀 `/api/v1`；错误码集中定义。
- **ID 序列化**：后端 boxed `Long`（雪花 ID）序列化为**字符串**，规避 JS 精度丢失；前端 `Id = string | number`。
- **注释**：类/接口/字段/配置项需中文注释；数据库字段注释落库（`COMMENT ON`）。
- **多租户**：`tenant_id` 行级 + 拦截器 + RLS 兜底；domain 不感知租户。
- **i18n**：前端多语种需全量同步并校验（无缺失/多余/空）。

## 本地开发（Java）

```bash
# 单模块编译（含依赖）
mvn -q clean compile -pl <module> -am

# 全量测试（部分集成测试需 Docker）
mvn -q test

# 架构守护
mvn -q test -Dtest="*ArchitectureTest" -DfailIfNoTests=false
```

## 本地开发（Python 运行时）

```bash
# 例：安装并测试某运行时 SDK/组件
pip install -e 'runtimes/common[test]'
pip install -e 'runtimes/<component>[test]'
pytest runtimes/common runtimes/<component>
```

## 本地开发（前端）

```bash
cd maas-web
npm install
npm run typecheck     # 类型检查
npm run build         # 构建
npm run i18n:audit    # 多语种一致性校验
```

## 提交规范

`<type>(<scope>): <subject>`；type ∈ `feat|fix|docs|refactor|test|chore|perf|style`。
一个提交/PR 聚焦单一意图，附验证方式。

## 文档规范

- 中文为主，术语统一；图表用 Markdown/mermaid，图片入 `images/`。
- 提交前自检**无敏感信息**（见 [SECURITY.md](../SECURITY.md)）。
