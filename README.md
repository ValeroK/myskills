# myskills

Personal [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
for my agents (the [Hermes agent](https://github.com/nousresearch/hermes-agent), etc.).

Each top-level directory is a self-contained skill: a `SKILL.md` with YAML
frontmatter (`name` + a trigger `description`, plus Hermes's `version` /
`metadata.hermes` fields) and optional `references/` files that are loaded on
demand (progressive disclosure). The layout works with **both** the Hermes agent
and Anthropic Agent Skills / Claude Code. Skills follow Anthropic's authoring
guidance: concise body (<500 lines), third-person description with specific
trigger terms, and reference files kept one level deep.

## Skills

| Skill | What it does |
|---|---|
| [`freellmapi/`](./freellmapi/SKILL.md) | Teaches Hermes about **FreeLLMAPI**, the self-hosted OpenAI-compatible proxy that serves all of its LLM calls — how to connect (`OPENAI_BASE_URL`), model selection, cross-provider failover, rate-limit quotas, the no-frontier-model capability ceiling, and provider gotchas. |

## Installing in the Hermes agent

Hermes has a native skills system: it loads markdown skills from
`~/.hermes/skills/` (its source of truth) plus any read-only directories listed
under `skills.external_dirs` in the Hermes config. Each skill is a folder
`category/skill-name/` containing a `SKILL.md` (+ optional `references/`,
`scripts/`, `templates/`, `assets/`), and every installed skill is exposed as a
slash command. This repo's `freellmapi/` skill already matches that layout.

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
