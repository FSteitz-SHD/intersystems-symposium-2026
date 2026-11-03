# Dev starter: reliably disable groupware communication

<!-- flow-next spec, neutralized: tracker keys, host names and internal class names removed; the ticket quotes are translated paraphrases. -->

## Conversation Evidence

> user (turn 1): "<ticket key>"
> ticket (reporter, description, translated): "The developer server starter lets the developer choose at startup that communication with groupware is switched off."
> ticket (reporter, description, translated): "That no longer works."
> ticket (reporter, description, translated): "The masses of stack traces flooding the console make it hard to simply follow what is going on."
> ticket attachments (two stack traces): `UnknownHostException` from the groupware REST login, reached via client RPC calls for the groupware connection test and the notification polling.

## Goal & Context

<!-- Source: 0% user / 60% [paraphrase] (ticket) / 40% [inferred] -->

The developer server starter offers a start option to switch off communication with the groupware server, an external service the ERP talks to over REST. The option no longer takes effect: the dev server still calls the groupware configured in the database, and on a dev machine that host is typically unreachable. Every such call logs a full stack trace, so the console fills with groupware noise and is useless for following what the server actually does. Goal: the option works again, so a developer can run against any test database without groupware noise.

## Architecture & Data Models

<!-- Source: 100% [inferred], each premise verified by reading the code -->

Verified premises the design depends on:

- **P1 — how the option is wired.** The starter's "disable groupware" setting is turned into a JVM system property that overrides the groupware settings with `{"url":""}` (before this fix; now `{"url":"","internalNetworkUrl":""}`). Verified: read of the server config builder.
- **P2 — the override is admin-only.** Per-class dev-property overrides are applied only when the logged-in user is the built-in admin user. A regular employee login, or any call path without a user, gets the real database values. Verified: read of the dev-property reader and its three call sites.
- **P3 — the override blanks the wrong field.** The groupware login resolves its endpoint as `internalNetworkUrl` if set, otherwise `url`. Blanking only `url` leaves groupware active whenever the database carries an internal network URL. Verified: read of the groupware login code.
- **P4 — the client gate keys on `url` only.** The client considers groupware "active and set up" when the module is licensed and `url` is non-blank, and skips the connection test otherwise. Verified: read of the client gate.
- **P5 — the stack traces' entry points are client RPC calls** (connection test at client start, notification polling). Verified: attachments. The attachments predate a rework of the groupware REST client, so the frames no longer match current class names.
- **P6 — the server cache bypassed every dev override for system-only settings.** The groupware settings are system-level; the server cache returned them straight from the system cache, never through the dev-property reader, so the override had no effect on the server path regardless of P2/P3. Verified: a failing test through the cache (found by impl review).
- **P7 — the dev-properties file beat the per-class system property** for the admin user. Verified: failing test.

Not runtime-verified: which of P2/P3/P6 produced the original traces on a real dev server; all three are fixed.

## Edge Cases & Constraints

- Dev-only: the switch is set by the developer starter; production servers never set the system property, so production groupware behaviour must not change. [inferred]
- The option must hold for every login on the dev server, not only the admin user — a developer often logs in as a regular employee to reproduce a reported behaviour. [inferred]
- The fix must not require editing the groupware settings rows in the database. [inferred]

## Acceptance Criteria

- **R1:** With the starter's "disable groupware" option on, the dev server makes no outbound groupware REST call (login, connection test, notifications, any other groupware resource) for any logged-in user — admin or regular employee. Errors: no error surface beyond R3. [paraphrase]
- **R2:** R1 holds regardless of what the database's groupware settings contain, including a configured internal network URL. [inferred]
- **R3:** With the option on, a client start followed by an idle client session of at least one notification-poll interval produces no groupware-related stack trace in the server console. At most one log line at server start states that groupware communication is disabled by the dev option. [paraphrase]
- **R4:** With the option on, the client treats groupware as not set up: groupware menu items are removed and no groupware notification popups appear. [inferred]
- **R5:** With the option off (and on every server not started via the dev starter), groupware behaviour is unchanged. [inferred]

## Boundaries

- No change to production groupware configuration or to the groupware REST client's behaviour when groupware is configured. [inferred]
- No new UI in the developer starter; the existing option is reused. [inferred]

## Decision Context

<!-- Source: 100% [inferred] -->

Fix the option where it is applied (the dev-property override and what it overrides), not by adding "is groupware disabled?" guards at each groupware call site: the call sites already react correctly to an groupware that is not set up (P4), so making the override reliable reaches every path at once. Rejected: suppressing or downgrading the groupware stack-trace logging — it hides real failures on test and production servers.

## Parked unknowns

- End-to-end confirmation on a real dev server (starter option on, database with groupware configured, login as admin and as regular employee, console free of groupware stack traces). Not automated.
