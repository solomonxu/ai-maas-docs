# 贡献指南（Contributing）

[简体中文](./CONTRIBUTING.md) | [English](./CONTRIBUTING.en.md)

感谢参与 OpenMaas。项目处于开源收尾阶段，欢迎 Issue、文档改进和代码贡献。

## 沟通

- 问题与建议：提交 Issue（Bug / Feature / Docs）；
- 安全漏洞：见 [SECURITY.md](./SECURITY.md)，请勿公开披露；
- 代码仓库公开前，先通过 Issue 或 Discussion 沟通方案。

## 分支与提交

- `main`：稳定分支；
- `feat/*`、`fix/*`、`docs/*`：功能、修复和文档分支；
- 提交信息：`<type>(<scope>): <subject>`；
- 一个 PR 聚焦单一意图，描述包含动机、改动、影响和验证方式。

## 代码贡献

代码仓库公开后：

- Java：遵循 `adapter → app → domain ← infrastructure`，domain 零框架依赖；
- Python：运行时组件复用公共 SDK，保持协议一致；
- 前端：Vue 3 + TypeScript，优先复用自研组件；
- 测试：单元测试、架构守护、前端类型检查/构建、i18n 校验。

## 文档贡献

- 中文为主，英文版本保持同步；
- 图表优先使用 Markdown/Mermaid 或 SVG；
- 图片放在 `images/`，命名语义化；
- 不在公开文档中写入内部 IP、域名、凭据、Token、私钥或客户信息。

## 开发日志

- 每日由生成脚本从代码提交生成草稿；
- 维护者审阅中文和英文内容后提交；
- 英文版必须完整重写，不做机器词表替换；
- 敏感信息命中时中止发布。

## 许可

文档贡献按本仓 [Apache-2.0](./LICENSE) 许可发布；代码仓许可证以代码仓为准。
