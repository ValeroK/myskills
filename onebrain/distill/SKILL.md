---
name: distill
description: Synthesize a topic into a 03-knowledge digest note
version: 1.0.0
metadata:
  hermes:
    category: note-taking
    tags: [obsidian, para, synthesis]
---

# Distill (OneBrain topic synthesis)

Gather everything the vault knows about a topic and write one clean synthesis note in
`03-knowledge/`. Uses Hermes file tools.

## When to Use

The user says "distill X", "synthesize what I know about X", or "make a knowledge note on X".
Load `skill_view("obsidian", "references/onebrain-conventions.md")` first for the frontmatter
and folder conventions if not already in context.

## Procedure

1. Resolve the vault path (see the `obsidian` skill).
2. `search_files` (`target: "content"`, `file_glob: "*.md"`) for the topic across the vault;
   `read_file` the strongest matches.
3. Synthesize in the user's voice: key points, open questions, connections — distill, don't
   concatenate.
4. Add a `## Sources` section with `[[wikilinks]]` to the notes you drew from.
5. `write_file` to `03-knowledge/<subfolder>/Topic Name.md` with frontmatter `tags`,
   `created`, and confidence (`conf: high|medium|low`, `verified: YYYY-MM-DD`). If a digest
   already exists, `patch`/update it instead of duplicating.
6. Report the file written and which sources fed it.

## Pitfalls

- Keep the user's own insight in `03-knowledge/`; external material belongs in `04-resources/`.
- Mark uncertain claims `conf: low`; never overstate confidence.
- Prefer updating an existing digest over creating a second one.

## Verification

A single `03-knowledge/` note captures the topic, carries confidence frontmatter, and links
its sources; no duplicate digest was created.
