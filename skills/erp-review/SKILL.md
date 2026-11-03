---
name: erp-review
description: Multi-perspective code review for the ERP codebase against the project's own rules (AGENTS.md + .agents/rules). Resolves the scope, auto-selects review lenses from the touched files, fans the read-only erp-* reviewer agents out in parallel, and returns one deduplicated, verified, advise-don't-block report. Report-only. Use when the user types /erp-review or asks for a code review, diff review, "schau drüber", or a review before a commit or MR.
argument-hint: "[scope] [--lens auto|dev|tester|architect|database|docs|silent-failure|performance|security|type-design|all] [--engine claude|gpt|both]  (bare = guided menu)"
---

<!-- Excerpt, shortened. The cross-vendor runtime details (CLI resolution, review tree, cleanup) are condensed to one paragraph. -->

# erp-review — the start-here review

Thin dispatcher over the shared review contract in `_review-shared/` and the read-only `erp-*-reviewer` agents. Three ways to drive it — a guided **menu**, **flags**, or a one-line **prompt** — all land in the same pipeline. This verb is **report-only**: it never edits, commits, or pushes.

## 0. Parse inputs
From `$ARGUMENTS`:
- a leading non-`--` token = the **scope** (a git ref / `sha` / path, or one of `working|staged|commit|branch|range|mr|files`);
- `--lens <csv>` (default `auto`), `--engine claude|gpt|both` (default `claude`).

**If `$ARGUMENTS` is empty and you are interactive,** ask once (three questions): **Scope** (open MR / staged / working tree / branch vs base), **Lenses** (auto by files / pick), **Engine** (Claude / GPT cross-vendor / both).

**Degrade:** headless or running as a subagent → skip the menu and use the flags or the defaults. A natural-language prompt ("review my DB changes") is also valid — infer scope + lenses from it.

## 1. Resolve scope
Establish a concrete `<base>` + diff and **state it in one line**. No MR and no explicit base → STOP and ask (never guess the base branch). Capture the changed-file list. Empty diff → say so and stop.

## 2. Resolve lenses
`--lens auto` (default): read `_review-shared/lens-map.md` and pick the lens set from the touched paths. Explicit `--lens` overrides auto. Map each lens to its agent and model per the lens map.

## 3. Run the engine

**`--engine claude` (default).** Spawn the selected agent(s), passing the model from the lens map explicitly. One lens = the fast path; **≥2 lenses = parallel fan-out — issue all agent calls in a single message.** Prompt each:
> Review `git diff <base>...<head>` (narrow to your files with `-- <path>`). Read the touched files **at the reviewed revision** — do not assume the working directory holds `<head>`. Read `_review-shared/severity-and-format.md` and report in that one-line format with a final `Verdict:` line. Read-only; confirm each behavioral finding against the source line before reporting.

**`--engine gpt`.** Run the cross-vendor reviewer (Codex CLI) read-only on a checked-out copy of the reviewed revision. **Tell the user the diff leaves the machine to an external model.** Always clean up the temporary review tree, even on error.

**`--engine both`** → delegate to `/erp-deep-review <scope>` (both engines, blind to each other) and stop here.

## 4. Synthesize + verify
Collect the findings and produce ONE report per `_review-shared/severity-and-format.md`:
- **Deduplicate** by `file:line` (same defect from two lenses = one finding; keep the highest severity, merge the `[rule:]` tags).
- **Verify before presenting:** drop any behavioral finding without a `file:line` citation you can tie to the diff; downgrade the unprovable.
- **Tag pre-existing** issues `[PRE-EXISTING]` — report but do not count them in the verdict.
- End with one `Verdict: N blocker / M major / K minor (+P pre-existing)` and a one-sentence recommendation.

## 5. Hand off
Relay the report. **Advise, don't act** — do not edit, commit, or push. To auto-apply mechanical fixes use `/erp-review-loop`; for a dual-model blind-spot pass use `/erp-deep-review`.
