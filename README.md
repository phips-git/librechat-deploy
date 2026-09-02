# librechat-deploy

Minimal, private LibreChat deployment via Docker Compose for 2-4 users on a VPS.

## Stack

| Service | Image | Purpose |
|---------|-------|---------|
| `api` | `ghcr.io/danny-avila/librechat:latest` | LibreChat backend + frontend |
| `mongodb` | `mongo:8.0` | Persistent chat / user storage |
| `meilisearch` | `getmeili/meilisearch:v1.35.1` | Full-text message search |

TLS termination is handled by your existing ingress proxy. The API binds to `127.0.0.1:3080`.

## Setup

### 1. Configure

```bash
cp .env.example .env
```

Fill in `.env`:
- `JWT_SECRET` and `JWT_REFRESH_SECRET` — `openssl rand -hex 32`
- `MEILI_MASTER_KEY` — `openssl rand -hex 32`
- `DOMAIN_CLIENT` — the URL your ingress proxy exposes, e.g. `https://chat.example.com`

### 2. Configure AI providers

API keys and endpoint settings live in `librechat.yaml`. Uncomment the relevant provider block and add the secret to `.env`:

```yaml
# librechat.yaml
endpoints:
  openAI:
    apiKey: "${OPENAI_API_KEY}"
```

```env
# .env
OPENAI_API_KEY=sk-...
```

Full reference: <https://www.librechat.ai/docs/configuration/librechat_yaml/ai_endpoints>

### 3. Start

```bash
docker compose pull
docker compose up -d
```

### 4. Ingress proxy

Forward your domain to `127.0.0.1:3080`. Pass `Upgrade` / `Connection` headers for WebSocket streaming.

## Managing users

Registration is closed by default. Use the built-in CLI scripts inside the running `api` container — no flags to toggle, no restarts needed.

**Create a user directly** (interactive; generates a password if left blank):
```bash
docker compose exec api npm run create-user
```

**Generate a one-time invite link** (printed to console; no email setup required):
```bash
docker compose exec api npm run invite-user -- user@example.com
```

The first registered user is automatically promoted to admin.

## File uploads

Without RAG, uploaded files are stored in the `librechat-uploads` volume and sent directly to the model as context. This supports image analysis, PDF reading, and code review for vision-capable models (e.g. GPT-4o, Claude 3+). Files are not indexed or searchable across conversations — that requires adding the RAG/pgvector services from the upstream `deploy-compose.yml`.

## Updates

```bash
docker compose pull && docker compose up -d --remove-orphans
```

## Security notes

- Only `127.0.0.1:3080` is published; MongoDB and Meilisearch are internal.
- Registration stays closed; users are added via CLI only.
