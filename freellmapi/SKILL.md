---
name: freellmapi
description: >-
  Reference for FreeLLMAPI, the self-hosted OpenAI-compatible proxy that serves
  the Hermes agent's LLM calls by aggregating the free tiers of ~18 providers
  (Gemini, Groq, Mistral, Cerebras, OpenRouter, Cohere, and more) behind one /v1
  endpoint. Explains the OpenAI-format endpoints, model selection ("auto" and
  pinned IDs), automatic cross-provider failover, the RPM/RPD/TPM/TPD rate-limit
  quotas, the no-frontier-model capability ceiling, inspectable response headers
  (X-Routed-Via), and provider gotchas. Use when reasoning about which model to
  use, overriding or pinning a model, handling 429s / rate limits / provider
  errors, configuring OPENAI_BASE_URL, or using embeddings, vision, tool calling,
  streaming, image generation, or token counting. Full catalog and per-endpoint
  request/response shapes are in references/reference.md.
---

# FreeLLMAPI — the LLM provider behind Hermes

Hermes routes every model call to **FreeLLMAPI**, a local proxy that fans
requests across the free tiers of ~18 upstream providers (Google Gemini, Groq,
Mistral, Cerebras, OpenRouter, Cohere, Cloudflare, NVIDIA NIM, HuggingFace,
Ollama Cloud, and others) behind a single OpenAI-compatible `/v1` surface. It
handles routing, automatic failover, and quota tracking. It is **for personal
experimentation, not production**, and carries **no SLA** — plan for that.

## The capability ceiling — internalize this first

FreeLLMAPI exposes **no frontier models**. The strongest reachable models are
roughly **Llama 3.3 70B, GLM-4.5, Qwen 3 Coder, and Gemini 2.5 Pro/Flash** —
mid tier, not GPT-4/Opus class. Consequences for how Hermes should work:

- **Decompose** hard problems into smaller, verifiable steps instead of
  one-shotting tasks that need frontier reasoning.
- **Expect variable quality and latency** — which model answers depends on which
  upstreams are healthy and un-throttled right now.
- **Quality degrades as daily caps fill.** The router falls back to weaker
  models later in the day; do the hardest reasoning on fresh quota and re-verify
  important results.

## How Hermes connects (OpenAI format)

Hermes is provider-agnostic and points at any OpenAI-compatible endpoint. Set:

```bash
export OPENAI_BASE_URL=http://localhost:3001/v1
export OPENAI_API_KEY=freellmapi-<your-unified-key>
```

The unified key (`freellmapi-…`) replaces every upstream provider credential and
comes from the dashboard **Keys** page at `http://localhost:3001`. All calls go
to `POST /v1/chat/completions` (streaming and non-streaming).

Minimal call:

```bash
curl http://localhost:3001/v1/chat/completions \
  -H "Authorization: Bearer freellmapi-<key>" \
  -H "Content-Type: application/json" \
  -d '{"model": "auto", "messages": [{"role": "user", "content": "hi"}]}'
```

> Non-OpenAI clients: an Anthropic-format surface (`POST /v1/messages`) also
> exists — see references/reference.md. Hermes should use the OpenAI path above.

## Choosing a model

Set the `model` field to one of:

- **`"auto"`** *(default — recommended)*: the router picks the highest-priority
  healthy model that is under its rate limits. Let it drive unless a task needs a
  specific model's strengths.
- **A specific ID** — e.g. `"gemini-2.5-flash"`, `"llama-3.3-70b-versatile"`,
  `"qwen-3-coder"`. Pin only for a concrete reason (e.g. a coding-tuned model for
  code, a fast model for cheap high-volume work).

Never assume a model is installed — the free-tier catalog lags. Confirm with
`GET /v1/models` before pinning. Full catalog: references/reference.md.

## What's supported — and what Hermes actually uses

FreeLLMAPI exposes the full capability set below. Hermes is a text/chat agent, so
it only uses a subset in its normal loop — the **Hermes** column says which:

| Capability | Endpoint | Hermes | Notes |
|---|---|---|---|
| Chat / conversation | `/v1/chat/completions` | **Core** | the main loop |
| Streaming | `/v1/chat/completions` | **Core** | `"stream": true` → SSE deltas |
| Tool / function calling | `/v1/chat/completions` | **Core** | OpenAI `tools`/`tool_choice`; **model-dependent** (see below) |
| List models | `GET /v1/models` | **Core** | source of truth for what's installed |
| Vision (image input) | `/v1/chat/completions` | Optional | only if Hermes sends images; `422 no_vision_model` if none enabled |
| Embeddings | `/v1/embeddings` | Optional | only if Hermes's memory/RAG points here; **failover never crosses families** |
| Image generation | `/v1/images/generations` | Tool only | standard OpenAI shape; usable only if wired as a Hermes tool |
| Text-to-speech | `/v1/audio/speech` | Tool only | standard OpenAI shape; usable only if wired as a Hermes tool |
| Legacy completion | `/v1/completions` | Not used | editor autocomplete |
| Anthropic messages / `count_tokens` | `/v1/messages*` | Not used | Anthropic-format; Hermes is OpenAI (usage comes back in the response `usage` field) |
| Codex responses | `/v1/responses` | Not used | Codex CLI wire format |

**Model-dependent caveat:** under `"auto"`, tool calling and vision work only if
the *routed upstream model* supports them. The proxy guards vision (returns 422
if no vision model is enabled) but does **not** guard tools — so for reliable
tool use, **pin a tool-capable model** (e.g. a Llama/Qwen/Gemini function-calling
model) rather than relying on `"auto"`.

Request/response shapes for each endpoint are in references/reference.md.

## Behavior Hermes can rely on and observe

- **Automatic failover:** on 429 / 5xx / timeout the router retries the next
  provider in the fallback chain — **up to 20 attempts** — before erroring. A
  single upstream 429 usually costs latency, not a failure.
- **Inspect what actually served the request** via response headers:
  - `X-Routed-Via: <platform>/<model>` — the real provider+model that answered.
  - `X-Fallback-Attempts: N` — how many providers were tried.
  When debugging quality or latency, read these — the answering model may not be
  the one requested (especially under `"auto"`).
- **Sticky sessions:** send `X-Session-Id: <id>` to pin a multi-turn
  conversation to one model for 30 minutes and avoid mid-thread model swaps.

## Handling failures

- **Persistent 429 / all attempts exhausted:** per-minute and per-day quotas
  across upstreams are drained. Back off, narrow scope, or switch to a lighter
  model — do not hammer it. RPD/TPD reset daily.
- **`422 no_vision_model`:** an image was sent but no vision-capable upstream is
  enabled. Drop the image or enable a vision model in the dashboard.
- **Embeddings:** the router only retries *within the same embedding family*
  (vectors from different models are incompatible). Don't mix embedding models in
  one index. Default family is `gemini-embedding-001` (3072 dims).
- **Quotas metered per key:** RPM, RPD, TPM, TPD; keys show as `healthy`,
  `rate_limited`, `invalid`, or `error` on the dashboard.

## Gotchas checklist

- Point `OPENAI_BASE_URL` at `.../v1`; the key is `freellmapi-…`, not an upstream
  provider key.
- No frontier models — calibrate ambition and verify important outputs.
- The catalog lags Premium by ~30 days; a documented model may not be installed —
  trust `GET /v1/models`, not docs.
- No SLA; upstream free tiers can change or vanish without notice.
- Some upstreams (Google Gemini, GitHub Models, NVIDIA NIM) carry business-use /
  evaluation-only ToS caveats — keep usage to personal experimentation.

For the exhaustive per-provider model catalog, full request/response examples for
every endpoint, headers, rate-limit detail, and environment variables, read
**references/reference.md**.
