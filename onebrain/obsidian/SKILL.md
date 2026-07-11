---
name: obsidian
description: Read, search, create, edit Obsidian vault notes (PARA)
platforms: [linux, macos, windows]
---

# Obsidian Vault

Use this skill for filesystem-first Obsidian vault work: read notes, list notes, search notes, create notes, append content, add wikilinks, and write into the vault from Hermes.

## Prerequisites and activation

- The `obsidian` skill operates **entirely through Hermes file tools** (`read_file`, `write_file`, `search_files`, `patch`, and `terminal` for environment resolution).  **No `hermes plugin install obsidian` step is needed**, and the skill does NOT add new Hermes tools.
- The skill is available as soon as the agent loads it — `skill_view(name='obsidian')`.
- No API keys, no external network calls; all work stays on the local filesystem.
- The vault must exist as a regular directory — create it with `mkdir -p` if it does not.

## Vault path

Use a known or resolved vault path before calling file tools.

The documented vault-path convention is the `OBSIDIAN_VAULT_PATH` environment variable, for example from `${HERMES_HOME:-~/.hermes}/.env`. If it is unset, use `~/Documents/Obsidian Vault`.

File tools do not expand shell variables. Do not pass paths containing `$OBSIDIAN_VAULT_PATH` to `read_file`, `write_file`, `patch`, or `search_files`; resolve the vault path first and pass a concrete absolute path. Vault paths may contain spaces, which is another reason to prefer file tools over shell commands.

If the vault path is unknown, `terminal` is acceptable for resolving `OBSIDIAN_VAULT_PATH` or checking whether the fallback path exists. Once the path is known, switch back to file tools.

## OneBrain vault (PARA)

This vault uses the **OneBrain** PARA layout (`00-inbox` … `07-logs`) and a four-tier memory model. Before organizing, capturing at scale, or synthesizing, load the conventions once:

`skill_view("obsidian", "references/onebrain-conventions.md")` — folder map, naming, frontmatter, `## Related` wikilinks, task syntax, memory tiers, and the companion skills (`consolidate`, `distill`, `daily`, `wrapup`).

**Quick capture:** write the raw thought to `00-inbox/YYYY-MM-DD-topic.md` with minimal frontmatter, `search_files` for 1–3 related notes, and add a `## Related` section. Leave deeper processing to the `consolidate` skill.

## Read a note

Use `read_file` with the resolved absolute path to the note. Prefer this over `cat` because it provides line numbers and pagination.

## List notes

Use `search_files` with `target: "files"` and the resolved vault path. Prefer this over `find` or `ls`.

- To list all markdown notes, use `pattern: "*.md"` under the vault path.
- To list a subfolder, search under that subfolder's absolute path.

## Search

Use `search_files` for both filename and content searches. Prefer this over `grep`, `find`, or `ls`.

- For filenames, use `search_files` with `target: "files"` and a filename `pattern`.
- For note contents, use `search_files` with `target: "content"`, the content regex as `pattern`, and `file_glob: "*.md"` when you want to restrict matches to markdown notes.

## Create a note

Use `write_file` with the resolved absolute path and the full markdown content. Prefer this over shell heredocs or `echo` because it avoids shell quoting issues and returns structured results.

## Append to a note

Prefer a native file-tool workflow when it is not awkward:

- Read the target note with `read_file`.
- Use `patch` for an anchored append when there is stable context, such as adding a section after an existing heading or appending before a known trailing block.
- Use `write_file` when rewriting the whole note is clearer than constructing a fragile patch.

For an anchored append with `patch`, replace the anchor with the anchor plus the new content.

For a simple append with no stable context, `terminal` is acceptable if it is the clearest safe option.

## Targeted edits

Use `patch` for focused note changes when the current content gives you stable context. Prefer this over shell text rewriting.

## Wikilinks

Obsidian links notes with `[[Note Name]]` syntax. When creating notes, use these to link related content.

## Meeting Summary Template

The vault includes a reusable **Meeting Summary Template** at `Templates/Meeting Summary Template.md`, designed for post-meeting extraction by the agent.

### Template structure
| Section | Purpose |
|---------|---------|
| **Metadata table** | Date, time, participants, related notes |
| **Agenda** | Checkbox list of planned topics |
| **Notes & Discussion Points** | Free-form raw notes |
| **Decisions Made** | Bullet list of what was decided |
| **Action Items table** | Action item, **Responsible**, **ETA**, Status (Pending/Done/Cancelled) |
| **Follow-Ups table** | Follow-Up, **Responsible**, **ETA**, Status |
| **Attachments** | Links to files or references |
| **Next Meeting** | Proposed date, agenda, required attendees |
| **Tag Index** | `#meeting`, company, topic, date tags |

### How to use
1. For a new meeting: copy the template to `Meetings/<Company> - <Meeting Topic>.md`, then fill in the {{placeholders}}.
2. After the meeting: give the agent raw notes. The agent will:
   - Extract **Action Items** into the table, assigning each a **Responsible** and **ETA**.
   - Extract **Follow-Ups** into the table, assigning each a **Responsible** and **ETA**.
   - Flag any missing owner or deadline.
   - Create a clean summary note in the vault.

### Pre-filled instance
A pre-filled meeting instance for the **Argu AI architecture meeting** already exists at `Meetings/Argu AI - Architecture Meeting.md` with agenda items pre-populated.

## Setup workflow — step‑by‑step

When a user wants to start using Obsidian with Hermes, follow these steps and get explicit approval for each state‑changing action:

1. **Load the skill** — `skill_view(name='obsidian')`.  The skill is already part of Hermes; no plugin installation is required.
2. **Resolve the vault path** — run `terminal` to check `OBSIDIAN_VAULT_PATH` or fall back to `~/Documents/Obsidian Vault`.
3. **Ensure the vault directory exists** — if missing, create it with `mkdir -p <resolved_path>` (approval required before running).
4. **Optionally harden permissions** — `chmod 700 <resolved_path>` (approval required).
5. **Verify with a read‑only list** — `search_files` with `target: "files"` under the vault path.
6. **Test a write** — after approval, create a trivial test note with `write_file` so the user can confirm the bridge works.

A helper script `scripts/init-obsidian-vault.sh` is provided within the skill to automate steps 2‑4 (create vault, .obsidian config, folders, welcome note). Review the script before running.

### Pitfalls

- **Do NOT** run `hermes plugin install obsidian`.  The command is invalid (`'plugin'` is not a valid Hermes subcommand) and `obsidian` is a **skill**, not a plugin — it has no CLI install step.  `obsidian` does not appear in `hermes plugins list` because it is a skill, not a plugin.
- **Do NOT** pass unexpanded shell variables to file tools — always resolve the path first.
- Always get explicit approval before writing into the vault.
- If the vault path has spaces, never wrap it in extra quotes when using file tools; use the plain unquoted path.
