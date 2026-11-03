# Order header query: supplier "leading position" bug

<!-- flow-next spec, neutralized and translated from German: tracker keys and package paths removed. Captured as a follow-up during a larger feature, never planned into tasks yet - this is what a spec looks like before /flow-next:plan. -->

Known, already identified bug (found while working on the order print data model spec): `BestellungHeaderQueryProvider` resolves `Nr_Lieferant`/`Txt_Lieferantenkurzbez`/`Id_Lieferant` (and possibly other supplier-related header fields) through a "leading position" assumption, instead of consistently through the order-wide supplier set (analogous to `BestellungQuerySupport.resolveLieferantenFuerBestellung`, which was introduced in task 3 for exactly this pattern, because a significant share of orders have several suppliers spread across their positions). This is the same bug pattern that occurred three times in task 6 (order number, purchase contract number twice) — here it occurs a fourth time, this time for the supplier name on the header.

## Goal

Fix `BestellungHeaderQueryProvider` so the supplier-related header fields are resolved correctly, also when the order has several suppliers spread across its positions.

## Approach

1. Confirm the exact defect first, live or from the code (do not take it over blindly from memory) — which fields exactly are affected (`Nr_Lieferant`, `Txt_Lieferantenkurzbez`, `Id_Lieferant`, possibly more), and how exactly is the "leading position" determined today (which line/method)?
2. Check `BestellungQuerySupport.resolveLieferantenFuerBestellung`/`resolveLeitpositionMitLieferant` (already there, from tasks 3/4) — one of them is probably the right reuse for the header case, depending on whether a single "leading" supplier (analogous to the contact anchor) or set semantics is needed. Use the requirements doc and the existing tests to decide which functional expectation holds for the header (one supplier for the header field, or leave it empty when suppliers differ?).
3. Fix following the established pattern: reuse the shared, existing helper instead of building new, independent logic (avoid repeating the mistake that happened several times in task 6 through independent inline copies).
4. Test: regression test with an order that has several suppliers across its positions — must fail before the fix (empty/wrong supplier name) and pass after it.

## Boundaries

- No scope change — only the one bug in `BestellungHeaderQueryProvider`, no other fields/tables.
- Not part of any of the three currently running epic stories — standalone bug fix, separate ticket if needed.
