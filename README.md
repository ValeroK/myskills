# myskills

Personal [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
for my agents (the [Hermes agent](https://github.com/nousresearch/hermes-agent), etc.).

Each top-level directory is a self-contained skill: a `SKILL.md` with YAML
frontmatter (`name` + a trigger `description`) and optional `references/` files
that are loaded on demand (progressive disclosure). Skills follow Anthropic's
authoring guidance: concise body (<500 lines), third-person description with
specific trigger terms, and reference files kept one level deep.

## Skills

| Skill | What it does |
|---|---|
| [`freellmapi/`](./freellmapi/SKILL.md) | Teaches Hermes about **FreeLLMAPI**, the self-hosted OpenAI-compatible proxy that serves all of its LLM calls — how to connect (`OPENAI_BASE_URL`), model selection, cross-provider failover, rate-limit quotas, the no-frontier-model capability ceiling, and provider gotchas. |

## Using a skill

- **Claude Code:** symlink or copy a skill directory into `~/.claude/skills/`
  (personal) or `.claude/skills/` in a project. It auto-loads when the
  `description` matches the task.
- **Anthropic API / Agent SDK:** point your skills directory at this repo, or
  package a skill as part of a plugin.
