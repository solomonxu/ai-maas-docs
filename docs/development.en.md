# Development

[English](./development.en.md) | [简体中文](./development.md)

> This is a **documentation-only** repository; code lives in the private main repository. Below is a summary of the engineering conventions.

## Stack

- Backend: Java 17 · Spring Boot 3.x · Spring Cloud Alibaba · COLA DDD
- Runtimes: Python 3.12 (CPU)
- Frontend: Vue 3 · TypeScript · Pinia · Vue Router (**in-house components, no third-party UI lib**)
- Storage: PostgreSQL · MinIO · Redis · RocketMQ · Nacos

## Conventions (highlights)

- **Layering**: `adapter → app → domain ← infrastructure`; `domain` has zero framework deps; enforced by ArchUnit.
- **Response body**: uniform `Result<T>`; REST prefix `/api/v1`; centralized error codes.
- **ID serialization**: boxed `Long` (snowflake IDs) are serialized as **strings** to avoid JS precision loss; frontend type `Id = string | number`.
- **Comments**: class/interface/field/config items need comments; DB column comments are persisted (`COMMENT ON`).
- **Multi-tenant**: `tenant_id` row-level + interceptor + RLS fallback; domain is tenant-agnostic.
- **i18n**: frontend locales must be fully in sync and validated (no missing/extra/empty entries).

## Local development (Java)

```bash
# compile a single module (with deps)
mvn -q clean compile -pl <module> -am

# full tests (some integration tests need Docker)
mvn -q test

# architecture guard
mvn -q test -Dtest="*ArchitectureTest" -DfailIfNoTests=false
```

## Local development (Python runtimes)

```bash
# e.g. install and test a runtime SDK/component
pip install -e 'runtimes/common[test]'
pip install -e 'runtimes/<component>[test]'
pytest runtimes/common runtimes/<component>
```

## Local development (frontend)

```bash
cd maas-web
npm install
npm run typecheck     # type check
npm run build         # build
npm run i18n:audit    # locale consistency audit
```

## Commit convention

`<type>(<scope>): <subject>`; type ∈ `feat|fix|docs|refactor|test|chore|perf|style`.
One commit/PR = one intent; include how you verified it.

## Documentation convention

- Chinese-first; keep terminology consistent; use Markdown/mermaid for diagrams; images go in `images/`.
- Self-check for **no sensitive info** before committing (see [SECURITY.md](../SECURITY.md)).
