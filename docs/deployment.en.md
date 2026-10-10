# Deployment

[English](./deployment.en.md) | [简体中文](./deployment.md)

> Currently focused on Docker Compose; Helm/K8s/offline are planned (incrementally added).

## Deployment modes

| Mode | Description | Status |
|---|---|---|
| Single-node Compose | Control plane + runtimes + dependencies in one stack | Available (M0) |
| Split deployment | Separate management / compute nodes | Available (M0) |
| GPU nodes | GPU (ROCm/CUDA) for training/inference | Being improved |
| Offline | Air-gapped: import images and models via offline packs | Planned |
| Helm / K8s | Cluster & elasticity | Planned |

## Single-node Compose

```bash
cd deploy/docker-compose
cp .env.example .env
docker compose up -d --build
```

- Entry: gateway `http://localhost:18080` (see `.env`).
- Ports and data locations: see `ports-mapping.md`.
- Dependencies: PostgreSQL / MinIO / Nacos / Redis / RocketMQ.

## Split deployment (management / compute)

- Management node: `bash deploy/docker-compose/scripts/deploy-management.sh`
- Compute node: copy `deploy/docker-compose/runtime/.env.example` to `.env`, set `MANAGEMENT_HOST`, then
  `bash deploy/docker-compose/scripts/deploy-runtime.sh`
- Host ports are uniformly +10000 (example convention); see `ports-mapping.md`.

## GPU deployment

- Use the GPU override file (e.g., `docker-compose.gpu.yml`) and the corresponding base images.
- Training/inference engine images must match the node `arch/vendor` (see engine manifest & multi-arch build docs).

## Environment variables

- **Single source of truth**: the deploy-env reference (code repo `docs/ref/deploy-env.md`).
- `.env.example` covers all referenced variables; never commit real secrets.

## Offline deployment (planned)

- Pre-pull base/runtime images, then `docker save/load` or import via a private registry.
- Offline import of models/datasets (transfer packs) → register/scan.
- See the upcoming offline deployment manual.

## Security & operations

- Rotate default passwords/secrets; enable TLS/network isolation; run with least privilege.
- Back up PostgreSQL and MinIO; plan upgrade/rollback and object lifecycle.
- In production, wire the diagnosis ports/health checks into monitoring and alerting.
