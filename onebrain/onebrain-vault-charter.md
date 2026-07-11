# OneBrain Vault

This directory is a **OneBrain vault** — a plain-Markdown Obsidian knowledge base organized
with the PARA layout below. Help the user capture, organize, synthesize, and retrieve
knowledge here; be proactive about connections, stale tasks, and next actions.

```
00-inbox/   raw captures      03-knowledge/  synthesized insights   06-archive/  done (never deleted)
01-projects/ active projects  04-resources/  external reference      07-logs/     session logs
02-areas/    responsibilities 05-agent/       agent-maintained notes
```

Flow: capture → `00-inbox` → process into `03-knowledge`/`04-resources` → `06-archive` when done.

**Skills** (loaded on demand — ask naturally, or `skill_view`):
`obsidian` (read/search/create/edit + quick capture), `consolidate` (process inbox),
`distill` (synthesize a topic), `daily` (briefing), `wrapup` (session summary).
Full conventions live in the `obsidian` skill: `skill_view("obsidian", "references/onebrain-conventions.md")`.

**Memory tiers:** Working = `00-inbox` + chat · Episodic = `07-logs/session` + Hermes sessions ·
Semantic = Hermes memory (`~/.hermes/memories/`) · Knowledge = `03-knowledge` · Archive = `06-archive`.
Use Hermes's own memory as the Semantic tier — don't build a parallel one.

Principles: think before acting; minimal footprint (don't reorganize unprompted); prefer
extending an existing note over a duplicate; never delete/archive without confirmation.

---

> **Install:** copy this file to your OneBrain vault root as `.hermes.md`. Hermes loads it as a
> project context file (first match of `.hermes.md`, `AGENTS.md`, `CLAUDE.md`, `.cursorrules`)
> from the working directory at session start.
