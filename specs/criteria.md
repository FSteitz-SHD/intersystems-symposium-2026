# Global acceptance criteria

<!-- flow-next global criteria (.flow/criteria.md), neutralized: tracker keys removed, module names generalized. -->

Standing, project-wide criteria for every flow-next completion review in this repo.

Derived from the rules review of an earlier feature (form-letter print rebuild, 2026-08-05): each criterion below encodes a defect class
that actually occurred in that feature and that the per-task `cursor impl-review` could not catch,
because that backend receives only the raw diff and is explicitly told to ignore `CLAUDE.md` /
`AGENTS.md` — i.e. it never sees `.agents/rules/`.

- **G1:** Newly authored Javadoc and code comments are in English; German is kept only for domain terms (`Formbrief`, `Druckvorgabe`, `Anschrift`, `Rechnung`). Pre-existing German text in a touched file may stay untouched, but a newly written block must be English — including `//` inline comments, not just Javadoc. [rule 222]
- **G2:** New i18n keys follow `<namespace>.[module].<action|entity>.[property].<category>` with a functional-area namespace and an explicit category suffix, and are added to EVERY locale bundle under `erp-resources` (German value in the default bundle, empty value in `_en`/`_fr`/`_nl`/`_zh`). Copying the flat legacy key style of a neighbouring entry is not a justification. [rule 221 §4/§5]
- **G3:** Criteria are judged against EVERY commit on the branch, not only the net diff. A violation that is introduced and later reverted still ships in the merged history and counts as a finding; a net-diff-only review is insufficient for i18n, documentation and migration scripts.
- **G4:** Every identifier a document names — class, method, table, column, constant, i18n key, or a constant's VALUE — exists in the code exactly as written. Verification documents (runbooks, smoke checks) are cross-checked against the literals they instruct a reader to search for; a doc that sends a tester after a non-existent value is a defect, not a nit.
- **G5:** No enum is persisted via `ordinal()`. A `Kz_*` or status column stores an explicit stable key or the enum `name()`. Inserting a constant into an existing enum must not be able to re-label already-written rows.
- **G6:** Every new FK column, and every column used in the `WHERE` of a bulk / `@Modifying` DML statement, has an index created in the same migration. On IRIS an unindexed scanning DELETE escalates to a whole-table lock past `LockThreshold`.
- **G7:** An invariant asserted in Javadoc (idempotency, "the caller invokes X", delete/flush ordering) is either exercised by a test or traced to the concrete code path that provides it. An unwired documented claim is a defect — especially when the Javadoc was copied from a reference implementation whose precondition this slice does not satisfy.
- **G8:** A feature gate that routes between an old and a new code path is decided from data whose live-DB state has been verified — never from a Java field default or from what a new entity would get. Where existing rows may already carry the new value, the runtime path is gated independently of the stored value.
- **G9:** Tests exercise the real lifecycle of the state under test. A test that injects the very value a production path is supposed to produce (a setter on a runtime carrier, a hand-built session, a mocked loader) does not count as coverage for that path. State captured in one phase and consumed in another must be tested across the phase boundary, including any session reload or serialization in between.
- **G10:** A silent auto-correction of persisted configuration is logged at WARN with the old and the new value, or surfaced to the operator. Config decisions must not be made for the user invisibly.

## Planning criteria

Added 2026-08-06 after the retrospective of that feature. The criteria below apply at SPEC time, not at
completion review. Both P0 reworks in that feature traced to the same gap: the interview captured
what the user knows (product intent — it was correct and never had to be revised), while the design
rested on unverified assumptions about what the EXISTING CODE does. Nobody could have asked the user
those; they had to be asked of the codebase.

- **G11:** Every factual claim about existing behaviour that the design depends on is written into the spec as a premise WITH its verification method (`grep`, `git show <pre-feature-sha>:<file>`, a live-DB query, a named test) and is verified before tasks are cut. A Java field default, an entity initializer, or "what a new row would get" is NOT evidence about what existing rows contain. An unverified premise becomes an investigation task, never an assumption a later task builds on.
- **G12:** Where a design depends on the semantics or distribution of EXISTING data, it is checked against a real database (dev/QA via the read-only IRIS introspection route in `CLAUDE.md`), not inferred from the mapping. One `SELECT DISTINCT` on the column in question outranks any amount of reading.
- **G13:** For a feature that spans a multi-phase runtime lifecycle (create → persist → reload → process, or anything crossing an RPC/serialization boundary), the FIRST task after the schema exists is an end-to-end walking skeleton that executes the whole lifecycle — however trivial its payload. Feature logic is not designed against a lifecycle nobody has run. Cf. G9, which is the same requirement stated for the finished tests.
- **G14:** A concern that applies to many call sites is specified at the shared funnel those sites pass through (framework method, listener, interceptor, base class), with completeness preferably enforced by the compiler or the framework, else by an architecture test. A plan whose acceptance needs a grep to prove "every site has the call" has picked the per-site mechanism; it is revised, or the spec names the funnel that was looked for and why it does not fit. A design states the goal ("every bound widget carries its property"), not the per-site mechanism. [rule 211 §5]
