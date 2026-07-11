---
name: wrapup
description: Write an end-of-session summary to 07-logs
version: 1.0.0
metadata:
  hermes:
    category: note-taking
    tags: [obsidian, para, session, memory]
---

# Wrapup (OneBrain session summary)

Close a working session by writing a durable summary note to the `07-logs/session` tree.
Uses Hermes file tools.

## When to Use

The user says "wrap up", "end session", "log this session", or is finishing for the day.

## Procedure

1. Resolve the vault path (see the `obsidian` skill).
2. Determine the log path `07-logs/session/YYYY/MM/YYYY-MM-DD-session-NN.md` (`search_files`
   to increment `NN` if earlier sessions exist that day).
3. Compose the summary with frontmatter (`tags: [session]`, `created`) and sections:
   **Done**, **Decisions**, **Open threads / next steps**, **Links** (`[[wikilinks]]` to notes
   touched).
4. `write_file` the note (the `YYYY/MM` folders are created from the path).
5. If a durable fact or preference was confirmed, also record it in Hermes memory
   (`~/.hermes/memories/`) — not just the log.
6. Report the path written.

## Pitfalls

- Summarize honestly from the actual session — don't embellish.
- The log is Episodic memory; lasting facts belong in Hermes Semantic memory too.

## Verification

A session note exists at the dated `07-logs/session/...` path with the four sections filled,
and any durable fact was mirrored into Hermes memory.
