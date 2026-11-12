# Development

[English](./development.en.md) | [简体中文](./development.md)

> OpenMaas has a code repository and this documentation repository. The code repository is coming soon; this repository holds product docs, architecture, protocol references, and development logs.

## Stack

- Backend: Java 17, Spring Boot 3.x, Spring Cloud Alibaba, COLA DDD;
- Runtimes: Python 3.12, PyTorch, Transformers, vLLM, PEFT;
- Frontend: Vue 3, TypeScript, Pinia, Vue Router, in-house components;
- Infrastructure: PostgreSQL, MinIO, Redis, Nacos;
- Deployment: Docker Compose, Helm/K8s, offline packages.

## Engineering Conventions

- `adapter → app → domain ← infrastructure`; the domain layer has no framework dependencies;
- unified `Result<T>`, `/api/v1` REST prefix, centralized error codes;
- boxed `Long` IDs are serialized as strings to avoid JavaScript precision loss;
- classes, interfaces, fields, and configuration items are documented; database comments are persisted;
- tenant isolation uses `tenant_id`, interceptors, and RLS fallback;
- frontend locales stay in sync; `npm run i18n:audit` must pass.

## Code Repository

After the repository is public:

```bash
git clone <OpenMaas repository URL> ai-maas
cd ai-maas
```

Java:

```bash
mvn -q clean compile -pl <module> -am
mvn -q test
mvn -q test -Dtest="*ArchitectureTest" -DfailIfNoTests=false
```

Python:

```bash
pip install -e 'runtimes/common[test]'
pip install -e 'runtimes/<component>[test]'
pytest runtimes/common runtimes/<component>
```

Frontend:

```bash
cd maas-web
npm install
npm run typecheck
npm run build
npm run i18n:audit
```

## Documentation Repository

This repository uses Markdown. Keep Chinese and English versions aligned, prefer Mermaid or SVG for diagrams, store images under `images/`, and review generated dev logs before committing. Scan for internal IPs, credentials, tokens, keys, and customer information before publishing.

## Commit Style

```text
<type>(<scope>): <subject>
```

`type`: `feat | fix | docs | refactor | test | chore | perf | style`. Keep each commit or PR focused and include verification details.

## Related

- [Architecture](./architecture.en.md)
- [Deployment](./deployment.en.md)
- [Contributing](../CONTRIBUTING.en.md)
- [Security](../SECURITY.en.md)
