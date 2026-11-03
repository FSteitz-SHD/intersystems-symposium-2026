---
satisfies: [R6]
---
# fault-tolerant-print-locations.4 Druckcenter: unreachable Druckort refusal and per-Beleg failure

<!-- flow-next task, neutralized: package paths shortened, commit hashes, receipt paths and tracker keys removed. The HTML comments marked "Updated by plan-sync" are original: plan-sync rewrote this task after tasks .1 and .3 had landed. -->

## Description
Make the new print path honest about unreachable Druckorte: release refused with the dedicated key before the printer check; a run with an unreachable-but-present Druckort starts and fails each Beleg's Drucker output with the dedicated reason (other Ausgabearten go out); an Ausführungsplan stays active on this refusal; the guard exception under total outage becomes a technical abort with reason. Repairs the dead null-guards only here (decision 4 of the story).

**Size:** M
**Files:** `erp-impl/.../print/PrintOutputResolver.java`, `PrintRunJob.java`, `plan/ScheduledPrintRelease.java`, tests `PrintOutputResolverTest`, `PrintRunJobFaultSplitTest`, `plan/ScheduledPrintReleaseTest`
**Touches:** [erp-impl/src/main/java/.../print/**, erp-impl/src/test/java/.../print/**]

### Approach
- `requirePrintLocation` (`PrintOutputResolver.java:451-478`): after the existing unknown/local checks, refuse `!isReachable()` with `print.printcenter.output.printLocationUnreachable.error.message` (`{0}` = Druckort; exists since .1; `PrintLocation#isReachable()` exists since .2) <!-- Updated by plan-sync: .1 key names -->; keep before `requirePrinter` (`:480-499`).
- `PrintRunJob#resolvePrintLocation` (`:536-584`): unreachable but present passes (no abort); catch the guard exception from `getPrintLocation` and abort technically with a reason (pattern `:570`).
  <!-- Updated by plan-sync: .3 `getDefaultPrintLocation()` now returns the first reachable Druckort or throws the central guard (message-dialog exception, key `print.printLocation.serverUnreachable.error.message`, `{0}` = comma-joined server names) when all are unreachable. `getPrintLocation(key)` returns a known key even if unreachable, but a blank/unknown key now falls back to the first reachable one or throws the guard. So the guard in resolvePrintLocation (l.558) is the only place to catch; the existing key-vs-substitute comparison still applies. `getDefaultPrintLocation` is `@Transactional`: catching the guard inside an outer transaction may leave it rollback-only (review FYI, same as before) - abort via `aborter.abortTechnical` without relying on that transaction. -->
- Per Beleg: fail only the Drucker Ausgabeart, via the existing withheld-target mechanism computed once at run start (`PrintOutputResolver#withheldTarget`, `PrintOutputResolver.java:600-640`, used from `PrintRunJob.java:379`), not by failing the whole position in `PrintRunPositionExecutor#enqueue` (that would drop E-Mail/Archiv of the same Beleg). Respect the comment near `PrintRunJob.java:344` about same-reason failures: this is the user's explicit choice (Decision Context).
- Scheduled release: `ScheduledPrintRelease#classifyOutputRefusal` (`plan/ScheduledPrintRelease.java:879-895`) must classify the new unreachable key as `PrintPlanReleaseOutcome.refusedTransient` (pattern: `KEY_REPORT_SERVICE_UNAVAILABLE`), otherwise it falls through to `deactivating(OUTPUT_TARGET_UNRESOLVABLE)` and a short outage switches the plan off.
- EDI / `withoutOutput` / channels not requiring a Druckort never check reachability (overlap with the EDI spec).
- Per-Beleg failure text: key `print.printcenter.position.printLocationUnreachable.error.message` (`{0}` = Beleg, `{1}` = Druckort; same argument order as `printLocationGone`) <!-- Updated by plan-sync: .1 key names -->.
- Stale comments to fix: `PrintOutputResolverTest:546` and `PrintRunJobFaultSplitTest:239` (and the matching comments in `PrintRunJob.java:561` / `PrintOutputResolver#requirePrintLocation`) say an unknown key yields "the first Druckort"; it is now the first *reachable* one, or the guard exception under total outage. <!-- Updated by plan-sync: .3 -->
- Existing tests using `mock(PrintLocation.class)` (e.g. `PrintRunJobFaultSplitTest:224,253`) must stub `isReachable()` true or switch to real instances.

### Investigation targets
**Required**:
- `erp-impl/.../print/PrintOutputResolver.java:360-500`
- `erp-impl/.../print/PrintRunJob.java:530-590`
- `erp-impl/.../print/PrintPositionHandOver.java:330-360`
- `erp-impl/.../print/PrintOutputResolverTest.java:530-600`
- `erp-impl/.../print/plan/ScheduledPrintRelease.java:500-630, 870-900`
**Optional**:
- the EDI spec's task on dispatch without Druckort

## Acceptance
- [ ] Release with unreachable Druckort → refused with the new key (not `printerMissing`)
- [ ] Run with unreachable Druckort → not aborted; every Beleg's Drucker output fails with the unreachable reason; no other Druckort used
- [ ] Run with [E-Mail, Drucker] and unreachable Druckort → E-Mail goes out per Beleg, only Drucker fails
- [ ] Ausführungsplan release with unreachable Druckort → `refusedTransient`, plan stays active (ScheduledPrintReleaseTest)
- [ ] Total outage during run start → technical abort with reason, no uncaught exception
- [ ] Deleted Druckort → existing `printLocationGone` behaviour unchanged; EDI / without-output unaffected
- [ ] `mvn -pl :erp-impl test -Dtest='PrintOutputResolver*Test,PrintRun*Test,ScheduledPrintReleaseTest'` green

## Done summary
The Druckcenter now handles an unreachable Druckort as R6 asks:
- **Release:** refused with `print.printcenter.output.printLocationUnreachable.error.message` (`PrintOutputResolver#KEY_PRINT_LOCATION_UNREACHABLE`, `{0}` = Druckort key). The check runs after the unknown/local checks and before the printer check, so the user never sees `printerMissing`.
- **Ausführungsplan:** `ScheduledPrintRelease#classifyOutputRefusal` classifies that key as `refusedTransient`, the same as the report service. The plan stays active.
- **Run:** a run whose Druckort exists but is unreachable starts without an abort. For each Beleg only the Ausgabeart that needs the Druckort fails, with `print.printcenter.position.printLocationUnreachable.error.message` (`PrintRunPositionExecutor#KEY_PRINT_LOCATION_UNREACHABLE`, `{0}` = Beleg, `{1}` = Druckort). E-Mail and Archiv go out. A Beleg whose only Ausgabeart is the Drucker fails with `noOutputLeft`, and its Drucker row carries the unreachable reason; this matches the existing withheld-channel behaviour.
- **Central guard (total outage plus an unknown key):** the release catches it as `printLocationUnknown`. The run catches it and aborts technically with the existing `printLocationGone` reason. It is never uncaught. A known key is still answered when unreachable, so the guard can only fire for a key the cache no longer knows.
- **Deleted Druckort / withoutOutput / EDI:** unchanged. Stale comments that said "first Druckort" now say "first reachable Druckort".

Deviation from the task approach (decision for the reviewer): the per-Beleg failure is not done through `PrintOutputResolver#withheldTarget` at run start. It is decided in `PrintPositionHandOver#resolve`, which now takes the run's Druckort (resolved once at run start), after the EDI routing. Two reasons:
1. A withheld target is set before routing. Withholding the Drucker would therefore also fail every Beleg that the document type routes onto EDI, which needs no Druckort (R6: EDI unaffected).
2. A reason computed at run start cannot carry the Beleg that the `{0}` parameter of the position key requires.

The mechanism is otherwise the same: a FAILED per-Ausgabeart attempt in `refused`, never a failure of the whole position in `enqueue`. The reachability flag is read off the Druckort instance that the job fetched once at run start.

Tests (8 new; the focused suite went from 489 to 497, all green):
- `PrintOutputResolverTest`: `refusesAnUnreachablePrintLocation`, `refusesAnUnknownPrintLocationUnderTotalOutage`.
- `ScheduledPrintReleaseTest`: `testUnreachableServiceIsTransient`, parametrized over the report-service key and the Druckort key.
- `PrintRunJobFaultSplitTest`: `anUnreachableDruckortStartsTheRun`, `aVanishedDruckortUnderTotalOutageAbortsByName`.
- `PrintRunPositionExecutorTest`: `anUnreachableDruckortFailsOnlyThePrinter` (`[E-Mail, Drucker]`), `anUnreachableDruckortFailsAPrinterOnlyBeleg`, `anEdiRoutedPrinterIgnoresAnUnreachableDruckort`. The mocked `printLocation` in `setUp` now stubs `isReachable()` to true.

Mutation check: I disabled each of the four production guards in turn (hand-over check, resolver rung, transient classification, job catch). Each mutant made at least one new test fail. All files were then restored.

baseline: green (same focused command, 489 tests, before any edit)
Verify: `gate classify` gave FULL. The focused tests were green (497). `erp-it` test-compile was green.
Feature route changed: none (behaviour only; no new UI entry).

stage: impl-review - skipped(policy: host-deferred - conductor owns the gate)

stage: impl-review - ran SHIP (host, 3 draws, model: claude-opus-5-5); approach deviation (PrintPositionHandOver instead of withheldTarget) accepted by review
follow-ups: UnexpectedRollbackException WARN on total outage at run start; Javadoc paragraph between @param tags

## Evidence
- Commits: (removed)
- Tests: mvn -pl :erp-impl -am test -Dtest=PrintOutputResolver*Test,PrintRun*Test,ScheduledPrintReleaseTest (497 green, worker; re-run green by 3 reviewers), mvn -pl :erp-it -am test-compile (green)
- PRs: (removed)

stage: plan-sync - ran (wave 3; updated .7)
