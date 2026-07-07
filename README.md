# myskills

Personal Agent Skills for my agents (Hermes, etc.).

Each top-level directory is a self-contained skill: a `SKILL.md` with YAML
frontmatter (`name` + a trigger `description`) and optional `references/` files
that are loaded on demand (progressive disclosure).

## Skills

| Skill | What it does |
|---|---|
| [`freellmapi/`](./freellmapi/SKILL.md) | Teaches Hermes about **FreeLLMAPI**, the OpenAI/Anthropic-compatible proxy that serves all of its LLM calls — endpoints, model selection, failover, rate limits, capability ceiling, and gotchas. |

## Using a skill

- **Claude Code:** symlink or copy a skill directory into `~/.claude/skills/`
  (personal) or `.claude/skills/` in a project. It auto-loads when the
  `description` matches the task.
- **Anthropic API / Agent SDK:** point your skills directory at this repo, or
  package a skill as part of a plugin.
