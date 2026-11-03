# Agentic Engineering bei SHD

Begleit-Repo zum Vortrag auf dem InterSystems DACH Symposium am 4. November 2026.

Das hier ist ein Auszug aus dem agentischen Setup unseres ERP-Systems. Das ERP ist 24 Jahre alt, läuft auf InterSystems IRIS und hat rund 4,7 Mio. Zeilen Java. Zwei Teams entwickeln daran, mit Claude Code und Codex als Agenten und flow-next als Spec- und Task-Workflow.

Die Werkzeuge wechseln schnell. Die Artefakte und der Prozess bleiben.

Deshalb zeigt dieses Repo keine Tool-Konfiguration, sondern die Artefakte selbst: wie eine Regel aussieht, wie ein Review-Agent aussieht, wie ein Skill aussieht und wie eine Spec aussieht.

## Wie die Teile zusammenspielen

- **Rules** legen fest, was gilt.
- **Skills** legen fest, wie Arbeit erledigt wird.
- **Agents** legen fest, wer reviewt.
- **Specs** legen fest, was gebaut wird.

Der Router ([`rules/AGENTS-excerpt.md`](rules/AGENTS-excerpt.md)) sagt dem Agenten, welche Regeln er für welche Aufgabe laden muss. Ein Skill wie `erp-review` wählt anhand der geänderten Dateien die passenden Review-Agenten aus, lässt sie parallel laufen und führt ihre Ergebnisse zu einem Bericht zusammen. Die Agenten prüfen gegen dieselben Regeln, die beim Schreiben des Codes galten.

## Inhalt

| Ordner | Inhalt |
|---|---|
| [`rules/`](rules/) | Router-Auszug aus `AGENTS.md` und fünf Regeln: wie Regeln geschrieben werden (011), DB-Migrationen (142), Logging (213), i18n (221), Client-Binding (701) |
| [`agents/`](agents/) | Zwei Review-Agenten (Silent Failures, Transaktionsgrenzen) und der gemeinsame Review-Vertrag in `_review-shared/`: Schweregrade, Finding-Format, Lens-Map. Wie die Reviewer arbeiten und was die Lenses bedeuten: [`agents/README.md`](agents/README.md) |
| [`skills/`](skills/) | `erp-review` (Review-Fan-out, nur Bericht) und `erp-review-loop` (der einzige Skill, der mechanische Fixes selbst anwendet) |
| [`guides/`](guides/) | Projekt-Guide zur InterSystems IRIS MCP-Integration: IRIS-SQL und Systemdaten als read-only MCP-Tools für Claude Code und Codex, lokal per Docker (AI Hub EAP) und als Zwischenlösung über die Atelier REST API |
| [`specs/`](specs/) | flow-next-Specs mit ihren Tasks (Bugfix, Feature mit sieben Tasks, Performance-Umbau, Spec im Rohzustand), globale Akzeptanzkriterien und zwei Memory-Einträge. Erklärt in [`specs/README.md`](specs/README.md) |

Die Skills verweisen auf `_review-shared/`. Im Original liegt das unter `.claude/skills/`, hier unter `agents/`.

Im Original sind die Regeln `.mdc`-Dateien. Hier heißen sie `.md`, damit GitHub sie rendert.

## Hinweis zu den Auszügen

Die Dateien sind gekürzt und neutralisiert. Produktname, Ticket-Keys, Kunden, Hostnamen, Personen und produktinterne Klassennamen sind entfernt oder verallgemeinert (aus dem Produktnamen wird schlicht "ERP"), Codebeispiele sind synthetisch. Format und Aufbau entsprechen dem Original, der Wortlaut der Regeln weitgehend. Die Artefakte selbst sind auf Englisch, wie im Original.

## Kontakt

Florian Steitz, SHD
