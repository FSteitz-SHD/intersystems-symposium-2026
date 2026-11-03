# AGENTS.md (excerpt: the rule router)

> Excerpt from the repository's `AGENTS.md`, shortened. Build commands, module overview, tool setup
> and tracker workflow are cut. `CLAUDE.md` imports this file and only adds Claude-Code-specific notes.

Guidance for every coding agent working in this repository (Claude Code, Codex, any other
AGENTS.md-aware tool). This file is the single source.

## Rules & AI Guidelines

Treat the rule files in `.agents/rules/` as operational instructions. Read the rule(s) matching the
file you are about to touch **before** editing.

## Source-of-truth policy

The `.md` files are derived AI guidelines. The original human-written guidelines remain
authoritative. If a `.md` file is ambiguous or contradicts the source, stay conservative and flag
it, do not silently reinterpret. (See `011-meta-guidelines.md`.)

## Rule bands

All rule files live flat in `.agents/rules/` as `NNN-kebab-topic.md`. The hundreds digit is the band:

| Band | Meaning |
|---|---|
| 000-099 | System & workflow: meta rules, agent commands, tool and orchestration patterns |
| 100-199 | Architectural layers: API, domain, transactions, persistence, migrations |
| 200-299 | Engineering standards: coding style, exceptions, logging, i18n, docs, testing |
| 300-399 | Business domains (reserved) |
| 400-899 | Integrations, security, infra, client/UI (`701-mvc`), performance |
| 900-999 | Legacy & risk: migration playbooks with a "Last Reviewed" date |

## How to use the rules

For any task, identify the file(s) you're about to touch and load every rule whose glob pattern
matches. Most rules are in the 210-230 range and apply to all Java files. Additionally:

- Rules with `alwaysApply: true` are mandatory for all matching files.
- Rules with `alwaysApply: false` apply only when their topic is explicitly in scope (e.g. authoring
  a `.md` or writing a migration script).

## Quick decision matrix

| Task                                     | Load rules                                 |
|------------------------------------------|--------------------------------------------|
| Any Java edit                            | 012, 132, 211, 212, 213, 221, 222          |
| New/changed REST or RPC contract         | + 121                                      |
| Service / domain logic                   | + 131                                      |
| JPA entity, repo, or query change        | + 141                                      |
| DB migration script                      | + 142                                      |
| Runtime UPDATE on an audited entity      | + 143                                      |
| Test code (`*Test` / `*IT`)              | + 231, 233                                 |
| Swing client code                        | + 701; check the change in the running client with `/erp-ui-check` |
| Authoring or editing a `.md` rule       | + 011                                      |
| Designing a multi-agent workflow/skill   | + 051                                      |
| Running mvn to verify a change           | + 052                                      |
