# Contributing

[English](./CONTRIBUTING.en.md) | [简体中文](./CONTRIBUTING.md)

Thanks for your interest in AI MaaS. The project is in **M0 (under development)**; issues, discussions and documentation improvements are welcome.

## Communication

- Questions/suggestions: open an issue (Bug / Feature / Docs).
- Security issues: see [SECURITY.md](./SECURITY.en.md); do not disclose publicly.

## Branches & commits

- Branch model: `main` (stable, PR required) · `develop` (integration) · `feat/*`, `fix/*`, `docs/*`.
- Commit message: `<type>(<scope>): <subject>`, type ∈ `feat|fix|docs|refactor|test|chore|perf|style`.
  - e.g. `docs(readme): add quickstart`, `fix(puller): progress regression on resume`.
- One PR = one intent; describe motivation / changes / impact / verification.

## Documentation contributions

- Chinese-first; keep terminology consistent; prefer Markdown/mermaid over binary images.
- Put images in `images/` with semantic names (e.g., `architecture-overview.png`).
- Before committing, self-check for **no internal IPs/domains/credentials/customer info** (see red lines in `SECURITY.md`).

## Code contributions (code repo)

> Code is developed in the organization's private main repository; this repository is **documentation-only**. To contribute code, please start with an issue.

- Backend: Java 17 + Spring Boot 3.x + COLA DDD; dependency direction `adapter → app → domain ← infrastructure`, `domain` has zero framework deps.
- Runtimes: Python 3.12 (CPU).
- Frontend: Vue 3 + TypeScript; **no third-party UI library** — use the in-house `components/common/*`.
- Quality gates: unit tests + architecture guard (ArchUnit) + frontend type-check/build + i18n locale validation (see the code repo conventions).

## License

By contributing, you agree to release under this repo's [Apache-2.0](./LICENSE); the code repository follows its own license (MIT).
