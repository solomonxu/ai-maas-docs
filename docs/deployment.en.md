# Deployment

[English](./deployment.en.md) | [简体中文](./deployment.md)

> OpenMaas primarily uses Docker Compose today. Helm/K8s and offline packages have a working foundation; use the code repository's `deploy/` directory as the source of truth.

## Deployment Modes

| Mode | Description | Status |
|---|---|---|
| Single-node Compose | Control plane, runtimes, and dependencies together | Available |
| Split deployment | Management node and compute nodes separated | Available |
| GPU nodes | ROCm or CUDA training/inference | Available with matching node images |
| Helm / K8s | Private cluster deployment | Foundation available |
| Offline | Import images and models without internet access | Offline packages and scripts exist |

## Single-Node Compose

After the code repository is public:

```bash
git clone <OpenMaas repository URL> ai-maas
cd ai-maas/deploy/docker-compose
cp .env.example .env
docker compose up -d --build
```

Endpoints:

```text
Web:     http://localhost:5173
Gateway: http://localhost:18080
```

The first startup may take time because many images are built or pulled.

## Split Deployment

- Management node: `deploy/docker-compose/scripts/deploy-management.sh`;
- Compute node: copy `deploy/docker-compose/runtime/.env.example` to `.env`, set `MANAGEMENT_HOST`, then run `deploy/docker-compose/scripts/deploy-runtime.sh`;
- See `deploy/docker-compose/ports-mapping.md` for port conventions.

## GPU Deployment

Training and inference require matching node architecture, vendor, and runtime. AMD uses ROCm images, NVIDIA uses CUDA images, and Jetson uses the corresponding arm64 images. Pin image digests and use a controlled registry in production.

## Helm / K8s

The Helm chart lives in `deploy/helm/maas/`. Before a production cluster deployment, prepare Kubernetes, Ingress, StorageClass, PostgreSQL, MinIO, Redis, Nacos, GPU device plugins (if needed), Secrets, network policies, TLS, and image-pull credentials. See the Helm README in the code repository for values and validation.

## Offline Deployment

Prepare images and models in a connected environment, transfer them through `docker save/load`, a private registry, or offline media, then register models and datasets in the target environment. Start with Compose or Helm.

## Environment Variables

- `docs/ref/deploy-env.md` in the code repository is the single source of truth;
- `.env.example` covers the main settings;
- never commit real secrets, tokens, or production passwords;
- replace default credentials and enable TLS, network isolation, and least privilege in production.

## Operations

Back up PostgreSQL and MinIO, monitor infrastructure and runtimes, configure GPU leases and capacity limits, and validate image, database migration, and rollback steps before upgrades.

## Related

- [Quickstart](./quickstart.en.md)
- [Architecture](./architecture.en.md)
- [Security](../SECURITY.en.md)
