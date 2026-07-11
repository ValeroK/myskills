---
name: web-retrieval-choice
description: >-
  Decision guide for choosing the right web-retrieval tool in Hermes:
  scrapper-tool (structured or bot-hostile scraping), Firecrawl
  (web_search / web_extract for HTML → LLM-ready JSON/Markdown, incl. bulk and
  JS-heavy pages), or Perplexity (mcp pplx_* cited AI answers and deep research).
  Covers when to use each, the Perplexity intent/quota tiers, pre-flight
  key/health checks, and combined discover-then-extract workflows. Use when
  deciding how to fetch or research something from the web, when a scrape is
  blocked by anti-bot defenses, when picking between web_search/web_extract and
  pplx_smart_query/pplx_deep_research, or when one retrieval attempt fails and
  you need the next option.
version: 1.0.0
metadata:
  hermes:
    category: research
    tags: [web, scraping, search, firecrawl, perplexity, scrapper-tool, retrieval, research]
---

# Web retrieval — which tool to use

Hermes has three web-retrieval paths. Choose by *what you need back*, not habit.

| You need… | Use | How |
|---|---|---|
| Specific structured fields, or a bot-blocked / hostile site | **scrapper-tool** | `scrapper_tool_wrapper` (`mode: auto` → `hostile`) |
| Page content / HTML as clean JSON/Markdown for an LLM — esp. bulk or JS-heavy | **Firecrawl** | `web_search` (find URLs) + `web_extract` (pull content) |
| A cited answer, exploration, or a multi-source report | **Perplexity** | `pplx_smart_query`, `pplx_deep_research_start`, `pplx_council` |

## Quick decision

1. Need specific structured data, or the site fights scrapers (Cloudflare /
   CAPTCHA)? → **scrapper-tool** (`mode: auto`, escalate to `hostile`). If it
   still fails, follow the `anti-bot-extraction` skill.
2. Need page content in LLM-ready form, or bulk-extract many similar URLs? →
   **Firecrawl** (`web_extract`; run `web_search` first if you lack URLs).
3. Asking a *question* (facts, synthesis, "what / why / compare")? →
   **Perplexity**.
4. No URLs yet but then need structured data? → **Perplexity to discover URLs →
   Firecrawl / scrapper-tool to extract**.

## Firecrawl scope ceiling (important)

Hermes wires Firecrawl as **search + scrape only** (`web_search`, `web_extract`).
The `crawl` (whole-site), `map` (URL discovery/sitemap), and `interact`
(clicks / forms / login) endpoints Firecrawl offers are **not exposed** by our
`web` tool. So:

- Need to crawl an entire site or map its URLs? `web_extract` won't do it —
  discover URLs via **Perplexity** or `web_search`, then extract each, or hit the
  Firecrawl REST API (`/v2/crawl`, `/v2/map`) directly.
- Page needs clicks / a form / login before content appears? `web_extract` can't
  drive the page — use **scrapper-tool** (`mode: hostile`) or Firecrawl
  `/v2/interact` via REST.

## Keyless fallback (last resort)

If `FIRECRAWL_API_KEY` is missing or you're rate-limited (429), Firecrawl's
**keyless free tier** still serves search/scrape/interact (rate-limited):

- MCP endpoint: `https://mcp.firecrawl.dev/v2/mcp`
- REST: same `/v2/*` endpoints with **no** `Authorization` header

Prefer the keyed path (higher limits, all endpoints); use keyless only when no key
is available.

## Perplexity intents & quota — know before you call

`pplx_smart_query` with `intent`:
- `quick` — simple facts, **no Pro-Search quota**.
- `standard` / `detailed` — deeper, cited; **1 Pro-Search each**.
- Deep report → `pplx_deep_research_start`, then poll `pplx_research_status`;
  **1 Deep-Research quota**.
- Multiple model perspectives → `pplx_council`.
- Add `"thinking": true` for logic / math / strategy questions.

Pro-Search and Deep-Research quotas are limited — match the intent to the need.

## Pre-flight checks (run first)

```bash
# Firecrawl key set?
[ -n "$FIRECRAWL_API_KEY" ] && echo "firecrawl ✓" || echo "firecrawl ✗ set FIRECRAWL_API_KEY"
# Perplexity Pro reachable?
hermes mcp pplx_usage | grep -q Pro && echo "perplexity ✓" || echo "perplexity ✗ re-auth"
# scrapper-tool container up?
curl -s http://localhost:5792/health | grep -q '"status":"ok"' && echo "scrapper ✓" || echo "scrapper ✗ start container"
```

## Invocation — call these as native tools

These are Hermes tools the agent calls directly, **not shell commands**:

- **Firecrawl** → the built-in `web` tool: `web_search` to discover URLs,
  `web_extract` to pull content, passing the URLs plus a natural-language
  description of the fields you want. Needs `FIRECRAWL_API_KEY` in the Hermes
  environment (see pre-flight check).
- **Perplexity** → the `pplx_*` MCP tools: `pplx_smart_query`
  (`{"query":"…","intent":"quick|standard|detailed"}`), `pplx_deep_research_start`
  (then poll `pplx_research_status`), or `pplx_council`.
- **scrapper-tool** → the `scrapper_tool_wrapper` tool's `scrape` action, e.g.
  args `{"url":"…","mode":"auto"}` (escalate `mode` to `hostile`), or add
  `"schema_json": {…}` for structured extraction. See the `scrapper-tool-wrapper`
  skill for startup and argument detail.

## Combined: discover → extract

When you lack URLs but need structured data: use Perplexity to surface candidate
URLs, then feed them to Firecrawl `web_extract` (or scrapper-tool with a schema)
for the structured pull.

## Writing a research deliverable

When the output is a written brief/summary/comparison (not raw data): **cite every
claim with its source URL** and end with a `Sources` list. The model won't do this
by default — without the instruction it answers from memory and omits sources.

## Related skills

- `scrapper-tool-wrapper` — the scrapper-tool executor (startup + args detail).
- `anti-bot-extraction` — fallback playbook when a site blocks scraping.
- `freellmapi` — model/provider guidance when post-processing retrieved data
  with an LLM.
