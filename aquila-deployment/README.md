# Aquila deployment package

Run Aquila using published container images, without installing the application source code or building images yourself.

**Start with [SETUP.md](SETUP.md).** It walks through registry login, environment configuration, image downloads, database migrations, startup, and verification. Commands are written for Windows PowerShell with Docker running Linux containers.

## Files to keep together

```text
aquila-deployment/
  README.md
  SETUP.md
  docker-compose.yml
  .env.example
  litellm_config.yaml
  custom-model-costs.json
  nginx/
    nginx.conf
  overlays/
    docker-compose.presidio.yml
```

Create your own `.env` from `.env.example` during setup. Do not distribute your populated `.env`. Keep the folder layout: Compose mounts the configuration files using these relative paths.

## Images used

| Service | Image | Purpose |
| --- | --- | --- |
| Gateway | `newaquilaacr.azurecr.io/aquila-gateway:1.0.0` | AI request processing |
| Web | `newaquilaacr.azurecr.io/aquila-web:1.0.0` | User interface and application APIs |
| LiteLLM migration | `newaquilaacr.azurecr.io/aquila-litellm-migrate:1.0.0` | LiteLLM database migrations |
| Aquila migration | Reuses the Web image | Aquila database migrations and initial administrator |
| PostgreSQL | `postgres:16-alpine` | Database |
| Redis | `redis:7-alpine` | Runtime cache |
| Nginx | `nginx:1.30.4-alpine` | Application entry point on port 4000 |

The Aquila images require authorized ACR access. PostgreSQL, Redis, and Nginx are pulled from public registries. The optional Presidio overlay adds the public Analyzer and Anonymizer images from `ghcr.io/data-privacy-stack`, pinned to version `2.2.362` and the digests in the overlay.

## Choose your setup

- **Standard:** Aquila without bundled Presidio services.
- **With Presidio:** Adds services for detecting and anonymizing personal information and exposes the managed Presidio integration in Aquila. You still configure and attach guardrails in the application.

Both use the same setup guide. Select your mode once in its instructions.

## Runtime and deployment scope

LiteLLM is installed dynamically in the Gateway container using `LITELLM_VERSION`. The first start needs outbound package/tool download access. Restarting the same container can reuse that installation; recreating it installs again because this package does not persist `/app/.venv` in a volume.

The supplied configuration runs HTTP at `http://localhost:4000` with bundled PostgreSQL and Redis. It is a starting point for evaluation, not a complete internet-facing production deployment. Production requires an agreed TLS, managed database/cache, backup, and access-control configuration. Setting `NGINX_TLS_DIR` alone does not enable HTTPS; this package does not include a TLS overlay.

Container health checks confirm service readiness. Complete setup by signing in, configuring a model provider, and testing an authorized model request.
