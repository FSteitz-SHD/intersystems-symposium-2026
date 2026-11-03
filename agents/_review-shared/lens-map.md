# Lens map — touched paths → which reviewer(s) to run

<!-- Excerpt, shortened; module paths generalized. -->

Drives `erp-review`'s `auto` lens selection. Given the changed-file list, pick the lens set and spawn the mapped agent(s). One agent = the fast path; ≥2 = a parallel fan-out. All agents are read-only and report against [`severity-and-format.md`](./severity-and-format.md).

## Auto-selected by touched files
| If the diff touches… | Lens | Agent | Model |
|---|---|---|---|
| any `*.java` (baseline) | `dev` | `erp-dev-reviewer` | sonnet |
| any `*.java` (baseline) | `silent-failure` | `erp-silent-failure-reviewer` | sonnet |
| `*Test.java` / `*IT.java`, or new public methods | `tester` | `erp-tester-reviewer` | sonnet |
| API module, RPC/REST contract, new aggregate/service | `architect` | `erp-architect-reviewer` | opus |
| repositories, `@Entity` / `@Query` / fetch strategy, migration scripts | `database` | `erp-database-reviewer` | opus |
| `@Transactional`, services doing REST/`java.net`/`Thread.sleep`/mail/print | `transaction` | `erp-transaction-reviewer` | opus |
| client module | `swing-mvc` | `erp-swing-mvc-reviewer` | sonnet |
| `*.properties`, new `getText(`, Javadoc, `documentation/**`, `.agents/rules/*.md` | `docs` | `erp-docs-reviewer` | sonnet |

## Opt-in only (add with `--lens`)
| Lens | Agent | When |
|---|---|---|
| `performance` | `erp-performance-reviewer` (opus) | N+1, projection candidates, hot-path algorithms |
| `security` | `erp-security-reviewer` (opus) | auth / crypto / input / SQL / file-IO / XML / outbound boundaries |
| `type-design` | `erp-type-design-reviewer` (sonnet) | raw-`Long` IDs, String-status, boolean-flag → enum/record |

## Notes
- **Proportionality.** A 1-file test diff → `tester` (+ `dev`). A multi-module persistence + RPC + UI diff → `dev` + `database` + `architect` + `swing-mvc` in parallel.
- **Dedup.** Where two lenses overlap (`dev` and `silent-failure` on a catch block), attribute each finding once at the highest severity.
