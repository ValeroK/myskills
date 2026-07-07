# FreeLLMAPI — full reference

Load this when SKILL.md isn't enough: exhaustive provider/model catalog,
per-endpoint request/response shapes, headers, rate-limit detail, environment
variables, and ToS notes. Source: <https://github.com/tashfeenahmed/freellmapi>
and <https://freellmapi.co/models.html>.

## Contents

- Endpoints
- Authentication
- Providers & models (catalog)
- Model selection values
- OpenAI-format examples (chat, tools, vision, embeddings)
- Anthropic-format surface (secondary — Hermes uses OpenAI)
- Headers
- Rate limiting & failover
- Environment variables
- Install / run
- Limitations & ToS notes

> Counts below (161 models, 18 providers, ~1.7B tokens/month) are the project's
> claims for a fully-configured install. Hermes's actual reachable set = whatever
> upstream keys are enabled and healthy. **Always trust `GET /v1/models` over
> this document.**

---

## Endpoints (all under `/v1`)

| Method + path | Purpose |
|---|---|
| `POST /v1/chat/completions` | OpenAI-compatible chat — **Hermes's primary endpoint** |
| `POST /v1/embeddings` | Vector embeddings, family-based routing |
| `POST /v1/images/generations` | Image generation |
| `POST /v1/audio/speech` | Text-to-speech |
| `POST /v1/completions` | Legacy prompt/suffix (editor autocomplete) |
| `GET  /v1/models` | List available models (content-negotiated shape) |
| `POST /v1/messages` | Anthropic Messages API (for non-OpenAI clients) |
| `POST /v1/messages/count_tokens` | Token counting, Anthropic format |
| `POST /v1/responses` | Codex CLI wire format with tool calls |

---

## Authentication

- **Unified Bearer (primary):** `Authorization: Bearer freellmapi-<key>`
  (OpenAI SDK: set `OPENAI_API_KEY=freellmapi-<key>` and
  `OPENAI_BASE_URL=http://localhost:3001/v1`).
- **Anthropic header (for `/v1/messages` only):** `x-api-key: freellmapi-<key>`
  or the Bearer token.

The unified key is generated at setup, lives on the dashboard **Keys** page
header, and replaces all upstream provider credentials.

---

## Providers & models (catalog)

~161 free models across 18 providers when fully configured:

| Provider | Representative models |
|---|---|
| Google | Gemini 2.5 Flash, Gemini 2.5 Pro, 3.x previews |
| Groq | Llama 3.3 70B, Llama 4, GPT-OSS, Qwen3 |
| Cerebras | Qwen3 235B |
| Mistral | Large 3, Medium 3.5, Codestral, Devstral |
| OpenRouter | 21 free-tier models |
| GitHub Models | GPT-4.1, GPT-4o |
| Cloudflare | Kimi K2, GLM-4.7, GPT-OSS, Granite 4 |
| Cohere | Command R+, Command-A |
| Z.ai (Zhipu) | GLM-4.5, GLM-4.7 Flash |
| NVIDIA | NIM (40 RPM free) |
| HuggingFace | DeepSeek V4, Kimi K2.6, Qwen3 |
| Ollama Cloud | GLM-4.7, Kimi K2, gpt-oss, Qwen3 |
| Kilo Gateway, Pollinations, LLM7, OVH AI Endpoints, AI Horde, custom | various |

**Capability ceiling:** the strongest reachable models are ~Llama 3.3 70B,
GLM-4.5, Qwen 3 Coder, Gemini 2.5 Pro. No Opus/GPT-4-class frontier models.

Browse the live catalog: <https://freellmapi.co/models.html>.

### Model selection values
- `"auto"` — router picks highest-priority healthy model under rate limits.
- Specific IDs — e.g. `"gemini-2.5-flash"`, `"llama-3.3-70b-versatile"`,
  `"qwen-3-coder"`.
- Claude family aliases (`/v1/messages` only): `"claude-opus"`,
  `"claude-sonnet-4-5"`, `"claude-haiku"`, `"claude-default"` — routing labels
  mapped to pinned upstreams on Keys → Anthropic, **not** real Anthropic models.

---

## OpenAI-format examples

### Chat (Python, OpenAI SDK) — Hermes's path
```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:3001/v1",
    api_key="freellmapi-your-unified-key",
)
resp = client.chat.completions.create(
    model="auto",
    messages=[{"role": "user", "content": "Summarise the fall of Rome."}],
)
print(resp.choices[0].message.content)
print("Routed via:", resp.headers.get("x-routed-via"))
```

### Chat (curl)
```bash
curl http://localhost:3001/v1/chat/completions \
  -H "Authorization: Bearer freellmapi-your-unified-key" \
  -H "Content-Type: application/json" \
  -d '{"model": "auto", "messages": [{"role": "user", "content": "hi"}]}'
```

### Tool / function calling
```json
{
  "model": "auto",
  "messages": [ ... ],
  "tools": [{
    "type": "function",
    "function": {
      "name": "function_name",
      "description": "...",
      "parameters": {"type": "object", "properties": { ... }, "required": [ ... ]}
    }
  }],
  "tool_choice": "required"
}
```
The response contains a `tool_calls` array; reply with
`{"role": "tool", "tool_call_id": "...", "content": "..."}` to continue.
Multi-step tool loops work across all provider ecosystems.

### Vision (image input)
```json
{
  "role": "user",
  "content": [
    {"type": "text", "text": "What's in this image?"},
    {"type": "image_url", "image_url": {"url": "data:image/png;base64,<...>"}}
  ]
}
```
The router restricts to vision-capable models and returns `422` with
`code: "no_vision_model"` if none are enabled. Streaming with images is supported.

### Embeddings
```bash
curl http://localhost:3001/v1/embeddings \
  -H "Authorization: Bearer freellmapi-..." \
  -H "Content-Type: application/json" \
  -d '{"model": "auto", "input": "hello world"}'
```
**Failover never crosses families** (incompatible vectors). Families include
`gemini-embedding-001` (default, 3072 dims), `text-embedding-3-large`,
`bge-m3` (1024), and others. Default set on dashboard **Models → Embeddings**.

---

## Anthropic-format surface (secondary)

Hermes uses the OpenAI path above. This section is for Anthropic-format clients
(e.g. Claude Code, Anthropic SDKs) pointed at the same proxy:

```bash
export ANTHROPIC_BASE_URL=http://localhost:3001
export ANTHROPIC_AUTH_TOKEN=freellmapi-your-unified-key   # NOT ANTHROPIC_API_KEY
```
`/v1/messages` speaks Anthropic's native format. ⚠️ Use `ANTHROPIC_AUTH_TOKEN`;
setting `ANTHROPIC_API_KEY` makes such clients treat it as a conflicting
credential and refuse to start. `GET /v1/models` returns Anthropic shape when the
client sends an `anthropic-version` header, OpenAI shape otherwise.

---

## Headers

**Request (optional):**
- `X-Session-Id: <id>` — sticky session; pin conversation to one model for 30 min.
- `anthropic-version: <date>` — makes `GET /v1/models` return Anthropic shape.

**Response (always present on chat):**
- `X-Routed-Via: <platform>/<model>` — which provider+model actually served it.
- `X-Fallback-Attempts: N` — number of provider attempts before success.

---

## Rate limiting & failover

Per-key counters: **RPM** (req/min), **RPD** (req/day), **TPM** (tokens/min),
**TPD** (tokens/day). On any limit hit, the key is cooled down and the router
retries the next provider in the fallback chain — **up to 20 attempts** — on
429/5xx/timeout. Periodic health checks set each key to `healthy`,
`rate_limited`, `invalid`, or `error` (visible on the dashboard). Fallback-chain
priority order is customizable in the dashboard.

---

## Environment variables

**Critical**
- `ENCRYPTION_KEY` — 32-byte hex key for at-rest AES-256-GCM encryption (required).

**Routing & features**
- `FREELLMAPI_CONTEXT_HANDOFF=on_model_switch` — inject a compact system message
  when the model changes mid-conversation (helps continuity across failovers).
- `REQUEST_ANALYTICS_RETENTION_DAYS=90` — request-log retention.
- `REQUEST_ANALYTICS_MAX_ROWS=100000` — log row cap.
- `FREEAPI_DB_PATH` — SQLite location (default `server/data/freeapi.db`).
- `PORT=3001`, `HOST_BIND=127.0.0.1` (use `0.0.0.0` for LAN, trusted only),
  `NODE_ENV=production|development`.

**DB backup (ephemeral disks)**
- `FREEAPI_DB_BACKUP_PATH`, `FREEAPI_DB_BACKUP_URL`, `FREEAPI_DB_BACKUP_TOKEN`,
  `FREEAPI_DB_BACKUP_KEY` (64-char hex; falls back to `ENCRYPTION_KEY`),
  `FREEAPI_DB_BACKUP_INTERVAL_MS=300000`.

**Declarative config**
- `FREEAPI_CONFIG_PATH=/path/to/freellmapi.config.json` or `FREEAPI_CONFIG_JSON`
  — idempotent JSON applied on every boot.

---

## Install / run (for reference)

Docker one-liner:
```bash
curl -fsSL https://freellmapi.co/install.sh | bash
```
Manual Docker Compose:
```bash
git clone https://github.com/tashfeenahmed/freellmapi.git
cd freellmapi
ENCRYPTION_KEY="$(openssl rand -hex 32)"
printf "ENCRYPTION_KEY=%s\nPORT=3001\n" "$ENCRYPTION_KEY" > .env
docker compose up -d
```
Dashboard at `http://localhost:3001` (dev UI on `:5173`). Multi-arch
(amd64/arm64, incl. Raspberry Pi); native desktop apps exist for macOS/Windows.

---

## Limitations & ToS notes

- **No frontier models**; quality degrades as daily caps drain; variable latency;
  **no SLA**; single-user/local-first (not multi-tenant).
- Free-tier catalog updates lag Premium ($19/yr or $49 lifetime) by ~30 days.
- Provider ToS: most marked "likely OK" for personal experimentation; **Google
  Gemini, GitHub Models, NVIDIA NIM flagged "caution"** (business-use /
  evaluation-only restrictions). Project stance: *personal experimentation and
  learning, not production.* MIT-licensed.
