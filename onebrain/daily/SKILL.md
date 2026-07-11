---
name: daily
description: Brief the day: tasks due plus last-session context
version: 1.0.0
metadata:
  hermes:
    category: note-taking
    tags: [obsidian, para, tasks, briefing]
---

# Daily (OneBrain briefing)

Give the user a short start-of-day briefing from the vault. Read-only.

## When to Use

The user says "daily", "briefing", "what's on today", "catch me up", or starts the day.

## Procedure

1. Resolve the vault path (see the `obsidian` skill).
2. `search_files` (`target: "content"`, `file_glob: "*.md"`) for open dated checkboxes:
   pattern `- \[ \].*📅`. Collect due-today and overdue items with their note paths.
3. `search_files` under `07-logs/session/` for the most recent summary note; `read_file` it
   for open threads / next steps.
4. Present a concise briefing: **Due today**, **Overdue**, **Where you left off** (last
   session), and a **Suggested focus** of 1–3 items.

## Pitfalls

- Read-only — never modify notes in a briefing.
- If there are no tasks or no session log, say so briefly rather than inventing items.
- Keep it scannable; reference notes by path or `[[wikilink]]`.

## Verification

The briefing lists real due/overdue tasks (or clearly states there are none) and reflects the
most recent session log, with no notes modified.
