# myskills

Personal [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
for my agents (the [Hermes agent](https://github.com/nousresearch/hermes-agent), etc.).

Each skill is a directory with a `SKILL.md` — YAML frontmatter (`name` + a
trigger `description`, plus Hermes's `version` / `metadata.hermes` fields) and
optional `references/` files loaded on demand (progressive disclosure).
Standalone skills sit at the repo root (`freellmapi/`, `omniroute/`, `skill-doctor/`, `web-retrieval-choice/`);
related skills are grouped under a **family folder** (`onebrain/`) — the folder
is organizational only, each subdirectory is still a separate, independently
triggerable skill. The layout works with **both** the Hermes agent and Anthropic
Agent Skills / Claude Code. Skills follow Anthropic's authoring guidance: concise
body (<500 lines), third-person description with specific trigger terms, and
reference files kept one level deep.

## Skills

| Skill | What it does |
|---|---|
| [`freellmapi/`](./freellmapi/SKILL.md) | Teaches Hermes about **FreeLLMAPI**, the self-hosted OpenAI-compatible proxy that serves all of its LLM calls — how to connect (`OPENAI_BASE_URL`), model selection (`auto`/`fusion`/pinned), cross-provider failover, rate-limit quotas, the no-frontier-model capability ceiling, and provider gotchas. |
| [`omniroute/`](./omniroute/SKILL.md) | Teaches Hermes about **OmniRoute**, a self-hosted OpenAI-compatible LLM gateway (`http://127.0.0.1:20128/v1`, interchangeable with FreeLLMAPI) — how to point Hermes at it, the `auto/*` routing combos vs. pinning a concrete model, inspecting which model served a request, and MCP/A2A + health checks. Secure Docker install lives in [`references/install.md`](./omniroute/references/install.md), loaded on demand. |
| [`web-retrieval-choice/`](./web-retrieval-choice/SKILL.md) | Decides **which web-retrieval tool** to use in Hermes — scrapper-tool (structured / bot-hostile scraping), Firecrawl (`web_search`/`web_extract`), or Perplexity (`pplx_*` cited answers / deep research) — with the Perplexity intent/quota tiers, pre-flight key/health checks, and combined discover→extract workflows. Merged from the former `firecrawl-perplexity-scrapper-choice` + `firecrawl_vs_perplexity` skills. |
| [`onebrain/obsidian/`](./onebrain/obsidian/SKILL.md) | Filesystem-first Obsidian vault work (read/search/create/edit + quick capture), extended with the **OneBrain** PARA layout and four-tier memory model. Heavy conventions live in [`onebrain/obsidian/references/onebrain-conventions.md`](./onebrain/obsidian/references/onebrain-conventions.md), loaded on demand. |
| [`onebrain/consolidate/`](./onebrain/consolidate/SKILL.md) | Process the OneBrain `00-inbox` into permanent PARA notes with frontmatter + `[[wikilinks]]`, then clear the inbox. |
| [`onebrain/distill/`](./onebrain/distill/SKILL.md) | Synthesize a topic scattered across notes into one `03-knowledge` digest with confidence frontmatter and source links. |
| [`onebrain/daily/`](./onebrain/daily/SKILL.md) | Daily briefing — tasks due/overdue plus context from the last session log. |
| [`onebrain/wrapup/`](./onebrain/wrapup/SKILL.md) | Write an end-of-session summary note to the `07-logs/session` tree. |
| [`skill-doctor/`](./skill-doctor/SKILL.md) | Grades the agent setup from **real local conversation history** (Claude Code / Codex / Warp) against efficiency + code-quality rubrics, then drafts concrete `SKILL.md` edits and renders one self-contained `report.html`. Vendored from [warpdotdev/common-skills](https://github.com/warpdotdev/common-skills/tree/main/.agents/skills/skill-doctor) (MIT). How it differs from Anthropic's `skill-creator` / `claude plugin eval`: [`skill-doctor/COMPARISON.md`](./skill-doctor/COMPARISON.md). |

### Running `skill-doctor` on this repo

`skill-doctor` discovers skills from `.agents/skills`, `.claude/skills`, and
`.codex/skills`. This repo keeps skills at the root instead, so point the
collector at them explicitly:

```bash
python3 skill-doctor/scripts/collect_sessions.py --out "$REPORT_DIR" \
  --repo . --skills-dir . --skills-dir ./onebrain --include-global-skills
```

Two things to know: its inventory parser reads a single-line `description:`, so
the folded (`>-`) descriptions used by most skills here show up empty in
`inventory.json` (cosmetic — scoring is unaffected), and the rendered report
carries Warp's "Request access to Warp Factories" call to action.

### OneBrain skill set (`onebrain/`)

`obsidian`, `consolidate`, `distill`, `daily`, and `wrapup` together port the
[OneBrain](https://github.com/…/onebrain) personal-knowledge workflow onto Hermes's native
skills system. They all target a **OneBrain vault** (PARA folders `00-inbox` … `07-logs`),
and are grouped under `onebrain/` in this repo. They remain five separate skills (five
`/slash` commands); the folder just signals the family.

The companion vault charter, [`onebrain/onebrain-vault-charter.md`](./onebrain/onebrain-vault-charter.md), is a
project context file — **copy it to your vault root as `.hermes.md`** so Hermes always knows
the vault's structure, conventions, and memory-tier mapping. It is intentionally small
(always-on), while the full conventions load on demand from the `obsidian` skill's
`references/`.

## Installing in the Hermes agent

Hermes has a native skills system: it loads markdown skills from
`~/.hermes/skills/` (its source of truth) plus any read-only directories listed
under `skills.external_dirs` in the Hermes config. Each skill is a folder
`category/skill-name/` containing a `SKILL.md` (+ optional `references/`,
`scripts/`, `templates/`, `assets/`), and every installed skill is exposed as a
slash command. This repo's `freellmapi/` skill already matches that layout.

The `onebrain/` family folder is a **repo-organization layer only** — it is not a
Hermes category. Deploy each of its subfolders into the target Hermes category
(these five ship under `note-taking/`), e.g. `cp -r onebrain/obsidian
~/.hermes/skills/note-taking/`.

Pick one of the two methods:

### Method 1 — copy into `~/.hermes/skills/` (simplest)

```bash
mkdir -p ~/.hermes/skills/llm-providers
cp -r freellmapi ~/.hermes/skills/llm-providers/
# → ~/.hermes/skills/llm-providers/freellmapi/SKILL.md
```

### Method 2 — link this repo via `skills.external_dirs` (stays in git, auto-updates)

Clone the repo, then add its path to the Hermes config (`~/.hermes/config.yaml`
or `cli-config.yaml`). Paths support `~` and `${VAR}` expansion; a local
`~/.hermes/skills/` skill of the same name takes precedence over an external one.

```yaml
skills:
  external_dirs:
    - ${HOME}/myskills            # this repo's checkout
```

### Verify and use

1. Restart Hermes (skills are discovered/registered at startup).
2. Confirm it loaded — it should appear in `skills_list()` and as the
   `/freellmapi` slash command.
3. Use it: `/freellmapi how should I pick a model for a tool-calling task?`
   Hermes reads `SKILL.md` first and pulls in `references/*.md` on demand via its
   three-level progressive disclosure (`skills_list` → `skill_view` →
   `skill_view` with a path).

The skill's frontmatter carries the Hermes metadata Hermes uses to categorize and
surface it (`version`, `metadata.hermes.category`, `metadata.hermes.tags`).

## Using a skill in Claude Code / Anthropic API

The same `SKILL.md` format works with Anthropic Agent Skills:

- **Claude Code:** symlink or copy a skill directory into `~/.claude/skills/`
  (personal) or `.claude/skills/` in a project. It auto-loads when the
  `description` matches the task. (The extra `version`/`metadata.hermes`
  frontmatter fields are ignored by Claude Code.)
- **Anthropic API / Agent SDK:** point your skills directory at this repo, or
  package a skill as part of a plugin.
