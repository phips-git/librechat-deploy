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
- At least one AI provider API key (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, …)

### 2. Configure LibreChat

Edit `librechat.yaml` to enable the AI endpoints you want.  
Full reference: <https://www.librechat.ai/docs/configuration/librechat_yaml>

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

## Updating LibreChat

```bash
docker compose pull
docker compose up -d --remove-orphans
```

## Security notes

- Registration is disabled by default (`ALLOW_REGISTRATION=false`). Create your first user via the LibreChat web UI on first run, then keep registration closed.
- The API port (3080) is bound to `127.0.0.1` only — not accessible from outside the host without going through your ingress proxy.
- MongoDB and Meilisearch are not exposed to the host network at all; they are internal Docker services.