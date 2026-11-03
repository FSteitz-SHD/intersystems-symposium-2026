---
satisfies: [R1, R2]
---
# fault-tolerant-print-locations.2 PrintLocation reachable flag + tolerant cache provider

<!-- flow-next task, neutralized: package paths shortened, commit hashes and tracker keys removed, product-internal class prefixes generalized. -->

## Description
Proof point. Add `reachable` to the shared DTO and make the provider probe per server, catch, log and build an unreachable Druckort from configuration. Leaves the default rule (task .3) for later; both ship together on this branch, never released apart.

**Size:** M
**Files:** `erp-api/.../print/remote/PrintLocation.java`, `erp-impl/.../cache/provider/PrintLocationManagerCacheProvider.java`, new `erp-impl/.../cache/provider/PrintLocationManagerCacheProviderTest.java`, new `PrintLocation` serialization test in erp-api
**Touches:** [erp-api/.../print/remote/PrintLocation.java, erp-api/src/test/**/PrintLocation*Test.java, erp-impl/.../cache/provider/PrintLocationManagerCacheProvider.java, erp-impl/src/test/.../cache/provider/**]

### Approach
- Per-server loop with catch-and-continue like the print-server health indicator (`:39-73`); log with the project logger at WARN/ERROR incl. root cause (rule 213); the message-dialog exception with hardcoded German disappears from the provider (the sentence moves to the guard key, task .1/.3).
- Unreachable Druckort: same fields as `toPrintLocation` (`:53-93`) minus printers; pass a mutable `new ArrayList<>()` (`setPrintServices` sorts in place).
- Field default `true` so all creation sites (provider, client session, print-server mocker, test server starter, tests) stay valid. `PrintLocation` has no `serialVersionUID`; client and server ship together, so do not claim (in Javadoc or tests) that streams from the old class deserialize.
- Rules: 121 (erp-api), 131, 211-213, 222.

### Investigation targets
**Required**:
- `erp-impl/.../cache/provider/PrintLocationManagerCacheProvider.java:25-93`
- `erp-api/.../print/remote/PrintLocation.java`
- `erp-impl/.../print/health/PrintServersHealthIndicator.java:39-73`
- `erp-impl/.../print/remote/PrintServersConfig.java:110-155`
**Optional**:
- an existing cache provider test — provider test pattern
- `erp-impl/.../printservice/SpoolClientFactory.java:42-70`

### Key context
`PrintServersConfig.get()` and `SpoolClientFactory.createPrinterQueryClient` are static: test via `mockStatic` or a minimal package-private seam; do not restructure the provider beyond that. Tests may be committed separately under the test story.

## Acceptance
- [ ] Provider test: all reachable; one server throws → full cache, that Druckort `reachable=false`, empty printers, name/key/hostname/config from configuration, no server call for those fields; all throw → load succeeds with all flagged
- [ ] Serialization round trip with the current class keeps `reachable` (true and false); a default-constructed `PrintLocation` is reachable
- [ ] One log line per failed server with name and root cause
- [ ] `mvn -pl :erp-api,:erp-impl test -Dtest=...` for the new tests green; erp-it compiles

## Done summary
PrintLocation now carries `reachable` (default true, serialized). PrintLocationManagerCacheProvider no longer throws: a server that fails the probe is logged once (WARN, server name + URL, exception with cause chain) and yields a Druckort built from configuration (name, key/hostname, PrintServerConfig), with an empty mutable printer list, no default printer and `reachable=false`. The hardcoded message-dialog sentence is gone from the provider (guard key belongs to tasks .1/.3).

Tests: PrintLocationManagerCacheProviderTest (all reachable / one dead / all dead incl. client-creation failure; asserts no server call beyond the probe and one log line per failed server with root cause), PrintLocationSerializationTest (default reachable; round trip true/false). Both green; erp-it test-compile green.

Baseline: green (existing cache provider test via `mvn -pl :erp-api,:erp-impl -am test`). `flowctl gate classify` could not be run (worktree-isolation sandbox refused the flowctl invocation); the diff touches Java production code, so focused tests were run as the gate.

Notes for the conductor: the worktree base lacked the spec's `.flow` task files, so `flowctl anchor` failed there; the spec was read from the conductor checkout. Tests are in a separate test-story commit after the core-story production commit.

stage: impl-review - skipped(policy: host-deferred - conductor owns the gate)

Integrated as (commits removed).
stage: impl-review - ran SHIP (host, 3 draws, model: claude-opus-5-5)
Integrated verification: TZ=Europe/Berlin xvfb-run -a mvn -pl :erp-resources,:erp-api,:erp-impl,:erp-client -am test -Dtest=PrintLocationSerializationTest,PrintLocationManagerCacheProviderTest,MultiSelectionComboNonSelectableTest,RegularExpressionComboBoxTest (green)

## Evidence
- Commits: (removed)
- Tests: TZ=Europe/Berlin xvfb-run -a mvn -pl :erp-resources,:erp-api,:erp-impl,:erp-client -am test -Dtest=PrintLocationSerializationTest,PrintLocationManagerCacheProviderTest,MultiSelectionComboNonSelectableTest,RegularExpressionComboBoxTest (green)
- PRs: (removed)

stage: plan-sync - ran (wave 1 batch; updated .3, .4, .6)
