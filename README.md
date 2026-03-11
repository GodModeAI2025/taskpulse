# TaskPulse — Agent Task Infrastructure

TaskPulse ist eine Dateisystem-Konvention, die Agents eine strukturierte Planungs-, Dokumentations- und Selbstorganisationsschicht gibt. Kein CLI, kein Binary, keine Datenbank. Nur Markdown-Dateien mit YAML-Frontmatter in einem `.taskpulse/`-Verzeichnis.

## Modi

- **Solo-Modus**: Ein einzelner Agent plant, arbeitet und dokumentiert sequentiell
- **Orchestrator-Modus**: Ein übergeordneter Agent verteilt Arbeit an Subagents, die parallel operieren — kollisionsfrei durch partitionierte ID-Ranges und isolierte Schreibbereiche

## Dateien

- `SKILL.md` — Vollständige Skill-Dokumentation und Instruktionen
- `references/examples.md` — Referenz-Beispiele und Edge Cases

## Nutzung

TaskPulse wird als Skill in Claude Cowork / Claude Code eingebunden. Die `SKILL.md` enthält alle Instruktionen, die ein Agent benötigt, um TaskPulse selbstständig zu nutzen.
