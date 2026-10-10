# Quickstart

[English](./quickstart.en.md) | [简体中文](./quickstart.md)

> Prerequisites: Docker 20+ and Docker Compose v2. The first build takes a while (many images).

## 1. Get the code

```bash
# use your organization's repository URL (placeholder)
git clone <your-code-repo-url> ai-maas
cd ai-maas
```

## 2. Start with Docker Compose

```bash
cd deploy/docker-compose
cp .env.example .env          # adjust ports/secrets as needed
docker compose up -d --build
```

- Gateway entry point: `http://localhost:18080` (see `.env` port mappings).
- Ports and volumes: see `deploy/docker-compose/ports-mapping.md`.

## 3. First use (recommended order)

1. **Nodes**: ensure runtime components are online (puller/trainer/server, …).
2. **Base model**: register → pull (HuggingFace / ModelScope) → post-pull analysis & scan → variant ready.
3. **Dataset**: register/import → scan and create subsets.
4. **Training**: create a fine-tuning job (LoRA) → metrics/logs → artifact.
5. **Serving**: create an inference service from base + LoRA; call it via an OpenAI-compatible endpoint.
6. **Eval/Compress**: run evaluation gates and quantization as needed.

## 4. Stop / clean up

```bash
docker compose down            # stop (keep data volumes)
docker compose down -v         # stop and delete data (caution)
```

## 5. FAQ

- Slow first build: wait, or use an offline image pack (see [deployment](./deployment.en.md)).
- Port conflict: change the port in `.env` and restart.
- Pull failures: check network/mirror settings (source, mirror endpoint, token).
- More: see [deployment](./deployment.en.md) and the troubleshooting doc (incrementally added).

## Next

- Develop: [development](./development.en.md)
- Architecture: [architecture](./architecture.en.md)
