# Enabling tool calling & vision on Hermes (via FreeLLMAPI)

How to make the Hermes agent use FreeLLMAPI's tool-calling and vision
capabilities. Two sides must both be set up: the **provider** (FreeLLMAPI must
expose a capable model) and the **agent** (Hermes must be pointed at it).

Hermes routes work across a **main model** (the think + tool-call loop) and
**auxiliary models** (side tasks — vision, web-extract, etc.). That split drives
the setup:

- **Tool calling → main model.** Must be a function-calling-capable model.
- **Vision → the `auxiliary.vision` task.** Has its own provider/model/base_url.

> Exact model IDs vary by install. **Always confirm the real IDs with
> `GET /v1/models`** and use those strings below.

## Contents
- Tool calling
- Vision
- Verification
- Caveats

---

## Tool calling

### FreeLLMAPI side
1. Enable at least one upstream whose models support OpenAI function calling —
   e.g. Groq **Llama 3.3 70B**, **Gemini 2.5** (Flash/Pro), Mistral, or a Qwen
   function-calling model. Add the provider key in the dashboard.
2. In the dashboard fallback chain, rank tool-capable models first so failover
   stays on models that support tools.
3. Confirm the model is present: `GET /v1/models`.

### Hermes side
Point the **main model** at FreeLLMAPI and **pin a tool-capable model** — do not
rely on bare `"auto"`, since the router may land on a model without tool support
(the proxy guards vision with a 422 but does **not** guard tools):

```yaml
model:
  default: "llama-3.3-70b-versatile"   # a tool-capable model ID from GET /v1/models
  provider: "auto"
  base_url: "http://localhost:3001/v1"
  # api_key: "freellmapi-<key>"        # or set OPENAI_API_KEY in .env
```

Hermes emits OpenAI `tools`/`tool_choice` to the main model automatically. Then
register the tools you want it to have — built-in toolsets and/or MCP servers:

```yaml
mcp_servers:
  - name: github
    # stdio (subprocess) or http transport per Hermes MCP docs
    # supports_parallel_tool_calls: true   # opt-in for safe parallel tools
```

No Hermes code change is needed — tool calling is native once the main model
supports it and tools are registered.

---

## Vision

### FreeLLMAPI side
Enable a **vision-capable** model/provider — e.g. Google **Gemini 2.5 Flash/Pro**
(or another VL model). Without one, image requests return
`422 { code: "no_vision_model" }`. Confirm with `GET /v1/models`.

### Hermes side
Vision is an auxiliary task. Point `auxiliary.vision` at FreeLLMAPI with a
vision-capable model (raise the timeout — image download + inference is slower):

```yaml
auxiliary:
  vision:
    provider: "custom"                 # custom OpenAI-compatible endpoint
    model: "gemini-2.5-flash"          # a vision-capable model ID from GET /v1/models
    base_url: "http://localhost:3001/v1"
    # api_key: "freellmapi-<key>"
    timeout: 120
    download_timeout: 30
```

Hermes sends images as OpenAI `image_url` content blocks to this task
automatically.

---

## Verification

- **Tool calling:** give Hermes a task that requires a registered tool; confirm
  the `tool_calls` round-trip completes, and read the response header
  `X-Routed-Via: <platform>/<model>` to confirm a tool-capable model answered.
- **Vision:** send an image. Success = a description; `422 no_vision_model` =
  no vision model enabled on the FreeLLMAPI side.

---

## Caveats

- Hermes docs note auxiliary-task **custom provider overrides are experimental
  beyond OpenRouter/Nous** — vision via a custom FreeLLMAPI endpoint works but is
  less battle-tested; keep the `base_url` override explicit.
- `"auto"` can route tool/vision requests to an incapable model. Pin capable
  models (main model for tools; `auxiliary.vision.model` for vision).
- Capability ceiling still applies — these are mid-tier models; verify important
  tool outputs and image interpretations.
- Model IDs above are examples. The authoritative list is `GET /v1/models`.
