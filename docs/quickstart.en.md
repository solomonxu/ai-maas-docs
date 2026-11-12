# Quickstart

[English](./quickstart.en.md) | [简体中文](./quickstart.md)

> Prerequisites: Docker 20+ and Docker Compose v2. The first build can take time. Until the code repository is public, the repository URL below is a placeholder.

## 1. Get the Code

```bash
git clone <OpenMaas repository URL> ai-maas
cd ai-maas
```

## 2. Start a Single-Node Environment

```bash
cd deploy/docker-compose
cp .env.example .env
docker compose up -d --build
```

Endpoints:

```text
Web:     http://localhost:5173
Gateway: http://localhost:18080
```

Actual ports follow `.env` and the Compose mapping.

## 3. First Steps

1. Confirm that compute nodes and runtime components are online;
2. Register and pull a base model;
3. Import or register a dataset;
4. Create a training job and inspect metrics and logs;
5. Register the resulting artifact;
6. Create an inference service and call the OpenAI-compatible endpoint;
7. Run evaluation, compression, or a pipeline when needed.

## 4. Use the Pipeline

In the web UI, open **Pipelines**:

```text
choose a goal → choose resources → choose a template
→ preflight → create a run → start
```

You can also save a pipeline definition first and reference it from a trigger for Cron or Webhook automation.

## 5. Stop and Clean Up

```bash
docker compose down       # stop, keep data
docker compose down -v    # stop and delete data; use with care
```

## 6. Troubleshooting

- Slow first startup: many images are required; wait or use an offline package.
- Port conflicts: update `.env` and restart.
- Model pull failure: check network, mirror settings, and tokens.
- GPU unavailable: check node architecture, drivers, images, and device mappings.
- Preflight failure: fix the reported resource or compatibility issue before starting.

## More

- [Deployment](./deployment.en.md)
- [Architecture](./architecture.en.md)
- [Development](./development.en.md)
