---
description: ERP Java DB migrations — schema and data scripts, naming, safety, idempotency. Load when writing or reviewing a migration script.
globs:
  - "**/migration-scripts/**/*.java"
alwaysApply: false
---

> Excerpt, shortened. Package layout, class naming, audit-table rules and the server-call rule are cut. Examples are synthetic.

# ERP-db-migration-AI-Guidelines

## Purpose
* Implement and review **Java database migrations**: **schema** (DDL) and **data** (DML).
* Ensure consistent, safe, and idempotent database changes.

## Mandatory Rules

### Safety and Idempotency
* **Idempotency:** MUST keep scripts idempotent (e.g. add a column only if it does not exist).
* **Idempotency for data seeds** (metadata rows, lookup tables, default settings):
    * MUST guard data-seed blocks with an existence check and **return early** when the target row already exists. The check belongs in the migration script at the call site — not deferred to a `createOr…` helper, even when the helper's name suggests upsert semantics.
    * MUST audit any `createOr…`, `saveOrUpdate`, `upsert`, `findOrCreate` helper before relying on it as the idempotency boundary. Many such helpers re-run every setter on the "update" branch. If any setter writes a constant value, a second run **clobbers** data an admin may have edited between runs — that is **not idempotent**.
    * MUST keep the guarded section side-effect-free aside from the seed itself: no expensive application bootstrap, no new transactions, no cache loading until *after* the existence check confirms work is needed.
* **SQL Injection:** NEVER string-concatenate user values into SQL.

```java
// Do — caller-side guard, early return
private void seedCountryLookup() {
    if (hasResult("SELECT 1 FROM Lookup.Country WHERE Code = 'DE'")) {
        return;                                          // idempotent skip
    }
    initApp();                                           // costs deferred until needed
    lookupService.createOrUpdateCountry("DE", "Deutschland");
}
```

```java
// Don't — trusting a "createOr…" helper as the idempotency boundary
private void seedCountryLookup() {
    initApp();                                           // paid even when no-op
    // The helper resets the display name on every call,
    // overwriting whatever an admin edited since the first run.
    lookupService.createOrUpdateCountry("DE", "Deutschland");
}
```

### Local Execution
* MUST add a `main` method to **every new** migration class so it can be run on its own against a local database.

## Do / Don't Examples
### Do
```java
if (!schema.hasColumn("MY_TABLE", "MY_COL")) {
    schema.addColumn("MY_TABLE", "MY_COL", "VARCHAR(255)");
}
```
*Reason: Ensures idempotency, allowing the script to be executed multiple times without errors.*

## Review Checklist

- [ ] **Data-seed idempotency:** Data-seed blocks guarded by an existence check that **returns early** before any bootstrap or transaction runs? `createOr…` / `saveOrUpdate` helpers verified to not silently overwrite admin-edited fields on re-run?
