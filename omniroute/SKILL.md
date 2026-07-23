---
name: omniroute
description: >-
  Reference for OmniRoute — a self-hosted, OpenAI-compatible AI gateway running
  on this box at http://127.0.0.1:20128/v1 that aggregates many LLM providers
  behind one endpoint with smart auto-routing and cross-provider failover.
  Covers pointing Hermes' LLM backend at it, the auto/* routing combos and when
  to pin a concrete model, inspecting which model served a request, MCP + A2A
  access, and health/verification. Use when choosing or pinning an OmniRoute
  model, wiring Hermes to OmniRoute (it is interchangeable with freellmapi),
  tuning routing, or checking gateway health. Docker install/redeploy lives in
  references/install.md and is loaded on demand.
version: 2.0.0
metadata:
  hermes:
    category: llm-providers
    tags: [omniroute, llm, provider, openai, routing, mcp, a2a, gateway]
---

# OmniRoute — OpenAI-compatible LLM gateway

OmniRoute is a self-hosted gateway that exposes many LLM providers through a
single **OpenAI-compatible** `/v1` endpoint with automatic routing and
cross-provider failover. On this box it runs in Docker, **bound to localhost
only**, at `http://127.0.0.1:20128`.

It is **interchangeable with `freellmapi`** — both are OpenAI-compatible `/v1`
backends. Point Hermes at whichever you want as the primary LLM.

For install / redeploy / upgrade, load `references/install.md`.

## Endpoints

- Base URL: `http://127.0.0.1:20128/v1`
- Dashboard: `http://127.0.0.1:20128`
- Models: `GET /v1/models` · Chat: `POST /v1/chat/completions`
- On localhost the `/v1` API answers **without an API key** (it's bound to
  `127.0.0.1` only). A key is only needed for the MCP endpoint (see below).

## Point Hermes at OmniRoute

Edit the `model` block in `~/.hermes/config.yaml`:

```yaml
model:
  default: auto/best-chat            # an OmniRoute combo (see below)
  provider: custom
  base_url: http://127.0.0.1:20128/v1
  api_key: omniroute-local           # any value works on localhost
```

Then restart the gateway so it re-reads config:

```bash
sudo systemctl restart hermes-gateway.service
```

Note: OmniRoute has no bare `auto` model — `default` must be a concrete combo
like `auto/best-chat` or `auto/smart`. Swapping back to freellmapi is just
changing `base_url` to `http://localhost:3001/v1` and `api_key` to
`${FREELLMAPI_KEY}`.

## Model selection — use the `auto/*` combos

OmniRoute's routing lives in **`auto/*` combo models**: you pick an intent, the
gateway picks the best available provider and fails over on error. Query the
live catalog with `curl -s http://127.0.0.1:20128/v1/models | jq -r '.data[].id'`.

| Want | Use |
|---|---|
| General chat | `auto/best-chat` |
| Coding | `auto/best-coding` (free-only: `auto/coding:free`) |
| Hard reasoning | `auto/best-reasoning` (`auto/reasoning:pro`) |
| Vision / multimodal | `auto/best-vision` |
| Lowest latency | `auto/best-fast` |
| Free tiers only | `auto/best-free` |
| Cheapest | `auto/cheap` |
| Max capability | `auto/smart`, `auto/pro-*` |
| Pin a family | `auto/claude-opus`, `auto/claude-sonnet` |

**There is no `fusion` model here** — that was FreeLLMAPI. For higher accuracy
use `auto/best-*` or the `auto/pro-*` tier.

**Pin a concrete model** only when you need a specific one. Real IDs look like
`aug/claude-opus-4.6`, `aug/gemini-3.1-pro`, `aug/gpt-5.5-high`,
`oc/deepseek-v4-flash-free`, `oc/qwen3.6-plus-free`, `ddgw/gpt-5-mini` — not the
OpenRouter-style `vendor/model:free` names. Always confirm against `/v1/models`.

## Inspect what actually served a request

```bash
curl -s -D - http://127.0.0.1:20128/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"auto/best-fast","messages":[{"role":"user","content":"ping"}],"max_tokens":5}' \
  -o /dev/null | grep -iE "x-routed|x-omniroute|x-provider"
```

Response headers reveal routing/provider metadata for the version you're running.

## MCP & A2A

- **A2A** agent card: `GET /.well-known/agent.json` (live). Tasks post to `/a2a`.
- **MCP** stream: `http://127.0.0.1:20128/api/mcp/stream` — **requires an API
  key** (returns 401 without one). Create a key in the dashboard → Endpoints,
  then point an MCP client (Claude Code / Cursor) at that URL with the key.

## Health & verification

```bash
curl -s http://127.0.0.1:20128/api/monitoring/health | jq .status   # "healthy"
cd ~/omniroute && docker compose ps                                  # containers healthy
curl -s http://127.0.0.1:20128/v1/models | jq '.data | length'       # model count
```

## Related skills

- `freellmapi` — the other OpenAI-compatible backend; interchangeable with this.
- `references/install.md` — secure Docker install / redeploy (load on demand).
