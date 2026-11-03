# Specs, Tasks und Memory

Hier liegen Auszüge aus dem `.flow/`-Ordner unseres ERP-Repos. `.flow/` ist das Arbeitsverzeichnis von [flow-next](https://flow-next.dev), dem Spec- und Task-Workflow, den wir mit Claude Code und Codex nutzen. Alles darin ist Markdown im Repo, versioniert wie Code. Es gibt kein externes Tool, in dem der Plan lebt.

Im Original sind es zum Stand dieses Auszugs 24 Specs mit rund 165 Tasks, drei kuratierte Memory-Einträge und eine Datei mit globalen Akzeptanzkriterien.

## Die Begriffe

- **Spec**: was gebaut wird und woran man erkennt, dass es fertig ist. Eine Spec pro Ticket oder Feature.
- **Task**: ein Arbeitspaket aus einer Spec, so geschnitten, dass ein Agent es mit frischem Kontext umsetzen kann. Jede Task sagt, welche Anforderungen der Spec sie erfüllt.
- **Memory**: Wissen, das man nicht aus dem Code ableiten kann und das über eine Spec hinaus gilt.
- **Globale Kriterien**: Anforderungen, die für jede Spec gelten, ohne dass sie jedes Mal neu hineingeschrieben werden.

## Der Ablauf

Grob passiert folgendes:

1. **Erfassen.** Aus einem Ticket oder einem Gespräch entsteht die Spec (`/flow-next:capture`, bei unklaren Anforderungen vorher `/flow-next:interview`). Was aus dem Ticket stammt, was der Nutzer gesagt hat und was der Agent geschlossen hat, wird markiert (`[paraphrase]`, `[inferred]`), siehe [`dev-starter-reliably-disable-groupware.md`](dev-starter-reliably-disable-groupware.md).
2. **Planen.** `/flow-next:plan` schneidet die Spec in Tasks und schreibt in die Spec, welche Task welche Anforderung abdeckt. Optional prüft ein Plan-Review durch ein zweites Modell den Schnitt, bevor Code entsteht.
3. **Umsetzen.** `/flow-next:work` arbeitet die Tasks ab, jede in eigenem Kontext. Am Ende jeder Task steht eine `Done summary` mit den Tests, die tatsächlich gelaufen sind.
4. **Review pro Task.** Ein Implementierungs-Review durch ein zweites Modell, oft eines anderen Anbieters (konfiguriert ist GPT über Codex). Das Ergebnis steht als `stage: impl-review` in der Task, inklusive `NEEDS_WORK`-Runden.
5. **Plan-Sync.** Nach jeder Task gleicht flow-next die nachfolgenden Tasks mit dem ab, was tatsächlich gebaut wurde. Hat sich ein Key-Name oder ein Methodenvertrag geändert, wird die nächste Task umgeschrieben. Die Spuren davon sieht man als `<!-- Updated by plan-sync: … -->` in [`tasks/fault-tolerant-print-locations.4.md`](tasks/fault-tolerant-print-locations.4.md).
6. **Completion Review.** Zum Schluss wird die ganze Spec gegen ihre Anforderungen und gegen die [globalen Kriterien](criteria.md) geprüft.

Nicht jede Spec läuft alle Schritte. Ein kleiner, klarer Umbau läuft als **eine** Task ohne Plan ("no-plan work item"): die Task sagt dann nur noch "prüf gegen die Anforderungen der Spec", siehe [`tasks/outbound-rest-shared-client.1.md`](tasks/outbound-rest-shared-client.1.md).

## Aufbau einer Spec

Die Abschnitte sind nicht in jeder Spec gleich, aber die tragenden Teile sind immer da:

| Abschnitt | Wozu |
|---|---|
| `Goal & Context` | Problem, Ziel, wer betroffen ist. Kein Lösungsweg. |
| `Architecture & Data Models` | Wer welchen Zustand besitzt (Ownership), oft mit einem Mermaid-Diagramm |
| `API Contracts` | Signaturen und Verhalten an den Schnittstellen, die sich ändern |
| `Edge Cases & Constraints` | Was die naheliegende Lösung kaputt machen würde. Hier steckt meist die meiste Arbeit. |
| `Acceptance Criteria` | Nummerierte Anforderungen `R1`, `R2`, …, jede mit einem `Errors:`-Teil für das Fehlerverhalten |
| `Early proof point` | Welche Task den Ansatz zuerst beweist, und was passiert, wenn sie scheitert |
| `Boundaries` | Was ausdrücklich nicht gemacht wird |
| `Decision Context` | Getroffene Entscheidungen mit verworfenen Alternativen, Datum und Entscheider |
| `Requirement coverage` | Welche Task welches `R` erfüllt |

Zwei Dinge sind uns dabei besonders wichtig:

**Prämissen werden verifiziert, nicht angenommen.** Jede Aussage darüber, was der bestehende Code heute tut, steht mit ihrer Prüfmethode in der Spec (gelesen, per Test bewiesen, per DB-Abfrage). Siehe `P1` bis `P7` in [`dev-starter-reliably-disable-groupware.md`](dev-starter-reliably-disable-groupware.md). Das ist keine Formalie. Die zwei teuersten Nacharbeiten eines früheren Features gingen beide auf Annahmen über bestehenden Code zurück, die niemand geprüft hatte. Daraus sind die Planungskriterien `G11` bis `G14` in [`criteria.md`](criteria.md) entstanden.

**Jede Anforderung hat ein Fehlerverhalten.** `Errors:` hinter jedem `R` zwingt dazu, den Fehlerfall mitzudenken. "Errors: no error surface beyond R1" ist eine erlaubte Antwort, aber eine bewusste.

## Aufbau einer Task

- **Frontmatter `satisfies`**: welche `R`s der Spec diese Task erfüllt.
- **`Size`, `Files`, `Touches`**: Umfang und welche Dateien die Task anfassen darf.
- **`Approach`**: der geplante Weg, mit Zeilenangaben und den Regeln, die gelten.
- **`Investigation targets`**: was der Agent lesen muss, bevor er anfängt.
- **`Acceptance`**: prüfbare Checkboxen, typischerweise konkrete Testfälle.
- **`Done summary`**: was tatsächlich gebaut wurde. Weicht der Agent vom geplanten Weg ab, steht das hier mit Begründung, und das Review entscheidet darüber. Ein Beispiel ist die Abweichung in [`tasks/fault-tolerant-print-locations.4.md`](tasks/fault-tolerant-print-locations.4.md), inklusive Mutationstest, der zeigt, dass die neuen Tests die Guards tatsächlich prüfen.
- **`Evidence`**: Commits, Testkommandos, PRs. Ehrlich, auch was nicht gelaufen ist ("Not run: live smoke …").

## Die Beispiele

| Datei | Was es zeigt |
|---|---|
| [`dev-starter-reliably-disable-groupware.md`](dev-starter-reliably-disable-groupware.md) + [Task](tasks/dev-starter-reliably-disable-groupware.1.md) | Bugfix aus einem Ticket. Conversation Evidence mit Quellenmarkierung, verifizierte Prämissen `P1`-`P7`, Defect Route in der Task |
| [`fault-tolerant-print-locations.md`](fault-tolerant-print-locations.md) + [Task .2](tasks/fault-tolerant-print-locations.2.md), [Task .4](tasks/fault-tolerant-print-locations.4.md) | Feature mit sieben Tasks. Ownership, API Contracts, Early proof point (Task .2), Requirement coverage, Plan-Sync-Kommentare und eine begründete Abweichung vom Plan (Task .4) |
| [`outbound-rest-shared-client.md`](outbound-rest-shared-client.md) + [Task](tasks/outbound-rest-shared-client.1.md) | Performance-Umbau über zwei Repos als eine Task ohne Plan. Implementierungsplan in der Spec, gemessene Evidenz, datierte Entscheidungen, Review über zwei Modelle |
| [`fix-order-header-supplier.md`](fix-order-header-supplier.md) | Eine Spec im Rohzustand: während einer anderen Arbeit erfasst, noch nicht geplant. Auch das ist eine legitime Spec. |
| [`criteria.md`](criteria.md) | Globale Akzeptanzkriterien. Jedes Kriterium ist eine Fehlerklasse, die tatsächlich vorkam |
| [`memory/`](memory/) | Zwei Memory-Einträge (siehe unten) |

## Memory

Memory ist für Wissen, das weder im Code noch in einer Regel steht und das eine spätere Session sonst neu herausfinden müsste. Jeder Eintrag endet deshalb mit dem Satz, **warum** er nicht aus dem Repo ableitbar ist. Ist er ableitbar, gehört er nicht in die Memory.

flow-next unterscheidet zwei Tracks:

- `bug/`: Post-mortems mit Problem, Ursache, Lösung, Vorbeugung. Bei uns bislang leer.
- `knowledge/`: kuratiertes Wissen mit `applies_when`, also wann ein Agent es heranziehen soll.

Die beiden Beispiele:

- [`domain-questions-decidable`](memory/knowledge/best-practices/domain-questions-decidable-2026-09-01.md): wie man eine fachliche Frage an die Fachseite stellt, damit die Antwort ein Satz sein kann. Und warum immer ein Freitext-Ausgang dazugehört: die gefährlichste Antwort ist "die Frage ist falsch gestellt".
- [`iris-read-via-jdbc-not-rest`](memory/knowledge/tooling-decisions/iris-read-via-jdbc-not-rest-2026-08-31.md): warum wir IRIS lesend per JDBC und nicht per Atelier REST abfragen. Die REST-Schnittstelle macht aus NULL eine `0`, und ein nacktes `COUNT(*)` über bestimmte Legacy-Tabellen liefert ein Vielfaches der echten Zeilenzahl.

## Was nicht im Auszug ist

- Die `.json`-Dateien neben jeder Spec und Task. Das ist der Zustand, den `flowctl` verwaltet (Status, Abhängigkeiten, Zeitstempel), nicht der Inhalt.
- `config.json`. Die für uns relevanten Einstellungen: Memory an, Plan-Sync an, Review-Backend Codex (GPT), maximal acht Review-Runden, Tracker-Sync aus.
- Review-Runs, Locks und Checkpoints.

Die Dateinamen sind gekürzt. Im Original beginnen Spec-IDs mit dem Ticket-Key oder einer laufenden Nummer (`fn-19-…`).
