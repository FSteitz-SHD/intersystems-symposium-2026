---
name: erp-review-loop
description: The only mutating review verb for the ERP. Runs /erp-review, classifies findings (mechanical-fix / needs-test / needs-design / noise), auto-applies the mechanical fixes only, re-runs the scoped module tests, and iterates to a clean pass or a safety cap. Never commits, never pushes. Use when the user types /erp-review-loop or asks to fix-and-recheck a diff.
argument-hint: "[scope] [--max-iters N] [--dry-run] [--only lens,…] [--skip lens,…]"
---

<!-- Excerpt, slightly shortened. -->

# erp-review-loop — fix, re-check, repeat

Wraps `/erp-review` in a fix-and-recheck loop. It **mutates the working tree** (mechanical findings only) and re-runs tests each round — but **never commits and never pushes**. The user commits separately after reviewing the tree. This is the one verb in the family that edits; everything else is report-only.

## Guardrails (read first)
- **Only mechanical fixes are auto-applied.** Anything touching behavior, tests, or design is **reported, not applied**.
- **Cap:** default `--max-iters 5`. Stop earlier on a clean pass or when a round applies nothing.
- **`--dry-run`:** classify only, apply nothing.

## Loop (iteration `n` in `1..MAX`)
1. **Pre-state.** Capture the touched-file set + the per-module test command. Run the tests **once before** any fix this round so a regression from a previous round is detectable.
2. **Review.** Run `/erp-review <scope> --lens auto`. The report is the ground truth of "what is still wrong".
3. **Classify** each finding into exactly one bucket:

   | Bucket | Examples | Action |
   |---|---|---|
   | **mechanical-fix** | `+ e.getMessage()` in a log → drop, pass `e` last; missing `e` arg on `LOG.error/warn` in a catch; `@Column` without `name=`; hardcoded UI string → `getText(key)` + bundle entries; wildcard class import; missing `final` on a param; stale `// TODO` without ticket | **auto-apply** |
   | **needs-test** | a fix that changes behavior and must be covered; a missing test branch; `assertTrue(true)` | **report** (don't write tests blind) |
   | **needs-design** | layer-boundary break, aggregate reshape, RPC shape change, fail-open redesign | **report** (escalate to a plan) |
   | **noise** | false positive, out-of-scope/pre-existing issue, style preference with no rule | **drop** (note it) |

4. **Apply mechanical fixes only** (skip if `--dry-run`), severity Blocker → Major → Minor. Smallest correct edit per finding. **Stop at the first Blocker that is not mechanical-fix** — surface it and ask the user. Anything not clearly mechanical → treat as needs-design/needs-test; a wrong mechanical fix is worse than a deferred one.
5. **Re-run the scoped tests** (touched modules only). If a fix regressed a test, **revert that fix** and re-classify it as needs-test/needs-design.
6. **Stop conditions.** Clean pass (no Blocker/Major) → success · only needs-* / noise remain → partial (hand back the open items) · same finding survives two rounds → stuck · tests regressed and the revert didn't restore green → stop, show the failing test · `--max-iters` hit → cap.

## Per-iteration output (one screen — don't reprint the full report)
```
### Iteration N
Lenses: dev, database, docs
Findings: 0 blocker / 3 major / 5 minor
Applied (4): <file:line — one-liner> ×4
Deferred (2): <needs-design / needs-test, with reason>
Tests: `mvn -T 4 -pl … test` → 40/40 green
```

## MUST / MUST NOT
- **MUST** run tests after every round's fixes; honour the severity/rule the reviewer cited; quote `file:line` for every applied fix; show `git status` / a diff stat at the end.
- **MUST NOT** commit or push; auto-fix needs-design/needs-test; suppress findings to force green; broaden scope mid-loop.
- If the very first review is clean, say so and stop — nothing to loop.
