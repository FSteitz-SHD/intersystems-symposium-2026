---
title: "Query IRIS read-only via JDBC, not via the Atelier REST API (NULL, count trap)"
date: "2026-08-31"
track: knowledge
category: tooling-decisions
module: documentation/guides/GUIDE-intersystems-iris-mcp.md
tags: [iris, jdbc, atelier, sql, "null", count]
applies_when: "read-only SQL analyses against an IRIS instance, especially discriminators and row counts"
---

<!-- flow-next memory entry, neutralized and translated from German: local paths, host names, the credentials class and table names generalized. -->

For read-only SQL analyses against an IRIS instance use **JDBC**, not the Atelier REST query action. No maintained tool in the repo: the probe is thirty lines and is rebuilt per session. A wrapper around the REST interface was tried and removed again, because it produced more errors of its own than it saved.

**The reason is not convenience: the REST action destroys NULL.** It returns SQL NULL as the type default, `0` for numeric and empty string for text columns. Measured: rows selected with `WHERE <discriminator> IS NULL` come back as `{"<discriminator>": 0}`. A `GROUP BY` over a nullable column thereby invents a value. That sent an analysis after a document type `0` that never existed. Over JDBC the trap does not exist.

**Ingredients on a developer machine:**

- Driver: `intersystems-jdbc-<version>.jar` from the local Maven repository
- JDK: the project JDK
- URL: `jdbc:IRIS://<host>:1972/<namespace>`
- Account: from the team's secret store. Read it programmatically, never write a value into prose, ticket, chat or onto a command line.

**Three things the probe must get right:** print NULL as its own marker (`rs.wasNull()`), send only `SELECT` and `EXPLAIN`, and stream when counting (see below). Read-only is a habit, not a boundary — the account may write, so dev and QA instances only.

**Count trap, reproduced over both transports:** a `COUNT` over a whole legacy print table is inflated. One header table reports `COUNT(*)` 3936 and `COUNT(SessionId)` 4056; streamed and counted client-side it is **144**. Its position table: 11402 reported, 307 actual. Caché extent shortcut. Filtered and grouped counts match the ground truth exactly. So filter, group or stream, and never quote a bare total.
