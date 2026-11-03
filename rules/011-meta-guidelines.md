---
description: Authoring rules for the ERP agent rule files (.md) — structure, scope, token budget, normative vocabulary. Load when editing `.agents/rules/**/*.md`.
globs: "**/.agents/rules/**/*.md"
alwaysApply: false
---

> **Human source:** the human-written authoring guideline — authoritative; this rule is the derived form.

# ERP Rule-Authoring Guidelines

## Purpose

* Defines how AI guidelines for the ERP are authored: consistent structure, normative vocabulary, token budget.
* The original human-written guideline is the source of truth. The derived `.md` is its operational interpretation; on contradiction, stay conservative and flag — do not silently reinterpret.

## File layout

* Path: flat under `.agents/rules/`, named `NNN-kebab-topic.md`. Band meanings: see the router (`AGENTS-excerpt.md`).
* Frontmatter: `description` (≤150 chars, single sentence — agents use it for trigger decisions), `globs` (precise — no `**/*.java` for a client-only rule), `alwaysApply` (true ONLY when every matching file genuinely needs it).

## Mandatory section order

Use exactly this order; omit a section entirely if it has no content (empty placeholders are forbidden).

1. **Purpose** — 1–3 bullets, no restating of the title.
2. **Scope** — max 3 bullets; do not repeat what `globs` already implies.
3. **Source Interpretation** — single line: `Source: <doc>. The source is authoritative; this file is its operational interpretation.`
4. **Mandatory Rules** — binding rules. Vocabulary: MUST / MUST NOT / NEVER / ALWAYS.
5. **Recommended Rules** — strong recommendations. Vocabulary: SHOULD / SHOULD NOT / MAY.
6. **Decision Rules** — only for ambiguous / choice-heavy situations.
7. **Do / Don't Examples** — ≤10 lines per block, include a one-line **Reason** explaining *why*.
8. **Refactoring / Legacy Guidance** — only if legacy treatment differs from the rules above.
9. **Review Checklist** — only items that add a check **beyond literally re-reading the Mandatory Rules** (cross-cutting greps, multi-file interactions).

## Token Efficiency (MANDATORY)

Every line costs context on every matching task.

* **One owner per rule topic.** A rule lives in exactly ONE guideline; others cross-reference (`see 212 §X`).
* **No `AI models` subject prefix.** Write `MUST NOT use wildcard imports.`, not `AI models MUST NOT use wildcard imports.`.
* **No "AI Behavior Expectations" with global defaults** (`act conservatively`, `match neighbours`). Include only behaviour the topic forces *beyond* the global defaults.
* **No duplicated rationale.** State a **Reason** once where the rule lives.
* **Target size:** ≤120 lines unless topic complexity genuinely requires more (justify in the file).

Global defaults that apply to every guideline — NEVER restate per file:

* act conservatively, preserve behaviour, avoid speculative redesign
* follow existing naming conventions, prefer consistency over novelty
* ASK THE USER on contract/schema/naming ambiguity, before risky writes, when patterns disagree

## Normative vocabulary

* **MUST / MUST NOT** — invariant; breaking it damages correctness or architecture.
* **NEVER / ALWAYS** — absolute; no project exception expected.
* **SHOULD / SHOULD NOT** — strong preference; exceptions possible.
* **MAY** — optional / context-dependent.

Mixing soft and hard wording inconsistently is forbidden.

## Writing style

Use: short technical sentences, operational statements, project terminology.
Avoid: motivational prose ("it would be good if"), vague hedging ("in many cases", "generally speaking"), large walls of text.

Reason blocks: add a one-line `*Reason:*` below a rule when the technical background is non-obvious (e.g. "Required for the binding framework, which reads properties via reflection"). Skip when the rule is self-evident.

## Transformation rules (source → .md)

* Convert descriptive text into explicit instructions: "child entities should normally only be modified through the aggregate root" → `Child entities MUST be modified through the aggregate root.`
* Split paragraphs that mix rule / explanation / example into separate bullets.
* Preserve domain terminology — do not rename.
* Do NOT invent rules that are only weakly suggested. If the source is ambiguous, prefer a narrow operational interpretation or capture it as a Decision Rule.

## Authoring-time self-check (NOT persisted)

Before saving the `.md`, verify: source fidelity, actionability (MUST rules concrete enough to grep or check), completeness, token efficiency, section order. The output of this check MUST NOT appear in the file.
