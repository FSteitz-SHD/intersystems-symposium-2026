# Severity, finding format, and report standard

<!-- Excerpt, shortened. Ticket references in the "failed in practice" notes are removed. -->

Every reviewer — the `erp-*-reviewer` agents and the cross-vendor (GPT) reviewer — reports against this scheme so findings are comparable and the orchestrator can deduplicate and synthesize them.

## Severity (3 levels + one tag)

**[BLOCKER]** — broken build, data corruption, or wrong runtime behavior.

**[MAJOR]** — real defects that are not build/data/behavior breakers: material rule violations — layer-boundary break, persistence-rule violation, fail-open in a demand/eligibility path, missing i18n on a user-facing string, an introduced N+1, a missing test for risky logic.

**[MINOR]** — naming, dead code, style, comment quality.

**[PRE-EXISTING]** — a defect already present in the touched files but **not** introduced by the change under review. Report it so it is not lost, but **do not count it in the verdict** and never treat it as a merge gate for this change.

## Verify before you flag
- Every behavioral finding needs a `file:line` citation pointing at the code that exhibits it — not an inference from a name. Open the file and confirm the line before reporting.
- If you cannot tie a finding to a quoted rule or to code you have read at a specific line, drop it or downgrade to Minor. A plausible-but-unverified finding costs the reader a wasted round trip — worse than a missed nit.
- **The fix needs the same proof as the finding.** Before proposing a change, name the in-repo precedent that shows it works, or mark the finding `fix unverified`. The costliest review errors are not wrong defects; they are real defects with a fix that breaks something else.
- **Framework semantics beat generic advice.** Before proposing a change to visibility, an annotation, a lifecycle hook or a package, state the framework constraint that governs it. A `@Transactional` method narrowed below `public` silently loses its boundary.
- **Judge the justification, not just the code.** Where a comment explains why something is safe, check that explanation against the code. Author-supplied reasoning is a claim under review.

### Verify the premise, not just the location
A correct `file:line` does not make a finding correct — the *reason* has to hold too. These premises have actually failed in practice:

- **Consistency arguments** ("set here but not there"). Establish that both sites carry the **same requirement**. Where one is load-bearing and the other cosmetic, there is no defect.
- **Hardening findings** ("this could throw on bad input"). Establish that the failure path is **reachable**. And weigh the cure: replacing a loud failure with a silent default is fail-open.
- **Rule citations.** Quote the clause that is violated, not the rule's topic. If the codebase overwhelmingly does it the other way (check the ratio), the finding is a repo-wide proposal, not a defect in this diff.

Three findings that passed the `file:line` gate but failed here were applied and then reverted by the author — the cost of a wrong finding is highest once someone acts on it.

## Finding line format
Group by severity, highest first. One line per finding:

```
- [BLOCKER] path/File.java:123 — <what is wrong / why it matters> — fix: <concrete replacement> [rule: 141-persistence-layer]
```

- **Cite the rule.** `[rule: <id>]` points at the `.agents/rules/*.md` that is violated — not generic Java advice. When project rules conflict with generic best practice, the project rule wins.
- **Be specific.** "Improve readability" without a concrete change is not a finding.
- **Mark unverified fixes.** `— fix (unverified): …` and name the open question.
- **Verdict line** ends every reviewer's output: `Verdict: N blocker / M major / K minor (+P pre-existing)` — or `Verdict: no findings`.

## Synthesis rules (orchestrator)
- **Deduplicate across reviewers.** Same `file:line` from two reviewers = one finding; keep the highest severity, merge the evidence and the `[rule:]` tags.
- **A rule status is `ok` only with quoted evidence.** No evidence → `warn`, never `ok`.
- **Report tests exactly as recorded.** Never claim "tests pass" without the recorded command + result.
- **Advise, don't block.** The report recommends; it never commits, pushes, or gates.
- **Treat AI-written code with the same scrutiny as human code** — hallucinated method names, plausible-but-wrong API calls, outdated patterns.
- **Never translate German domain terms** (`Rechnung`, `Steuersatz`, `Filiale`, `Warengruppe`) — codebase contract.
