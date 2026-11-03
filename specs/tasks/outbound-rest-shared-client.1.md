---
satisfies: [R1, R2, R3, R4, R5, R6, R7, R8, R9]
---
# outbound-rest-shared-client.1 Implement Outbound REST: stateless client infrastructure once per JVM

<!-- flow-next task, neutralized: commit hashes, merge-request references, tracker keys and person initials removed; internal library and integration names generalized as in the spec. -->

## Description
TBD

## Acceptance
Every R-ID in the parent spec's ## Acceptance Criteria is satisfied; judge this task against the spec's criteria directly.

## Done summary
- RESTResourceFactory closes the per-call client on every path; a Response result is buffered first (R1, R9).
- SharedHttpClients: one HttpClient pool per (url, pool size), non-owning engine per call, cookies off, checkout/connect/socket timeouts, stale validation; connection tests use an owned pool; pools closed on shutdown (R2-R5).
- Base library (merged, shipped in the next dependency BOM patch): no withConfig, ClientJacksonProvider subclass, ObjectMapper per resource via ClassValue (R6, R7). RESTClientExtension relies on super.config.
- R8: test-classpath harness (client build 1.79 -> 0.24 ms, call 4.7 -> 1.3 ms) and OutboundRestEndToEndTest; fat-jar measurement waived (decision tech lead 2026-10-14).
- Decision log under documentation/decisions/.
- Not run: live smoke against groupware / the bridge / address lookup (the live-bridge IT is @Disabled, needs the dev bridge).

stage: impl-review - ran (model: cursor gpt-5.6-sol-high: NEEDS_WORK -> SHIP; fable-5.1 second opinion: one P2 timeout finding fixed; base library fable-5.1: SHIP)

## Evidence
- Commits: (12 commits, removed)
- Tests: mvn -o -nsu -pl :erp-impl test -Dtest=OutboundRestEndToEndTest,RESTClientExtensionTest,CachingRESTResourceProviderBuilderTest,SharedHttpClientsTest,RESTResourceFactoryTest,RestLoginFactoryBridgeTest, mvn -o -pl base-rest-client -am test (base library repo)
- PRs: (removed)
