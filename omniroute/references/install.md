# OmniRoute — secure Docker install (localhost-only)

Install/redeploy OmniRoute with **all ports bound to `127.0.0.1`** so the
dashboard (which holds provider API keys) and API are never exposed to the
LAN/WAN. This mirrors the working deployment in `~/omniroute`.

> Never bind these ports to `0.0.0.0`. Keep the `127.0.0.1:` prefix on every
> published port.

## 1. Project directory

```bash
mkdir -p ~/omniroute && cd ~/omniroute
```

## 2. docker-compose.yml

```yaml
services:
  redis:
    image: docker.io/library/redis:7-alpine
    container_name: omniroute-redis
    restart: unless-stopped
    ports:
      - "127.0.0.1:${REDIS_PORT:-6379}:6379"
    volumes:
      - redis-data:/data
    command: redis-server --save 60 1 --loglevel warning
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  omniroute:
    image: diegosouzapw/omniroute:latest
    container_name: omniroute
    restart: unless-stopped
    stop_grace_period: 40s
    env_file: .env
    environment:
      - DATA_DIR=/app/data
      - PORT=${PORT:-20128}
      - DASHBOARD_PORT=${DASHBOARD_PORT:-20128}
      - API_PORT=${API_PORT:-20129}
      - API_HOST=${API_HOST:-0.0.0.0}
      - LIVE_WS_PORT=${LIVE_WS_PORT:-20132}
      - LIVE_WS_HOST=${LIVE_WS_HOST:-0.0.0.0}
      - LIVE_WS_ALLOWED_ORIGINS=http://localhost:20128,http://127.0.0.1:20128
      - REDIS_URL=${REDIS_URL:-redis://redis:6379}
    ports:
      - "127.0.0.1:${DASHBOARD_PORT:-20128}:${DASHBOARD_PORT:-20128}"
      - "127.0.0.1:${API_PORT:-20129}:${API_PORT:-20129}"
      - "127.0.0.1:${LIVE_WS_PORT:-20132}:${LIVE_WS_PORT:-20132}"
    volumes:
      - ./data:/app/data
    healthcheck:
      test: ["CMD", "node", "healthcheck.mjs"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s

volumes:
  redis-data:
    name: omniroute-redis-data
```

`API_HOST`/`LIVE_WS_HOST` are `0.0.0.0` **inside the container** — that's fine;
the localhost-only guarantee comes from the `127.0.0.1:` published-port prefixes.

## 3. .env.example → .env

```bash
# ── REQUIRED SECRETS ──────────────────────────────────────────────────
JWT_SECRET=            # openssl rand -base64 48
API_KEY_SECRET=        # openssl rand -hex 32
INITIAL_PASSWORD=CHANGEME   # change before first login

# ── PORTS ─────────────────────────────────────────────────────────────
PORT=20128
DASHBOARD_PORT=20128
API_PORT=20129
LIVE_WS_PORT=20132

# ── REDIS ─────────────────────────────────────────────────────────────
REDIS_URL=redis://redis:6379
```

Generate secrets and a strong password:

```bash
cp .env.example .env
JWT=$(openssl rand -base64 48) && API_KEY=$(openssl rand -hex 32) && \
  sed -i "s|^JWT_SECRET=.*|JWT_SECRET=$JWT|" .env && \
  sed -i "s|^API_KEY_SECRET=.*|API_KEY_SECRET=$API_KEY|" .env
# Then edit .env and set a strong INITIAL_PASSWORD
```

## 4. Start

```bash
docker compose pull
docker compose up -d
```

## 5. Verify (health + localhost-only)

```bash
docker compose ps                                                    # both healthy
curl -s http://127.0.0.1:20128/api/monitoring/health | jq .status    # "healthy"
ss -tlnp | grep -E "20128|20129|20132|6379"                          # all 127.0.0.1:*
```

Every listener must be `127.0.0.1:*` — never `0.0.0.0:*` or a LAN IP.

## 6. First-run setup

1. Open `http://127.0.0.1:20128`, log in as `admin` with your `INITIAL_PASSWORD`, change it.
2. **Providers** → connect at least one provider (API key).
3. (Optional) **Endpoints** → create an API key — only needed for the MCP
   endpoint; the `/v1` API is open on localhost.

## Update / redeploy

```bash
cd ~/omniroute && docker compose pull && docker compose up -d
```

## Security notes

- Keep `.env` out of version control and backups — it holds real secrets.
- Only the `127.0.0.1:` port prefixes keep this private; don't remove them.
- If exposure is suspected: regenerate `JWT_SECRET`/`API_KEY_SECRET`, reset the
  password, and recreate dashboard API keys.
