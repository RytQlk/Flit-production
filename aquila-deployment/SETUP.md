# Set up Aquila from published images

Follow these steps in order. Run each command separately and stop if it fails. These instructions use Windows PowerShell. You do not need Git, a local Python installation, application source code, or Dockerfiles.

## 1. Before you begin

Install Docker Desktop with Linux containers and Azure CLI. Start Docker Desktop and wait for the engine to be ready. Your Azure account must have permission to pull the Aquila images from `newaquilaacr`; ask the deployment owner to grant access before continuing.

Check your tools:

```powershell
docker version
docker compose version
az version
```

`docker version` should show both Client and Server. You need access to ACR, Docker Hub, and package/tool download services for Gateway startup. Presidio also needs access to GitHub Container Registry. Port 4000 must be available. Another deployment using the container name `nginx` will conflict with this package.

Extract the complete deployment package, preserving its subfolders. Open PowerShell in that folder. For example, if you extracted it to `C:\Aquila\aquila-deployment`:

```powershell
Set-Location C:\Aquila\aquila-deployment
Get-ChildItem -Force
```

Use your actual extraction path. You should see the files listed in [README.md](README.md).

## 2. Sign in to the image registry

```powershell
az login
```

Select the account and tenant supplied by your administrator. If multiple subscriptions are available, select the appropriate one:

```powershell
az account set --subscription "YOUR_SUBSCRIPTION_ID"
```

Then authenticate Docker to ACR:

```powershell
az acr login --name newaquilaacr
```

Expected: `Login Succeeded`. This grants no new permissions; your account must already be authorized. Customers do not create a registry, build images, or push images.

## 3. Create your environment file

Run this only if `.env` does not already exist. It preserves an existing installation's settings:

```powershell
if (-not (Test-Path .\.env)) { Copy-Item .\.env.example .\.env }
notepad .\.env
```

`.env.example` is a reusable template with blank secrets. `.env` is your private configuration. Save it as exactly `.env`, not `.env.txt`. Use plain `NAME=value` lines and plain URLs, without Markdown formatting.

### Generate secrets without installing Python

Pull the published Web image:

```powershell
docker pull newaquilaacr.azurecr.io/aquila-web:1.0.0
```

This command starts a temporary Python process from that image to generate values; it does not start the application or connect to its database:

```powershell
docker run --rm --entrypoint python newaquilaacr.azurecr.io/aquila-web:1.0.0 -c "import secrets; import base64; print('LITELLM_MASTER_KEY=sk-' + secrets.token_hex(32)); print('LITELLM_SALT_KEY=' + secrets.token_hex(32)); print('POSTGRES_PASSWORD=' + secrets.token_hex(24)); print('AQUILA_COOKIE_ENCRYPTION_KEY=' + base64.urlsafe_b64encode(secrets.token_bytes(32)).decode()); print('AQUILA_BOOTSTRAP_ADMIN_PASSWORD=' + secrets.token_hex(16))"
```

For a **new installation**, replace the five matching blank lines in `.env` with the generated lines. Store them securely and do not share the command output. Do not append duplicate keys. On an existing installation, preserve the existing database password and encryption keys; replacing them here does not rotate stored database credentials and can prevent encrypted data from being read.

| Setting | What to use |
| --- | --- |
| `LITELLM_MASTER_KEY` | Generated value beginning with `sk-` |
| `LITELLM_SALT_KEY` | Generated separate encryption secret; keep stable |
| `POSTGRES_PASSWORD` | Generated database password; hexadecimal avoids URL-special characters in the current connection string |
| `AQUILA_COOKIE_ENCRYPTION_KEY` | Generated URL-safe base64 key |
| `AQUILA_BOOTSTRAP_ADMIN_PASSWORD` | Generated initial login password, at least 12 characters |
| `AQUILA_BOOTSTRAP_ADMIN_USERNAME` | Keep `admin` or choose your initial username |

For the bundled services, keep these settings:

```dotenv
POSTGRES_HOST=db
POSTGRES_USER=admin
POSTGRES_DB=aquila_db
POSTGRES_VOLUME_NAME=aquila_pgdata
REDIS_URL=
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_SCHEME=redis
REDIS_VOLUME_NAME=aquila_redisdata
LITELLM_VERSION=1.98.0
LITELLM_PACKAGE_SHA256=
SSO_COOKIE_SECURE=false
RAG_ENABLED=false
```

Use volume names belonging to this installation. Reusing an older volume also reuses its database and credentials. Do not change volume names on an existing installation unless intentionally switching data stores.

Keep `REDIS_PASSWORD` empty for the bundled evaluation configuration: its supplied health check does not authenticate. A secured or managed Redis deployment needs a separately reviewed configuration. Leave optional classifier, RAG, and custom-cost overrides at their template defaults unless you are configuring those features.

Keep the release's LiteLLM version and migration settings. `LITELLM_PACKAGE_SHA256` may be filled with a wheel hash supplied by the release owner; blank means no explicit wheel hash verification. Changing the LiteLLM version independently can make it incompatible with the migration image.

### Optional Microsoft SSO

For local username/password login, leave the three Microsoft identity fields blank. The UI will show that Microsoft SSO is not configured; this is expected.

To enable SSO, obtain an Entra application registration and enter your values:

```dotenv
MICROSOFT_TENANT_ID=<your-tenant-id>
MICROSOFT_CLIENT_ID=<your-application-client-id>
MICROSOFT_CLIENT_SECRET=<your-client-secret-value>
MICROSOFT_REDIRECT_URI=http://localhost:4000/auth/sso/callback
MICROSOFT_POST_LOGIN_REDIRECT_URL=http://localhost:4000/chat
```

Replace the placeholders. Register the callback URL in Entra as a Web redirect URI. Normal users land on chat; administrator access depends on the user's role. Use `AQUILA_ENTRA_ADMIN_IDENTITIES` for approved administrator bindings in the format `<tenant-id>:<object-id>`. A redirect URL alone does not grant administrator permissions.

## 4. Choose standard or Presidio mode

Choose **one** block below. The PowerShell array holds the Compose options so all later commands use the same environment file and mode. Run your chosen block again if you open a new terminal.

**Standard deployment:**

```powershell
$composeArgs = @('-f', 'docker-compose.yml', '--env-file', '.env')
```

**Deployment with Presidio:**

```powershell
$composeArgs = @('-f', 'docker-compose.yml', '-f', 'overlays/docker-compose.presidio.yml', '--env-file', '.env')
```

Presidio adds the Analyzer and Anonymizer containers. The overlay enables the managed integration in Web and makes Gateway wait for both containers to become healthy. Keep its supplied image tag and digest defaults. No extra public host ports are needed.

## 5. Validate and download the images

```powershell
docker compose @composeArgs --profile migration config --quiet
```

No output and an exit code of zero mean Compose accepted the configuration. This does not check that secrets or database credentials are correct.

Confirm the image names:

```powershell
docker compose @composeArgs --profile migration config --images
```

You should see the three Aquila ACR images, PostgreSQL, Redis, and Nginx. Presidio mode also lists its two public images. The Web image is shared by Web and Aquila migrations. Migration services require the `migration` profile to appear.

Pull everything for the selected mode:

```powershell
docker compose @composeArgs --profile migration pull
```

Wait for this to finish successfully. No `--build` is needed: this package uses prebuilt images.

## 6. Start the database and cache

```powershell
docker compose @composeArgs up -d --no-build db redis
docker compose @composeArgs ps db redis
```

Wait until both show `healthy`. If either fails, inspect its logs before proceeding:

```powershell
docker compose @composeArgs logs --tail 100 db redis
```

## 7. Run the database migrations

Run LiteLLM migrations first:

```powershell
docker compose @composeArgs --profile migration run --rm litellm-migrate
$LASTEXITCODE
```

Continue only when the exit code is `0`. A message saying there are no pending migrations is also a success.

Then run Aquila migrations:

```powershell
docker compose @composeArgs --profile migration run --rm aquila-migrate
$LASTEXITCODE
```

Again, expect `0`. This job creates the initial administrator when initializing an empty Aquila schema. Changing the bootstrap password in `.env` later does not reset an existing account. These jobs finish and their temporary containers are removed; they are not expected to remain running in `docker compose ps`.

## 8. Start Aquila

The same command works in either selected mode. In Presidio mode, Compose also starts the required Presidio services:

```powershell
docker compose @composeArgs up -d --no-build web gateway nginx
docker compose @composeArgs ps
```

Gateway's first startup can take several minutes while it installs LiteLLM and any required dependencies and generates the Prisma client. Follow progress:

```powershell
docker compose @composeArgs logs -f --tail 100 gateway web
```

Press Ctrl+C to stop following logs; the containers continue running. Wait for Web and Gateway to become healthy. Nginx should be running, and Presidio mode should also show both Presidio services healthy.

If startup reported an unhealthy dependency, inspect its logs. After resolving the issue, rerun the startup command above.

## 9. Verify and sign in

Check each application service directly:

```powershell
docker compose @composeArgs exec web python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:4005/health/ready', timeout=10).status)"
docker compose @composeArgs exec gateway /app/.venv/bin/python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:4010/health/readiness', timeout=10).status)"
```

Expect `200` from each. Check the browser entry point:

```powershell
(Invoke-WebRequest http://localhost:4000 -UseBasicParsing).StatusCode
```

Open **http://localhost:4000** and sign in with the bootstrap username and password. Complete any required password change. Configure a model provider and test a request with an authorized user/key; a healthy container alone does not prove model inference is configured.

For Presidio, also check:

```powershell
docker compose @composeArgs ps presidio-analyzer presidio-anonymizer
```

In Aquila's guardrail administration, configure and attach a Presidio guardrail to the intended scope. Test with synthetic personal information and verify the configured detection/anonymization behavior. Starting the services alone does not apply a guardrail to every request.

To inspect the installed LiteLLM version:

```powershell
docker compose @composeArgs exec gateway /app/.venv/bin/python -c "from importlib.metadata import version; print(version('litellm'))"
```

Expect `1.98.0` for this release. The installation lives inside the container. A normal restart can reuse it; container recreation or `down` followed by `up` requires installation again.

## 10. Everyday commands

Run these from the same folder after selecting `$composeArgs` as in step 4.

Start an existing installation:

```powershell
docker compose @composeArgs up -d --no-build
```

Stop services while keeping containers and data:

```powershell
docker compose @composeArgs stop
```

After editing and saving `.env`, recreate affected application services:

```powershell
docker compose @composeArgs up -d --no-build --force-recreate web gateway nginx
```

A plain `restart` does not load changed environment values. Recreating Gateway triggers runtime installation again. Wait for health checks before using the application.

Remove containers and the Compose network while preserving named data volumes:

```powershell
docker compose @composeArgs down
```

Do not add `-v` or `--volumes` unless you intend to delete stored data. Do not rerun migrations for an ordinary restart. For upgrades, obtain the release's deployment files and migration instructions, back up the database, pull the specified image versions, and run the required migrations. Restoring an older image does not undo database migrations.

## Troubleshooting

| Problem | Next step |
| --- | --- |
| Docker cannot connect to the engine | Start Docker Desktop and confirm Linux containers are enabled. |
| ACR says authentication required or access denied | Repeat `az acr login --name newaquilaacr`; ask the owner to confirm your registry pull permission and tenant. |
| Migration image is missing from image listings | Include `--profile migration`. |
| PostgreSQL reports authentication failure | Verify your volume and original credentials. Editing `.env` does not change the password inside an initialized database. Do not delete data to troubleshoot. |
| Web or Gateway is unhealthy | Run `docker compose @composeArgs logs --tail 100 web gateway`. Resolve the reported failure before retrying startup. |
| Browser shows 502 | Check Web/Gateway health and Nginx logs. After both are healthy, `docker compose @composeArgs restart nginx` can refresh upstream addresses after container replacement. |
| SSO remains unconfigured | Save the correct deployment folder's `.env`; check all three identity fields are filled, then recreate Web/Gateway. Do not share full resolved configuration because it contains secrets. |
| Local login reports identity provisioning unavailable | Inspect Web and Gateway logs; readiness alone does not confirm user provisioning succeeded. |
| Presidio is missing | Re-select the Presidio `$composeArgs` block and rerun startup. |
| Port 4000 or container name nginx is already used | Identify the existing deployment and stop it if appropriate. Do not run the old and new deployment concurrently with the same port/name. |

For support, provide image versions, `docker compose @composeArgs ps`, and relevant logs with secrets removed. Never send your populated `.env`.
