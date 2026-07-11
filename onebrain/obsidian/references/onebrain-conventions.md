# OneBrain Conventions

Full vault conventions for this OneBrain vault. Loaded on demand from the `obsidian` skill.

## PARA folders

```
00-inbox/        Raw braindumps and quick captures (process regularly)
01-projects/     Active projects with a goal and an end date
02-areas/        Ongoing responsibilities (health, finances, career…)
03-knowledge/    The user's own synthesized thinking and insights
04-resources/    External info — research, summaries, reference material
05-agent/        Agent context (durable notes the agent maintains)
06-archive/      Completed projects and retired areas — never deleted
07-logs/         Session logs and skill logs
```

Core flow: **capture → `00-inbox` → process into `03-knowledge` / `04-resources` → `06-archive` when done.**

## Naming conventions

- Notes: `[folder]/[subfolder]/Topic Name.md` — Title Case filename, kebab-case subfolder.
- Subfolders: lowercase-with-hyphens, max 2 levels deep (`technology/ai` ok, deeper not).
- Inbox items: `00-inbox/YYYY-MM-DD-topic.md` (flat, no subfolders).
- Session logs: `07-logs/session/YYYY/MM/YYYY-MM-DD-session-NN.md`.
- Pick the best subfolder automatically; the user can move it later.

## Frontmatter

Every new note starts with:

```yaml
---
tags: [topic, type]
created: YYYY-MM-DD
---
```

Promoted knowledge in `03-knowledge/` may carry confidence frontmatter — `conf: high|medium|low`
and `verified: YYYY-MM-DD` — so it grows more reliable as it is re-verified.

## Related links

Before creating a note, `search_files` the vault and add the top 1–3 relevant `[[wikilinks]]`
under a `## Related` section. Omit the section if nothing relevant is found.

## Tasks (Obsidian Tasks plugin)

Write action items as Markdown checkboxes with a due-date emoji, inline in the note they
belong to (never a standalone file):

```
- [ ] Task description 📅 YYYY-MM-DD
- [ ] High-priority task 🔺 📅 YYYY-MM-DD
```

Priority markers: `🔺` high · `⏫` medium · `🔽` low.

## Four-tier memory (mapped onto Hermes)

Knowledge sinks downward as it earns trust; recall upward on demand.

| Tier | Lives in | Written when |
|---|---|---|
| **Working** | `00-inbox/` + current conversation | raw capture |
| **Episodic** | Hermes `sessions/` + `07-logs/session/` summary notes | end of session (wrapup) |
| **Semantic** | Hermes memory (`~/.hermes/memories/MEMORY.md`, `USER.md`) | a durable fact/preference is confirmed |
| **Knowledge** | `03-knowledge/` | a topic is synthesized (distill) |
| **Archive** | `06-archive/` (dormant, never deleted) | work is complete |

Use **Hermes's own memory** as the Semantic tier — do not build a parallel memory system.
The PARA folders are the durable, human-navigable knowledge base that complements it.

## Companion OneBrain skills

| Skill | Purpose |
|---|---|
| `consolidate` | Process `00-inbox/` into permanent knowledge/resources |
| `distill` | Crystallize a topic across notes into a `03-knowledge/` digest |
| `daily` | Daily briefing — tasks due + last-session context |
| `wrapup` | Write a session summary note to `07-logs/session/` |

Quick capture (a note straight into `00-inbox/` with auto-linking) is handled by the
`obsidian` skill directly — no separate skill needed.
