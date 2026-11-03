---
satisfies: [R1, R2, R3, R4, R5]
---
# dev-starter-reliably-disable-groupware.1 Implement Dev starter: reliably disable groupware communication

<!-- flow-next task, neutralized: commit hashes, merge-request link, tracker keys and person handles removed. -->

## Description
Direct-route owner task: implement the spec; see Done summary.

## Acceptance
Every R-ID in the parent spec's ## Acceptance Criteria is satisfied; judge this task against the spec's criteria directly.

## Done summary
- Dev system-property overrides apply to every caller and on the server cache path for system-only settings; the dev-properties file stays admin-only and loses to the per-class override.
- The starter's disable-groupware override blanks `url` and `internalNetworkUrl`.
- Side effect: the starter's disable-proxy override (same mechanism) now applies to all users too.

Defect route:
- prior fixes: git log on touched files, remote branches, tracker assignee — none
- diagnosis: confirmed admin-only gate, internalNetworkUrl precedence, system-only cache bypass, file-over-sysprop precedence (each by a test failing on base)
- introduced by: skipped: no known-good revision named
- base: 5 of 7 new tests fail on the base commit | head: 47/47 properties tests green (incl. 4 tests from MR review)
- live: not run: needs dev server + database with groupware configured

stage: impl-review - ran (cross-model, GPT): NEEDS_WORK (3 P1; 2 fixed, 1 pre-existing → follow-up) then SHIP
Follow-ups: one client search path gates groupware only on the licence; end-to-end test from starter option to client-visible setting.

## Evidence
- Commits: (removed)
- Tests: `mvn -o -nsu -pl :<server-module> -am test -Dtest=<properties test classes>`
- PRs: (removed)
