---
name: consolidate
description: Process the OneBrain inbox — markdown captures and staged binary files — into permanent PARA notes
version: 1.2.0
metadata:
  hermes:
    category: note-taking
    tags: [obsidian, para, knowledge-management, import]
---

# Consolidate (OneBrain inbox → knowledge)

Turn everything waiting in `00-inbox/` into well-formed permanent notes filed under the
right PARA folder, moving each consumed original out of the inbox. Two kinds of item
arrive: markdown captures, and binary files staged in `00-inbox/imports/`. Handle the
binaries first, so their notes already exist when phase 2 goes looking for link targets.
Uses Hermes file tools plus a few read-only extractors. Safe to run unattended (the
every-2h cron): it never deletes — it moves.

## When to Use

The user says "process my inbox", "consolidate", "clean up inbox", "import my files", or
the scheduled cron fires. Load `skill_view("obsidian", "references/onebrain-conventions.md")`
first if the PARA/naming conventions aren't already in context.

Resolve the vault path before either phase (see the `obsidian` skill). Run both phases in
order. If neither finds anything, say so and stop.

## Phase 1 — Staged binaries (`00-inbox/imports/`)

1. `search_files` (`target: "files"`) under `00-inbox/imports/`. Ignore `.gitkeep` and
   anything under `imports/failed/`. If nothing is staged, go straight to phase 2.
2. Extract each file's content using the **Extraction map** below. An extractor that
   errors — or returns empty output for a non-empty file — means extraction failed: apply
   **Failure handling** and continue to the next file. Never describe a file you could not
   actually read; guessing from the filename is fabrication.
3. Pick a **topic** subfolder under `04-resources/`. `search_files` to see which kebab-case
   topic folders already exist and reuse one when it fits; create a new one only when
   nothing matches. Imported material is external reference, so it belongs in
   `04-resources/`, never `03-knowledge/`.
   Name the folder for the **subject matter, never the file format**: a CUPS diagram goes
   in `printing/`, not `images/`; a pricing spreadsheet goes in `finance/`, not `office/`.
   `pdf/`, `images/`, `scripts/` and `office/` are not valid topic folders — if you are
   about to use one, you have picked the file type by mistake.
   **Never write a note inside `04-resources/attachments/`** — that tree holds binaries
   only, and a note placed there is misfiled.
4. `write_file` the note at `04-resources/<topic>/<Title>.md` — titled from the document's
   own title where it has one, otherwise from the filename:

   ```
   ---
   tags: [import, <type>]
   created: YYYY-MM-DD
   source: /import
   file_type: <pdf|office|image|svg|video|script>
   ---

   # <Title>

   ![[<original filename>]]

   ## Summary

   Two or three sentences on what this document is and why it matters.

   ## Key Points

   Three to seven bullets carrying the substance. For scripts, add a `## Code`
   section with the full source in a fenced block. For spreadsheets, render each
   sheet as its own markdown table.

   ## Related

   [[existing note]]
   ```

   If that path already exists, append ` (Imported YYYY-MM-DD)` to the filename.
5. Move the original binary to `04-resources/attachments/<type>/<filename>`, where `<type>`
   is `pdf`, `office`, `images`, `video`, or `scripts`. Keeping it in the vault is what
   makes the `![[...]]` embed resolve — Obsidian finds attachments by filename anywhere in
   the vault. **Then confirm the move actually happened** by re-checking that
   `00-inbox/imports/<filename>` is gone. A copy that leaves the original behind gets
   re-imported on every subsequent run.
6. Add 1–3 `[[wikilinks]]` under `## Related`, subject to the **Linking rule** below.

### Extraction map

| Extension | How to extract |
|---|---|
| `.pdf` | terminal: `pdftotext -layout "<file>" -` |
| `.docx` `.xlsx` `.xls` `.pptx` `.ppt` | terminal: `~/.hermes/bin/uvx --from markitdown markitdown "<file>"` |
| `.png` `.jpg` `.jpeg` `.gif` `.webp` | **`vision_analyze`**, passing `image_url` as the file's absolute path. This is the only way to see an image: `read_file` cannot, and `file`/`identify` give dimensions, not content. If `vision_analyze` is unavailable or fails, the extraction has failed — send the image to `failed/` rather than describing it from its filename |
| `.svg` | `read_file` as text; describe what it renders |
| `.mp4` `.mov` `.webm` `.mkv` | terminal: `ffprobe -v quiet -print_format json -show_format "<file>"` |
| `.py` `.sh` `.bash` `.zsh` `.sql` | `read_file` |

Any other extension is unsupported: leave it in `00-inbox/imports/` untouched and list it
in the summary.

### Failure handling

Never delete, and never leave a broken file where the next run will retry it forever:

- Extraction failed, returned empty, or the file is 0 bytes → move it to
  `00-inbox/imports/failed/` and report the reason in the run summary. **Write no note at
  all** — not even a stub describing the failure. A note whose Summary says the file could
  not be read is pure speculation about content nobody has seen, and it pollutes
  `04-resources/`. The summary line and the file sitting in `failed/` are the record.
- A non-empty PDF or Office file that extracts to nothing is usually password-protected —
  say so in the summary so the user can unlock and re-stage it.
- If the note was written but the move in step 5 failed, report it as a partial success.
  The note is correct; the original just needs moving by hand.

## Phase 2 — Markdown captures (`00-inbox/`)

1. `search_files` (`target: "files"`, `pattern: "*.md"`) recursively under `00-inbox/` — the
   inbox has subfolders (e.g. `AI/`, `MNG/`, `1 1/`), so process them all. Skip
   `00-inbox/imports/`; phase 1 already handled it. If there is nothing to do, say so.
2. For each item: `read_file` it. **Skip any note whose frontmatter has `processed: true`.**
   Items with no frontmatter, or with `processed: false`, are unprocessed — process them.
3. Pick the destination — `01-projects/` (project material), `02-areas/` (ongoing
   responsibility), `03-knowledge/` (the user's own insight), or `04-resources/` (external
   reference). For **meeting notes** specifically: a matching active project folder wins
   (e.g. `01-projects/argu.ai/`); otherwise 1:1s go to `02-areas/people/[first-name]/` and
   team/management/planning meetings go to `02-areas/management/`. Meeting notes do not
   belong in `03-knowledge/` — that is reserved for distilled insight.
4. `search_files` for related notes and collect 1–3 `[[wikilinks]]`, subject to the
   **Linking rule** below.
5. Compose the permanent note — frontmatter (`tags`, `created`), cleaned body (preserve the
   original language; do not translate), and a `## Related` section — and `write_file` it at
   `[dest]/Topic Name.md`. Merge into an existing note when a strong fit exists (append via
   `patch` rather than creating a duplicate).
6. **Retire the original — never delete it:**
   - If the item became its own note, it is already relocated; nothing more to do.
   - If its content was merged into an existing note, **move** the original to
     `07-logs/captures/YYYY/MM/` (create the folder if missing). This keeps `00-inbox/`
     empty without destroying the raw capture.

## Linking rule (both phases)

Every `[[wikilink]]` must be earned by a search hit. Before adding one, `search_files` for
that exact note name; add the link only if the search returned a real file, and only using
that file's actual name. If the search returned nothing, do not add the link — an invented
target is an orphan, and a plausible-sounding title such as `[[cups]]` or
`[[printing-system]]` is not evidence a note exists. Never link to an item still sitting in
`00-inbox/`: it has no permanent note yet, so guessing its future filename creates an orphan
too. When nothing verifiable fits, write `_No related notes found._` — most imports have no
related notes, and that is the expected outcome, not a failure.

## Summary

Close every run by reporting what moved where: each source → its destination, the topic
subfolder chosen for imports, links added, and any files sent to `imports/failed/` with the
reason. Report what you verified, not what you intended.

## Pitfalls

- Preserve the source language — transcripts may be non-English; carry them verbatim.
- Never invent facts — carry only what the capture or file actually says. Hedging words
  ("appears to be", "likely shows") are a sign the content was never read: extract it
  properly or send it to `failed/`.
- No orphan links — every `[[wikilink]]` must resolve to a note that exists *now*.
- Never delete anything — move it. Git history in `kv-brain` is the deeper safety net.
- `ffprobe` yields metadata only (duration, codecs, resolution), not speech. A video note
  describes the file; it does not summarize its content.
- `.xls` (legacy Excel) often extracts garbled. Treat garbled output as a failure rather
  than writing a nonsense note.
- Idempotent by construction: processed items leave `00-inbox/`, so a re-run only sees new
  arrivals.

## Verification

`00-inbox/` holds only unprocessed items — ideally empty, aside from `imports/.gitkeep`,
`imports/failed/`, and any unsupported staged file. Each processed item exists as a
permanent note under the correct PARA folder with frontmatter and links; each consumed
markdown original is either promoted into PARA or sitting in `07-logs/captures/`; and each
imported binary sits in `04-resources/attachments/<type>/` with its note in a topic folder
alongside, not in the attachments tree.
