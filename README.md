# librechat-deploy

Production Docker Compose deployment of [LibreChat](https://github.com/danny-avila/LibreChat) for private use (2-4 users) on a VPS.

## Stack

| Service | Image | Purpose |
|---------|-------|---------|
| `api` | `ghcr.io/danny-avila/librechat:latest` | LibreChat backend + frontend |
| `mongodb` | `mongo:8.0` | Persistent chat / user storage |
| `meilisearch` | `getmeili/meilisearch:v1.35.1` | Full-text message search |

RAG and the admin panel are intentionally omitted – they add complexity and resource usage that isn't worthwhile for a small private deployment.

TLS termination and public routing are handled by your own ingress proxy (e.g. Traefik, Caddy, or nginx running on the host). The API is bound to `127.0.0.1:3080` so only the host and processes on it can reach it.

## Setup

### 1. Clone and configure

```bash
git clone https://github.com/phips-git/librechat-deploy.git
cd librechat-deploy
cp .env.example .env
```

Edit `.env` and set at minimum:
- `JWT_SECRET` and `JWT_REFRESH_SECRET` – generate with `openssl rand -hex 32`
- `MEILI_MASTER_KEY` – generate with `openssl rand -hex 32`
- `DOMAIN_CLIENT` – the URL your ingress proxy exposes LibreChat on, e.g. `https://chat.example.com`

### 2. Configure AI providers

AI provider API keys and endpoint settings are managed in `librechat.yaml`, not in `.env`. Uncomment and fill in the relevant blocks for the providers you want to use. Each key is read from an environment variable so you can store the actual secret in `.env`:

```yaml
# librechat.yaml
endpoints:
  openAI:
    apiKey: "${OPENAI_API_KEY}"
  anthropic:
    apiKey: "${ANTHROPIC_API_KEY}"
```

Then add the corresponding variable to `.env`:

```
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
```

Full endpoint reference: <https://www.librechat.ai/docs/configuration/librechat_yaml/ai_endpoints>

### 3. Start

```bash
docker compose pull
docker compose up -d
```

Check logs:

```bash
docker compose logs -f api
```

### 4. Point your ingress proxy to port 3080

The API listens on `127.0.0.1:3080`. Configure your existing reverse proxy (Traefik, Caddy, nginx, …) to forward traffic for your domain to that address.  
Make sure to pass `Upgrade` / `Connection` headers for WebSocket support (required for chat streaming).

## Adding users

Registration is closed by default (`ALLOW_REGISTRATION=false`). The first user to register becomes an admin, so the workflow for onboarding additional users is:

1. Temporarily enable registration in `.env`: `ALLOW_REGISTRATION=true`
2. Restart the stack: `docker compose up -d`
3. Have the new user navigate to your LibreChat URL and create their account.
4. Once done, set `ALLOW_REGISTRATION=false` again and restart.

This keeps the instance effectively invite-only without requiring the admin panel.

## File uploads (without RAG)

Users can attach files to any conversation. Without the RAG pipeline, uploaded files are:

- **Stored locally** in the `librechat-uploads` Docker volume (persisted across restarts).
- **Sent directly to the model** for context — ideal for image analysis (vision), PDF reading, and code review with models that support file/vision input (e.g. GPT-4o, Claude 3.x).
- **Not indexed or searchable** — without RAG there is no vector database, so files cannot be semantically searched across conversations.

If you later want document search/Q&A over uploaded files, you can add the RAG API and pgvector services from the upstream `deploy-compose.yml`.

## Updating LibreChat

```bash
docker compose pull
docker compose up -d --remove-orphans
```

## Security notes

- Registration is disabled by default (`ALLOW_REGISTRATION=false`). Re-enable it only briefly when onboarding a new user.
- The API port (3080) is bound to `127.0.0.1` only — not accessible from outside the host without going through your ingress proxy.
- MongoDB and Meilisearch are not exposed to the host network at all; they are internal Docker services.