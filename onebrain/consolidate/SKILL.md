---
name: consolidate
description: Process the OneBrain inbox into permanent PARA notes
version: 1.0.0
metadata:
  hermes:
    category: note-taking
    tags: [obsidian, para, knowledge-management]
---

# Consolidate (OneBrain inbox → knowledge)

Turn raw `00-inbox/` captures into well-formed permanent notes filed under the right PARA
folder, then clear the inbox. Uses only Hermes file tools.

## When to Use

The user says "process my inbox", "consolidate", "clean up inbox", or after a burst of
captures. Load `skill_view("obsidian", "references/onebrain-conventions.md")` first if the
PARA/naming conventions aren't already in context.

## Procedure

1. Resolve the vault path (see the `obsidian` skill), then `search_files` (`target: "files"`,
   `pattern: "*.md"`) under `00-inbox/`. If empty, say so and stop.
2. For each inbox note: `read_file` it, then pick the destination —
   `01-projects/` (project material), `02-areas/` (ongoing responsibility),
   `03-knowledge/` (the user's own insight), or `04-resources/` (external reference).
3. `search_files` for related notes; collect 1–3 `[[wikilinks]]`.
4. Compose the permanent note — frontmatter (`tags`, `created`), cleaned body, and a
   `## Related` section — and `write_file` it at `[dest]/Topic Name.md`.
5. After approval, delete the original inbox file once its content is safely filed.
6. Summarize what moved where (source → destination, links added).

## Pitfalls

- Merge into an existing note when a strong fit exists (prefer `patch`/append over a duplicate).
- Never invent facts — carry only what the capture says.
- Get explicit approval before any deletion.

## Verification

`00-inbox/` is empty (or only holds items you deliberately left), and each processed item
exists as a permanent note under the correct PARA folder with frontmatter and links.
