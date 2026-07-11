# PLAN — Hermes skill consolidation

## Feature description & goal

The Hermes install on `hermes-box` has **80 enabled skills**. A review surfaced
several overlapping/duplicate skills. Goal: **remove genuine duplication and
cross-link adjacent skills**, so skill triggering is unambiguous and each skill
earns its place — applying the same lens we used to unify `freellmapi`
("only keep what the agent doesn't already know; don't ship redundant skills").

Success = fewer overlapping skills, no lost capability, each surviving skill has
a clear, non-competing trigger `description`, and `hermes skills list` reflects
the new set.

## Files / sources explored

- `hermes skills list` (console) — authoritative enabled set + source (local/builtin).
- `~/.hermes/skills/**/SKILL.md` — local (user-editable) skills.
- `~/.hermes/hermes-agent/skills/**/SKILL.md` — builtin (packaged) skills.
- Scan of all `SKILL.md` frontmatter descriptions (name + description per skill).

## Existing-design review (matters for how we edit)

Hermes resolves skills from two roots: **local** (`~/.hermes/skills/`, fully
editable, survives updates) and **builtin** (shipped in the `hermes-agent`
package). Editing a builtin in place is risky — `hermes update` can overwrite it
— though Hermes tracks user edits (`hermes skills list-modified` / `diff` /
`reset`) and *keeps* modified bundled skills on update. Safest changes touch
**local** skills only.

Edit-safety of the merge targets (verified via `skills list`):

| Skill | Source | Safe to edit? |
|---|---|---|
| `firecrawl-perplexity-scrapper-choice` | local | ✅ |
| `firecrawl_vs_perplexity` | local | ✅ |
| `scrapper-tool-wrapper` | local | ✅ |
| `writing-plans` | local | ✅ |
| `plan` | builtin | ⚠️ overwrite risk |
| `spike` | builtin | ⚠️ |
| `claude-code` / `codex` / `opencode` | builtin | ⚠️ |

**Repo vs in-place decision (open question below):** the `myskills` git repo is
the authoring source but currently only holds `freellmapi` + the OneBrain
note-taking skills. The consolidation targets are *not* in `myskills` today. We
can either (a) edit them in place on `hermes-box` (fast, but untracked), or
(b) bring the consolidated survivors into `myskills` and deploy (tracked, but
expands the repo's scope).

## Consolidation plan — phased by risk

### Phase 1 — Merge the two web-retrieval decision guides (both local, safest) — ✅ DONE (2026-07-11)

**Outcome:** merged into `web-retrieval-choice` (category `research`), authored in
the `myskills` repo and deployed to `~/.hermes/skills/research/`. Both originals
(`firecrawl-perplexity-scrapper-choice`, `firecrawl_vs_perplexity`) deleted.
Union of content kept (all 3 tools + checks + Perplexity intent/quota); generic
scraping-vs-search prose dropped; stale `freellmapi-best-practices` reference
fixed to `freellmapi`. Verified: `hermes skills list` → 80 → 79 enabled, 0
disabled; `scrapper-tool-wrapper` retained as the executor and cross-linked.

Original plan below (for reference):


`firecrawl-perplexity-scrapper-choice` (root) and `firecrawl_vs_perplexity`
(mlops) are the same skill: "when to use Firecrawl vs Perplexity (vs
scrapper-tool)." Two names, two categories, one job — a real triggering hazard.

- **Action:** merge into **one** local skill (proposed name
  `web-retrieval-choice`, category `research` or `mlops`). Keep the superset of
  guidance; cross-link `scrapper-tool-wrapper` (the executor) and
  `anti-bot-extraction` (the fallback playbook). Delete the two originals.
- **Why safe:** both are local; no builtin risk.
- **Capability preserved:** yes — union of both bodies.

### Phase 2 — Merge `plan` + `writing-plans` (mixed source)

Both "write an actionable markdown plan, bite-sized tasks, exact paths, no
execution." `spike` (throwaway validation) is adjacent, not a dup — keep, but
cross-link.

- **Action:** keep **one** planning skill. Preference: keep the **local**
  `writing-plans` as the survivor (editable, update-safe), fold in anything
  unique from builtin `plan`, then **disable** builtin `plan`
  (`hermes skills config` / opt-out) rather than deleting the package file.
- **Risk:** `plan` is builtin — don't hand-delete package files; disable instead.
- **Open question:** is `plan` wired to a "plan mode" elsewhere (slash command /
  mode) that expects that exact skill name? Verify before disabling.

### Phase 3 — Rationalize the coding-delegation trio (all builtin)

`claude-code`, `codex`, `opencode` are three near-identical "delegate coding to
CLI X" skills; also overlap `subagent-driven-development`.

- **Action (conservative):** leave the three as-is but confirm their
  `description`s are differentiated enough to trigger on the *right* CLI; add a
  one-line cross-reference to `subagent-driven-development`. Optionally group
  them into a **bundle** (`hermes bundles`) instead of merging.
- **Why not merge:** they're builtin (edit risk) and separate triggering per CLI
  may be intentional. Merging is higher-effort, lower-value than Phases 1–2.

### Phase 4 — Housekeeping (low effort)

- `dogfood` is enabled under a category literally named `._disabled` — likely
  accidental. Recategorize or disable.
- **Local shadow copies:** several `mlops`/`creative` builtin skills also exist
  as local copies in `~/.hermes/skills/`, which can drift from upstream. Audit
  and remove the redundant local shadows (or intentionally keep + track).
- Cross-link (no merge) the adjacent clusters: debugging
  (`systematic-debugging` as entry point → `python-debugpy` /
  `node-inspect-debugger` / `debugging-hermes-tui-commands`); code-review
  (`requesting-code-review` / `github-code-review` / `simplify-code`);
  design (`claude-design` / `sketch`).

## Open questions (please decide)

1. **Scope for this pass:** just Phase 1, or Phases 1–2, or all four?
2. **Repo vs in-place:** bring survivors into the `myskills` repo (tracked +
   deploy), or edit directly on `hermes-box`?
3. **Coding trio (Phase 3):** leave separate + cross-link (recommended), bundle,
   or actually merge into one?
4. **Deletion vs disable:** OK to hard-delete *local* duplicates, and to
   *disable* (not delete) *builtin* ones?

## Tests / verification

- After each phase: `hermes skills list` shows the intended set (survivors
  present, duplicates gone/disabled), still `0 disabled` conflicts, total count
  drops by the expected number.
- No surviving pair shares an overlapping trigger `description`.
- Spot-check triggering: a scraper-choice prompt resolves to the single merged
  skill; a "write a plan" prompt resolves to the single planning skill.
- Capability check: the merged skill's body contains the union of the originals'
  unique guidance (no dropped content).
