# librechat-deploy

Production Docker Compose deployment of [LibreChat](https://github.com/danny-avila/LibreChat) for private use (2-4 users) on a VPS.

## Stack

| Service | Image | Purpose |
|---------|-------|---------|
| `api` | `ghcr.io/danny-avila/librechat:latest` | LibreChat backend + frontend |
| `mongodb` | `mongo:8.0` | Persistent chat / user storage |
| `meilisearch` | `getmeili/meilisearch:v1.35.1` | Full-text message search |
| `nginx` | `nginx:1.27-alpine` | TLS termination / reverse proxy |

RAG and the admin panel are intentionally omitted – they add complexity and resource usage that isn't worthwhile for a small private deployment.

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
- `DOMAIN_CLIENT` – your public domain, e.g. `https://chat.example.com`
- At least one AI provider API key (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, …)

### 2. TLS certificates

Place your TLS certificate and key in `nginx/certs/`:

```
nginx/certs/fullchain.pem
nginx/certs/privkey.pem
```

With Certbot (standalone, before nginx is running):

```bash
certbot certonly --standalone -d chat.example.com
cp /etc/letsencrypt/live/chat.example.com/fullchain.pem nginx/certs/fullchain.pem
cp /etc/letsencrypt/live/chat.example.com/privkey.pem   nginx/certs/privkey.pem
```

### 3. Update nginx domain

Edit `nginx/nginx.conf` and replace every occurrence of `your.domain.com` with your actual domain.

### 4. Configure LibreChat

Edit `librechat.yaml` to enable the AI endpoints you want.  
Full reference: <https://www.librechat.ai/docs/configuration/librechat_yaml>

### 5. Start

```bash
docker compose pull
docker compose up -d
```

Check logs:

```bash
docker compose logs -f api
```

## Updating LibreChat

```bash
docker compose pull
docker compose up -d --remove-orphans
```

## Security notes

- Registration is disabled by default (`ALLOW_REGISTRATION=false`). Create your first user via the LibreChat web UI on first run, then keep registration closed.
- The MongoDB instance is not exposed to the host network.
- Meilisearch is not exposed to the host network.
- Only ports 80 and 443 are published; all internal traffic stays on the Docker network.
- TLS 1.2+ only; strong cipher suite configured in `nginx/nginx.conf`.