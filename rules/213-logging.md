---
description: Logging rules for the ERP (project logger, levels, lambda messages, exception logging). Applies to all Java that adds or changes logging.
globs: "**/*.java"
alwaysApply: true
---

> Excerpt, shortened. Logger declaration details and scheduler-protocol rules are cut.

# ERP-logging-AI-Guidelines

## Purpose

* Defines how to implement and review logging in the ERP: log levels, lambda messages, and exception-argument discipline.

## Mandatory Rules

### Logging Levels and Usage

* MUST use `error()` for critical failures, `warn()` for unexpected-but-non-fatal situations, `info()` for significant lifecycle events, `debug()` for troubleshooting detail.
* MUST NOT use `System.out.println()` or `System.err.println()`.
    * *Reason:* Standardized logging enables log management, levels, and rotation.
* MUST write new or edited log messages in **English**. Log messages are developer-facing diagnostics, never user-facing text.

### Dynamic Content and Performance

* MUST combine lambda-based logging WITH placeholders: `LOG.info(() -> "Message: {}", parameter)`.
* MUST NOT concatenate values into a log message with `+` inside a lambda — pass them as `{}` placeholder arguments instead.
    * *Reason:* Lambda + placeholders defer formatting until the level is enabled.

### Exception Logging

* MUST pass the exception object as the **last argument** to `error()`, `warn()`, **and `debug()`** — regardless of log level.
* MUST NOT pass `e.getMessage()` / `e.toString()` / `e.getClass()` in place of, or concatenated alongside, the exception object.
    * *Reason:* The logger prints message AND formatted stack trace (with cause chain) when given `e`; inlining `getMessage()` silently drops both.

#### Canonical exception-log patterns

| Scenario | Correct call |
|---|---|
| Constant message + exception | `LOG.debug(() -> "Lookup failed; falling back to default.", e);` |
| Dynamic context + exception | `LOG.debug(() -> "Lookup failed for id={}; falling back to default.", id, e);` |
| Multi-arg + exception (last) | `LOG.error(() -> "Save failed for user={} in tenant={}", userId, tenantId, e);` |

Anti-patterns — each is a MUST-fix in review, regardless of nearby project convention:

```java
LOG.debug(() -> "Lookup failed; cause=" + e.getMessage());               // WRONG — drops stack trace; concat
LOG.warn(() -> "id=" + id + ": " + e.getMessage(), e);                   // WRONG — concat + redundant getMessage()
LOG.debug(() -> "Lookup failed for id={}, cause={}", id, e.getMessage()); // WRONG — getMessage() instead of e
```

## Recommended Rules

* SHOULD provide meaningful context in log messages (IDs, key parameters).
* SHOULD avoid logging sensitive data (passwords, private personal info).
* SHOULD use `info()` sparingly to avoid log pollution in production.

## Refactoring / Legacy Guidance

* SHOULD replace legacy `e.printStackTrace()` — see `212-exception-handling.md`.

## Review Checklist

* [ ] Grep every `LOG.` call in the diff: exception passed as last arg, no `getMessage()` inlining?
