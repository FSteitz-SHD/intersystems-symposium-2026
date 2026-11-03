---
title: "Ask domain questions so they can be decided, with a mandatory free-text way out"
date: "2026-09-01"
track: knowledge
category: best-practices
module: documentation/guides/GUIDE-print-rebuild-orchestrator.md
tags: [domain, question, spec, decision, question-format]
applies_when: "a domain question goes to the business side, in a ticket or in a conversation"
---

<!-- flow-next memory entry, neutralized and translated from German: tracker link removed. -->

Ask a domain question to the business side so that the answer can be one sentence. Proven in a print-rebuild story: two questions in this form, each decided in one sentence.

**Fixed form:**

1. **Summary** first, the whole problem in two or three sentences.
2. **Why the question comes up now** — what changed so that it became a question at all.
3. **What it does today** — backed by real field, class and message names, messages quoted verbatim.
4. **What the change would do** — the contrast that creates the choice.
5. **Named variants (A) / (B) / …** — each with its consequence, not only its name.
6. **Own recommendation with the argument against the alternative.** "We think (A) is right, because (B) would lose something that exists today" beats "we lean towards (A)".
7. **What we explicitly cannot decide.** A recommendation on the mechanism plus "which word is the right one is for the business side to decide" is an honest split.
8. **Blast radius** — number of affected places, counted, not estimated.
9. **What is blocked and what keeps going** — name the dependency concretely, do not park the whole phase.

**Mandatory: always an open way out.** Every list of variants ends with a free-text exit, and that is not politeness. Two failure cases, and the second is the dangerous one:

- no variant fits;
- **the question rests on a wrong assumption** — wrongly cut axis, wrong presumption about what varies.

In the same ticket the story had phrased an idempotency key as "per output", although the real axis was "per print template"; it only surfaced while building. A neatly sorted choice on that basis would have made the mistake look like a settled matter. So: never present variants as complete, do not read silence on the open way as agreement, and when the question itself is rejected, **rephrase the question** instead of presenting the variants again.

**The answer can enlarge the job.** Do not treat it as filler. In the case at hand the more expensive variant of an optional extra question was chosen and the main question was corrected along the way. Then write it back into every artifact that had assumed the narrower reading, and say explicitly that the scope has moved.

Why not derivable: only domain knowledge about how we work together. Applies beyond the print rebuild to every domain question.
