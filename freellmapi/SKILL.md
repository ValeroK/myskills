---
name: freellmapi
description: >-
  FreeLLMAPI is the LLM provider behind ALL of Hermes's model calls — a
  self-hosted, OpenAI- AND Anthropic-compatible proxy that aggregates the free
  tiers of ~18 providers behind one /v1 endpoint. Read this whenever you reason
  about your own capabilities or model choice, override or pin a model, hit rate
  limits / 429s / provider errors, or need embeddings, vision, tool-calling,
  streaming, image generation, TTS, or token counting. Covers the endpoints,
  model selection ("auto" + Claude family aliases), automatic failover, the
  RPM/RPD/TPM/TPD quotas, the no-frontier-model capability ceiling, response
  headers you can inspect (X-Routed-Via), and the provider gotchas that change
  how you should plan work. Full model catalog and per-endpoint request/response
  shapes live in references/reference.md — load it before making non-obvious calls.
---

# FreeLLMAPI — your LLM provider

Every model call you make is served by **FreeLLMAPI**, a local proxy that fans
your requests out across the free tiers of ~18 upstream providers (Google
Gemini, Groq, Mistral, Cerebras, OpenRouter, Cohere, Cloudflare, NVIDIA NIM,
HuggingFace, Ollama Cloud, and more) behind a single OpenAI/Anthropic-compatible
`/v1` surface. It handles routing, automatic failover, and quota tracking for
you. It is **for personal experimentation, not production**, and offers **no
SLA** — plan accordingly.

## The one thing to internalize: your capability ceiling

FreeLLMAPI exposes **no frontier models.** The strongest models you can reach are
roughly **Llama 3.3 70B, GLM-4.5, Qwen 3 Coder, and Gemini 2.5 Pro/Flash** — mid
tier, not Opus/GPT-class. Calibrate ambition to that:

- Prefer decomposing hard problems into smaller, verifiable steps over
  one-shotting a task that needs frontier reasoning.
- Expect **variable quality and latency** — the model that answers depends on
  which upstreams are healthy and un-throttled *right now*.
- **Quality degrades as daily caps fill up:** the router falls back to weaker
  models later in the day. Do your hardest reasoning early / on fresh quota when
  you can, and re-verify important results.

## How your calls reach it

Two wire formats, same proxy. You (a Claude agent) almost always use the
**Anthropic** surface:

- **Anthropic (Claude) — `POST /v1/messages`:** speaks Anthropic's native format.
  Configured via env, not code:
  ```bash
  export ANTHROPIC_BASE_URL=http://localhost:3001
  export ANTHROPIC_AUTH_TOKEN=freellmapi-<your-unified-key>   # NOT ANTHROPIC_API_KEY
  ```
  ⚠️ Use `ANTHROPIC_AUTH_TOKEN`. Setting `ANTHROPIC_API_KEY` makes Claude Code
  treat it as a conflicting credential and refuse to start.
- **OpenAI — `POST /v1/chat/completions`:** for OpenAI SDK / raw HTTP tools.
  Base URL `http://localhost:3001/v1`, `Authorization: Bearer freellmapi-<key>`.

The unified key (`freellmapi-…`) replaces every upstream provider credential and
comes from the dashboard **Keys** page (`http://localhost:3001`).

## Choosing a model

Set the `model` field to one of:

- **`"auto"`** *(default, recommended)* — router picks the highest-priority
  healthy model that's under its rate limits. Let it drive unless you have a
  reason not to.
- **A specific ID** — e.g. `"gemini-2.5-flash"`, `"llama-3.3-70b-versatile"`.
  Pin only when a task needs a specific model's strengths (e.g. a coder model).
- **Claude family aliases** *(on `/v1/messages` only)* — `"claude-opus"`,
  `"claude-sonnet-4-5"`, `"claude-haiku"`, `"claude-default"`. These map to
  *pinned upstream models* on the dashboard's Keys → Anthropic tab — they are
  **not** real Anthropic models, just routing labels.

Full catalog (161 models across 18 providers): `references/reference.md`.

## What's supported

| Capability | Endpoint | Notes |
|---|---|---|
| Chat / conversation | `/v1/chat/completions`, `/v1/messages` | streaming + non-streaming |
| Streaming | both chat endpoints | `"stream": true` → SSE deltas |
| Tool / function calling | `/v1/chat/completions` | OpenAI `tools`/`tool_choice`, multi-step |
| Vision (image input) | chat endpoints | `image_url` blocks; 422 if no vision model enabled |
| Embeddings | `/v1/embeddings` | **failover never crosses model families** |
| Token counting | `/v1/messages/count_tokens` | Anthropic format |
| Image generation | `/v1/images/generations` | |
| Text-to-speech | `/v1/audio/speech` | |
| Legacy completion | `/v1/completions` | prompt/suffix autocomplete |
| List models | `GET /v1/models` | Anthropic shape if `anthropic-version` header sent, else OpenAI |

## Behavior you can rely on and observe

- **Automatic failover:** on 429 / 5xx / timeout the router retries the next
  provider in the fallback chain — **up to 20 attempts** — before erroring. A
  single 429 from an upstream usually costs you latency, not a failure.
- **Inspect what actually served you** via response headers:
  - `X-Routed-Via: <platform>/<model>` — the real provider+model that answered.
  - `X-Fallback-Attempts: N` — how many providers were tried.
  Log/surface these when debugging quality or latency — the answering model may
  not be the one you asked for.
- **Sticky sessions:** send `X-Session-Id: <id>` to pin a multi-turn
  conversation to one model for 30 minutes and avoid mid-thread model swaps.

## Handling failures

- **Persistent 429 / all attempts exhausted:** daily/minute quotas across
  upstreams are drained. Back off, narrow scope, or switch to a lighter model —
  don't hammer it. Quotas reset (RPD/TPD daily).
- **`422 no_vision_model`:** you sent an image but no vision-capable upstream is
  enabled. Drop the image or ask the user to enable a vision model in the
  dashboard.
- **Embeddings:** the router only retries *within the same embedding family*
  (vectors from different models are incompatible). Don't mix embedding models
  in one index. Default family is `gemini-embedding-001` (3072 dims).
- **Quota tracking:** the proxy meters **RPM, RPD, TPM, TPD** per key and marks
  keys `healthy` / `rate_limited` / `invalid` / `error`. Health is visible on
  the dashboard.

## Gotchas checklist

- `ANTHROPIC_AUTH_TOKEN`, never `ANTHROPIC_API_KEY`.
- Claude alias names are routing labels, not real Anthropic models — no frontier
  capability behind them.
- Free-tier catalog lags Premium by ~30 days; a model you read about may not be
  installed yet — trust `GET /v1/models`, not documentation.
- No SLA, no guarantees; upstream free tiers can change or vanish without notice.
- Some upstreams (Google Gemini, GitHub Models, NVIDIA NIM) carry
  business-use/evaluation-only ToS caveats — keep usage to personal
  experimentation.

For exhaustive per-provider model lists, rate-limit specifics, full
request/response examples for every endpoint, and environment variables, read
**`references/reference.md`**.
