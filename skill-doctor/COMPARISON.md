# `skill-doctor` vs. Claude's skill-review tooling

Notes on how the vendored Warp [`skill-doctor`](./SKILL.md) differs from what
Anthropic ships for reviewing skills: the **`skill-creator`** eval/iterate loop
(bundled with Claude Code and Claude.ai, including its `agents/grader.md`,
`agents/comparator.md`, `agents/analyzer.md` sub-agents) and the
**`claude plugin eval`** CLI.

## The one-line difference

- **`skill-doctor` is retrospective and observational.** It reads conversations
  that already happened on your machine and asks *"was my agent setup pulling
  its weight in real work, and where is it leaking?"*
- **`skill-creator` / `claude plugin eval` are prospective and experimental.**
  They run *new* tasks against a skill, with a control arm, and ask *"does this
  skill — or this edit to it — measurably improve output?"*

One grades the past; the other runs experiments on the future. They answer
different questions and they don't overlap much.

## Side by side

| | Warp `skill-doctor` | Anthropic `skill-creator` | `claude plugin eval` |
|---|---|---|---|
| **Input** | Real local session history: Warp conversation DBs, Claude Code project JSONL, Codex rollout JSONL | Synthetic test prompts you author into `evals/evals.json` | Eval cases: `case.yaml` or `prompt.md` + `graders/*.md` |
| **Unit judged** | A whole agent session (behaviour + artifact) | One skill's output on one task | One case run against a plugin |
| **Control arm** | **None** — no counterfactual run exists | Yes: no-skill baseline, or the old skill snapshot | Yes: `--ablation with-without` no-plugin arm |
| **Cost per run** | Cheap: no agent executions, only transcript scoring | Expensive: 2 full agent runs per case per iteration | Bounded (`--max-cost-usd`, `--judge-model`, default haiku) |
| **Scoring** | Two LLM rubrics — Efficiency (0.2–1) and Code Quality (binary `approve`/`block`, `insufficient_evidence` excluded) — plus `skill_coverage` | Per-assertion pass/fail by the grader agent, plus tokens and wall-clock, mean ± stddev vs. baseline | Per-grader scores and the score **delta** vs. baseline |
| **Aggregate** | `overall = 0.5·efficiency + 0.35·code_quality + 0.15·skill_coverage`, curved `0.5 + 0.5·raw`, rendered as a letter grade | `benchmark.json` / `benchmark.md` pass rates + an analyst pass for non-discriminating assertions and high-variance evals | JSON run result |
| **Human in the loop** | Two questions at startup, then it reports | Central: an HTML viewer where the user reviews each output and types feedback that drives the next rewrite | Optional; graders are the judge |
| **Output** | One self-contained `report.html` (scorecard, 3 findings, diffs) + proposed `SKILL.md` files under `proposed/` — never edits real skills | An actually-improved `SKILL.md`, applied by the agent, plus benchmark artifacts | Scored results (`--json`) |
| **Triggering / description** | Infers it: an installed skill that never fired in a failed conversation is flagged as a description problem | Dedicated optimizer — 20 hand-reviewed trigger queries, 60/40 train/held-out split, 3 samples/query, ≤5 iterations, picks best by *test* score | `tool_used: Skill` as a with-only "the plugin fired" indicator |
| **Harnesses** | Warp, Claude Code, Codex — hard-gated at startup, refuses anything else | Claude | Claude Code plugins |
| **Privacy stance** | Explicit: everything local, never upload transcripts or excerpts | Runs are local, but the tasks are synthetic, so no history is read at all | Local |

## What `skill-doctor` has that Claude's tooling doesn't

1. **It works from real usage, so it needs no test set.** The hardest part of
   `skill-creator` is writing test prompts that represent real work.
   `skill-doctor` skips that entirely — your last 45 days *are* the test set,
   with all the messy context that synthetic prompts never capture.
2. **It scores the sessions, not just the outputs.** The Efficiency rubric
   grades cost-to-reach-result: rework cycles, repeated user corrections,
   redundant reads, serial instead of batched calls, flailing, late
   verification. Nothing in `skill-creator` measures how much of *your*
   attention the agent burned; `claude plugin eval` reports tokens and time but
   doesn't reason about waste.
3. **It finds the skills you should have written.** Because it starts from
   failures rather than from a skill, "no skill exists for this recurring
   waste" is a first-class finding. Claude's tools all start with a skill you
   already have.
4. **It has a discipline for *not* filing changes.**
   [`references/skill-improvements.md`](./references/skill-improvements.md) is
   the most reusable part of the skill: verify each finding against the repo,
   prefer replacing guidance over appending, require that the rule *would have*
   prevented the failure, require recurrence across runs, and — "when nothing
   clears this bar, open no change and say why not — that is a success. A
   speculative change is worse than none." That bar is stated much more sharply
   than anything in `skill-creator`.
5. **Cross-harness.** It reads Codex and Warp history too, not just Claude's.

## What Claude's tooling has that `skill-doctor` doesn't

1. **A control arm.** This is the big one. `skill-doctor` never runs the same
   task without the skill, so it can't attribute an outcome to a skill — only
   correlate. `skill-creator` and `claude plugin eval` both run an explicit
   baseline and report the delta, which is the only way to know an edit helped.
2. **Repeat sampling and variance.** LLM runs are noisy; `skill-creator`
   reports mean ± stddev, flags high-variance (flaky) evals and assertions that
   pass regardless of the skill. `skill-doctor` scores each transcript once and
   averages — no variance signal, and a single-sample LLM rubric is itself a
   noisy instrument.
3. **Measured triggering.** `skill-doctor` *guesses* that a skill didn't fire
   because of its description; `skill-creator`'s optimizer measures trigger rate
   against held-out queries and iterates the description until it improves.
4. **Grading the eval, not just the run.** `agents/grader.md` explicitly
   critiques weak assertions ("a passing grade on a weak assertion is worse than
   useless") and extracts and verifies implicit claims in the output.
   `skill-doctor` has no equivalent self-check on its own rubrics.
5. **A closed loop.** `skill-creator` applies the edit, re-runs, and compares
   against the previous iteration. `skill-doctor` stops at a diff in
   `proposed/` and asks whether you'd like it applied — there's no
   verification that the edit actually helped.

## Things to know before trusting the grade

- **The curve is generous.** `curve(x) = 0.5 + 0.5·x` maps the worst possible
  efficiency (0.2) to 0.6 and a `block` on code quality to 0.6. A genuinely bad
  set of sessions still reports a passing-looking number; read the three
  findings, not the letter.
- **`skill_coverage` is 15% of the grade and rewards *firing* skills, not
  useful ones.** Installing more skills that trigger raises the score whether or
  not they helped. It's a Warp-flavoured incentive baked into the metric.
- **Code quality is binary.** One defect a reviewer would block on scores 0.2,
  the same as a disastrous change. Sessions with no diff score
  `insufficient_evidence` and drop out, so a doc-heavy month grades almost
  entirely on efficiency.
- **The report is Warp-branded.** `report.json` carries a `cta_url` and
  `SKILL.md` requires ending every response with a "Request access to Warp
  Factories" line; `render_report.py` stamps `warp.dev/skill-doctor` onto the
  share image. Harmless, but it's marketing in the output — strip it if this
  ever runs for anyone but you.
- **No repeat runs.** Scores come from one LLM pass per transcript, so
  re-running the same window can produce a different grade.

## How I'd actually use the two together

`skill-doctor` is the diagnostic; `skill-creator` is the treatment and the
follow-up.

1. Run `skill-doctor` over the last month to find where real work leaked
   (that's the part synthetic evals can't tell you).
2. Take its `proposed/<skill>/SKILL.md` diff as a *hypothesis*, not a patch.
3. Hand that hypothesis to `skill-creator` (or `claude plugin eval`), which
   supplies the missing control arm: run the failing scenario with the old skill
   and the new one and see if the delta is real.
4. If the finding was "this skill never fired", run `skill-creator`'s
   description optimizer instead of hand-editing the trigger text.

The gap `skill-doctor` fills is that nothing on the Claude side looks at what
actually happened in your terminal. The gap it leaves is that it never proves
its own suggestions work.
