# 贡献指南（Contributing）

[简体中文](./CONTRIBUTING.md) | [English](./CONTRIBUTING.en.md)

感谢参与 AI MaaS。当前处于 **M0（开发中）**，欢迎 Issue、讨论与文档改进。

## 沟通

- 问题/建议：提交 Issue（Bug / Feature / Docs）。
- 安全漏洞：见 [SECURITY.md](./SECURITY.md)，请勿公开披露。

## 分支与提交

- 分支模型：`main`（稳定，需 PR）· `develop`（合流）· `feat/*`、`fix/*`、`docs/*`。
- 提交信息：`<type>(<scope>): <subject>`，type ∈ `feat|fix|docs|refactor|test|chore|perf|style`。
  - 例：`docs(readme): 补充快速开始`、`fix(puller): 断点续传进度回退`。
- 一个 PR 聚焦一件事；标题清晰，描述含动机/改动/影响/验证。

## 文档贡献

- 中文为主，术语保持一致；图表优先 Markdown/mermaid，避免二进制大图。
- 图片放 `images/`，命名语义化（`architecture-overview.png`）。
- 提交前自检：**无内部 IP/域名/凭据/客户信息**（见 `SECURITY.md` 红线）。

## 代码贡献（代码仓）

> 代码在本组织的私有主仓开发；本仓为**纯文档仓**。如需贡献代码，请先通过 Issue 沟通。

- 后端：Java 17 + Spring Boot 3.x + COLA DDD；分层依赖 `adapter → app → domain ← infrastructure`，`domain` 零框架依赖。
- 运行时：Python 3.12（CPU）。
- 前端：Vue 3 + TypeScript；**不引入第三方 UI 库**，使用自研 `components/common/*`。
- 质量门禁：单元测试 + 架构守护（ArchUnit）+ 前端类型检查/构建 + i18n 全语种校验（详见代码仓规范）。

## 许可

贡献即表示同意以本仓 [Apache-2.0](./LICENSE) 许可发布；代码仓遵循其自身许可（MIT）。
