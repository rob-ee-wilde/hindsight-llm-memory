# Hindsight - Dokploy self-host (`deploy` branch)

This is an **orphan branch**: it contains only the deployment config for running
[Hindsight](https://github.com/vectorize-io/hindsight) on a [Dokploy](https://dokploy.com)
server. It shares no history with `main`, so syncing the fork from upstream never
touches or conflicts with these files.

It deploys the **upstream-published image** `ghcr.io/vectorize-io/hindsight` as-is
(no custom build): one container running the REST/MCP API on `8888` and the Control
Plane UI on `9999`, backed by an external PostgreSQL with pgvector.

## Files
- `docker-compose.yaml` - the Dokploy Compose stack (`db` + `hindsight`).
- `.env.example` - variables to set in the Dokploy Environment tab.
- `README.md` - this runbook.

## Host requirements
- Dokploy server with **>= 8 GB RAM, 2+ vCPU, ~15 GB free disk** (the image bundles
  local embedding + reranker models). First retain/recall warms models into RAM.
- DNS A-records pointing at the Dokploy host for the subdomains you will use.

## Deploy (Compose from Git)
1. **DNS**: point `memory.<your-domain>` (UI) and, if needed, `memory-api.<your-domain>` (API) at the Dokploy host.
2. Dokploy -> your Project -> **Create Service -> Compose**.
3. **Provider = GitHub** (or Git). Repo `rob-ee-wilde/hindsight-llm-memory`,
   **Branch `deploy`**, Compose path `docker-compose.yaml`.
   - If the repo is private, connect Dokploy's GitHub App or add a deploy key.
     This branch holds no secrets, so it is safe to keep public if you prefer.
4. **Environment** tab: set the variables from `.env.example` (mask secrets).
   Minimum: `HINDSIGHT_API_LLM_API_KEY`, `HINDSIGHT_DB_PASSWORD`, `HINDSIGHT_CP_ACCESS_KEY`.
5. **Deploy.** Watch logs for DB migrations and `Hindsight is running!`.
6. **Domains** tab -> add:
   - UI: host `memory.<domain>` -> service `hindsight`, port **9999**, HTTPS + Let's Encrypt.
   - API (optional): host `memory-api.<domain>` -> service `hindsight`, port **8888**, HTTPS.
   Redeploy after adding/changing domains.

## Environment variables
| Variable | Required | Default | Notes |
|---|---|---|---|
| `HINDSIGHT_API_LLM_API_KEY` | yes | - | LLM key (secret) |
| `HINDSIGHT_DB_PASSWORD` | yes | - | Postgres password (secret) |
| `HINDSIGHT_CP_ACCESS_KEY` | recommended | - | Gates the UI login (secret) |
| `HINDSIGHT_API_LLM_PROVIDER` | no | `openai` | `anthropic`, `gemini`, `groq`, `ollama`, `atlas`, ... |
| `HINDSIGHT_API_LLM_MODEL` | no | `gpt-4o-mini` | |
| `HINDSIGHT_API_MCP_AUTH_TOKEN` | no | - | Bearer required on `/mcp` |
| `HINDSIGHT_DB_USER` / `HINDSIGHT_DB_NAME` | no | `hindsight_user` / `hindsight_db` | |
| `HINDSIGHT_VERSION` | no | `latest` | Pin to a released tag, e.g. `0.8.3` |

## Security (read this)
The Hindsight **REST API has no built-in authentication** in the OSS build - anyone
who can reach `:8888` can read/write every memory bank. Pick one:
- **Do not expose the API** (recommended for personal/team use): add only the UI
  domain. The Control Plane reaches the API in-container via `localhost:8888`.
- **Expose the API with protection**: add a Traefik **Basic Auth** middleware to the
  `memory-api` domain in Dokploy, and/or set `HINDSIGHT_API_MCP_AUTH_TOKEN` if you
  only consume `/mcp`.
Always set `HINDSIGHT_CP_ACCESS_KEY`.

## Continuous deployment
- **Config changes (this branch):** enable the Compose service's **auto-deploy
  webhook** in Dokploy and register it as a GitHub webhook scoped to `deploy`.
  Editing `docker-compose.yaml` / env and pushing then redeploys automatically.
  (Domain changes still need a manual redeploy.)
- **New Hindsight release:** you deploy a published image, so a new upstream release
  does not auto-redeploy. Either pin `HINDSIGHT_VERSION` and bump it deliberately
  (recommended), or enable Dokploy's registry/tag watch (or a scheduled redeploy)
  if you track `latest`.

## Verify
- `curl -I https://memory.<domain>` -> CP login / dashboard.
- In the container: `curl localhost:8888/health` -> `200` (or via the API domain).
- Smoke test: `pip install hindsight-client`, then `retain` + `recall` against the API URL.

## Updating this config
`deploy` is an orphan branch - edit these files on `deploy` directly
(`git switch deploy`) and push. Never merge `deploy` into `main` or vice-versa.

## Alternative: embedded database (quickest, no external Postgres)
Drop the `db` service and `HINDSIGHT_API_DATABASE_URL`, and give `hindsight` a named
volume for the embedded engine:

    volumes:
      - hindsight_pg0:/home/hindsight/.pg0
    # ...
    volumes:
      hindsight_pg0:

Embedded mode relies on graceful shutdown (handled by the image) plus the named
volume. Use Dokploy Volume Backups on that volume.
