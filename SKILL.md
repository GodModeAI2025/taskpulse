---
name: taskpulse
description: Agent-Infrastruktur für Aufgabenplanung, Fortschrittsdokumentation und Selbstorganisation — mit Orchestrator-Modus für parallele Subagents. IMMER verwenden wenn ein Agent Arbeit planen, dokumentieren, priorisieren, nachverfolgen oder Subagents koordinieren muss. Auch bei Teilaufgaben zerlegen, Fortschritt festhalten, Entscheidungen protokollieren, Blocker melden, Abhängigkeiten verwalten, Ergebnisse zusammenführen. Triggers sind Task anlegen, Fortschritt dokumentieren, was steht an, nächster Schritt, Zwischenstand, Blocker melden, Entscheidungslog, Arbeitsprotokoll, Sprint planen, Backlog pflegen, Board-Status, Roadmap, Subagents koordinieren, parallele Arbeit verteilen. Dateibasiert per Konvention — keine API, kein CLI, keine Installation.
---

# TaskPulse — Agent Task Infrastructure

TaskPulse ist eine Dateisystem-Konvention die Agents eine strukturierte Planungs-, Dokumentations- und Selbstorganisationsschicht gibt. Kein CLI, kein Binary, keine Datenbank. Nur Markdown-Dateien mit YAML-Frontmatter in einem `.taskpulse/`-Verzeichnis.

TaskPulse funktioniert in zwei Modi:

- **Solo-Modus**: Ein einzelner Agent plant, arbeitet und dokumentiert sequentiell
- **Orchestrator-Modus**: Ein übergeordneter Agent verteilt Arbeit an Subagents die parallel operieren — kollisionsfrei durch partitionierte ID-Ranges und isolierte Schreibbereiche

---

## Wer nutzt TaskPulse

Nicht der Mensch direkt. TaskPulse ist die Infrastruktur für **andere Skills und Agents**. Ein Agent der eine komplexe Aufgabe bearbeitet — Code schreiben, Dokument erstellen, Analyse durchführen, Recherche machen — nutzt TaskPulse um:

- Die Aufgabe in Teilschritte zu zerlegen
- Fortschritt festzuhalten (auch über Konversationsgrenzen hinweg)
- Entscheidungen und Begründungen zu protokollieren
- Blocker und Abhängigkeiten sichtbar zu machen
- Dem Menschen jederzeit einen Statusüberblick geben zu können
- Arbeitsergebnisse an Tasks zu verknüpfen
- Subagents parallel arbeiten zu lassen ohne Konflikte

Der Mensch sieht das Board wenn er möchte, aber die primären Konsumenten und Produzenten sind Agents.

---

## Verzeichnisstruktur

```
projekt/
├── .taskpulse/
│   ├── config.yml              # Projektkonfiguration
│   ├── TP-001.md               # Task
│   ├── TP-002.md               # Task
│   ├── log/                    # Agent-Logs (ein File pro Agent pro Tag)
│   │   ├── 2026-03-08_orchestrator.md
│   │   ├── 2026-03-08_code-agent.md
│   │   └── 2026-03-08_test-agent.md
│   └── archive/                # Erledigte Tasks
│       └── TP-000.md
└── ROADMAP.md                  # Generierte Roadmap (optional)
```

Wichtig: Log-Dateien sind **pro Agent partitioniert** (`YYYY-MM-DD_{agent-name}.md`). Kein Agent schreibt in das Log eines anderen. Der Orchestrator konsolidiert bei Bedarf.

---

## Konfiguration: config.yml

Wird beim ersten Zugriff angelegt. Wenn ein Agent TaskPulse nutzen will und `.taskpulse/` nicht existiert, erstellt er es selbstständig.

```yaml
project:
  name: "Projektname"
  prefix: "TP"
  created: "2026-03-08"

defaults:
  type: task
  status: backlog
  priority: medium

statuses:
  - backlog
  - ready
  - in_progress
  - blocked
  - review
  - done
  - archived

types:
  - task
  - bug
  - feature
  - epic
  - spike
  - decision
  - finding

priorities:
  - critical
  - high
  - medium
  - low

labels: []

# Orchestrator-Konfiguration (nur im Orchestrator-Modus relevant)
orchestrator:
  id_range_size: 50          # IDs pro Subagent-Partition
  next_range_start: 100      # Nächste verfügbare Range-Startposition
  active_ranges: []          # Wird zur Laufzeit vom Orchestrator befüllt
```

---

## Task-Datei: Schema

```markdown
---
id: TP-001
title: "Kurzer, prägnanter Titel"
type: task
status: backlog
priority: medium
agent: ""
labels: []
epic: ""
parent: ""
blocked_by: []
output: []
created: "2026-03-08"
updated: "2026-03-08"
---

Beschreibung und Kontext.

## Plan

Zerlegung in Arbeitsschritte (vom Agent befüllt).

1. [ ] Schritt 1
2. [ ] Schritt 2
3. [ ] Schritt 3

## Ergebnis

Zusammenfassung des Arbeitsergebnisses (vom Agent nach Abschluss befüllt).

## Entscheidungen

Begründungen für Designentscheidungen während der Bearbeitung.

- **Entscheidung**: Warum so und nicht anders.
```

### Felder

| Feld         | Typ    | Pflicht | Beschreibung                                           |
|--------------|--------|---------|--------------------------------------------------------|
| `id`         | string | ja      | `{prefix}-{nummer}`, dreistellig                       |
| `title`      | string | ja      | Max 80 Zeichen                                         |
| `type`       | enum   | ja      | task, bug, feature, epic, spike, decision, finding     |
| `status`     | enum   | ja      | backlog, ready, in_progress, blocked, review, done     |
| `priority`   | enum   | ja      | critical, high, medium, low                            |
| `agent`      | string | nein    | Name/Kennung des bearbeitenden Agents oder Skills      |
| `labels`     | list   | nein    | Freie Tags                                             |
| `epic`       | string | nein    | Übergeordnete Epic-ID                                  |
| `parent`     | string | nein    | Eltern-Task bei Zerlegung                              |
| `blocked_by` | list   | nein    | IDs blockierender Tasks                                |
| `output`     | list   | nein    | Pfade zu Arbeitsergebnissen (Dateien, Artefakte)       |
| `due`        | date   | nein    | Fälligkeitsdatum                                       |
| `created`    | date   | ja      | ISO-Datum                                              |
| `updated`    | date   | ja      | ISO-Datum, bei jeder Änderung aktualisieren            |

---

## Betriebsmodi

### Erkennung des Modus

TaskPulse erkennt den Modus automatisch:

- **Solo-Modus**: Ein einzelner Agent arbeitet. Kein Orchestrator, keine parallelen Subagents. Einfache sequentielle ID-Vergabe.
- **Orchestrator-Modus**: Ein übergeordneter Agent spawnt Subagents (z.B. via Claude Code Subagents, Cowork Tasks, oder vergleichbare Mechanismen). Der Orchestrator verteilt Arbeit, vergibt ID-Ranges, konsolidiert Ergebnisse.

Die Entscheidung trifft der Agent der TaskPulse initialisiert. Wenn er Subagents spawnen wird → Orchestrator-Modus. Wenn er allein arbeitet → Solo-Modus. Ein Wechsel zur Laufzeit ist möglich: Ein Solo-Agent der merkt dass er Subagents braucht, wechselt in den Orchestrator-Modus indem er die `orchestrator`-Sektion in `config.yml` aktiviert.

---

## Solo-Modus

### Agent-Protokoll

#### Bei Arbeitsbeginn

1. **Board prüfen**: Existiert `.taskpulse/`? Falls nein → anlegen mit `config.yml`
2. **Kontext laden**: Alle `.taskpulse/*.md` scannen, nur Frontmatter lesen (Token-effizient)
3. **Bestehende Tasks prüfen**: Gibt es bereits einen Task für diese Aufgabe?
4. **Haupttask erstellen**: Falls neu, Task mit type, priority und Beschreibung anlegen
5. **Zerlegung**: Komplexe Aufgaben in Sub-Tasks zerlegen (`parent`-Feld verknüpfen)

#### Während der Arbeit

6. **Status aktualisieren**: `in_progress` setzen, `agent`-Feld befüllen
7. **Plan-Checkboxen abhaken**: Im Task-Body `[ ]` → `[x]` wenn Schritte erledigt
8. **Entscheidungen protokollieren**: Im Entscheidungen-Abschnitt festhalten
9. **Blocker melden**: `status: blocked`, `blocked_by` setzen, Begründung im Body
10. **Neue Tasks erstellen**: Wenn weitere Aufgaben auftauchen

#### Bei Abschluss

11. **Ergebnis dokumentieren**: Zusammenfassung im Ergebnis-Abschnitt
12. **Output verknüpfen**: Pfade zu erstellten Dateien im `output`-Feld
13. **Status**: `done` setzen
14. **Log schreiben**: Zusammenfassung in `log/YYYY-MM-DD_{agent-name}.md`

#### ID-Vergabe im Solo-Modus

Sequentiell: Höchste bestehende ID + 1. Kein Konfliktpotenzial weil nur ein Agent schreibt.

---

## Orchestrator-Modus

Der Orchestrator-Modus löst die fundamentalen Probleme paralleler Agents: ID-Kollisionen, Status-Races, Log-Konflikte und unkontrollierte Schreibzugriffe.

### Architektur

```
┌─────────────────────────────────────┐
│           ORCHESTRATOR              │
│                                     │
│  • Vergibt ID-Ranges               │
│  • Erstellt Auftrags-Tasks          │
│  • Prüft Abhängigkeiten            │
│  • Konsolidiert Ergebnisse          │
│  • Schreibt Orchestrator-Log        │
│  • Einziger der config.yml ändert   │
└──────┬──────────┬──────────┬────────┘
       │          │          │
  ┌────▼────┐ ┌──▼─────┐ ┌─▼───────┐
  │ Agent A │ │Agent B │ │ Agent C │
  │ ID-Range│ │ID-Range│ │ ID-Range│
  │ 100-149 │ │150-199 │ │ 200-249 │
  │         │ │        │ │         │
  │ Eigenes │ │Eigenes │ │ Eigenes │
  │ Log     │ │Log     │ │ Log     │
  └─────────┘ └────────┘ └─────────┘
```

### Prinzipien

1. **Partitionierte IDs**: Jeder Subagent bekommt einen exklusiven ID-Bereich. Keine zwei Agents können dieselbe ID erzeugen.
2. **Isolierte Schreibbereiche**: Ein Subagent schreibt NUR auf Tasks innerhalb seiner ID-Range oder auf Tasks die ihm explizit per `agent`-Feld zugewiesen sind.
3. **Partitionierte Logs**: Jeder Agent schreibt sein eigenes Log-File. Kein gemeinsames Log.
4. **Orchestrator-Exklusivrechte**: Nur der Orchestrator darf `config.yml` ändern, ID-Ranges vergeben und Tasks zwischen Agents umverteilen.
5. **Read-All, Write-Own**: Jeder Agent darf alle Tasks lesen, aber nur seine eigenen schreiben.

### Orchestrator-Protokoll

#### Phase 1 — Initialisierung

1. Board erstellen oder prüfen (`.taskpulse/`, `config.yml`)
2. Aufgabe analysieren und in verteilbare Einheiten zerlegen
3. Für jede Einheit: Auftrags-Task erstellen (im Orchestrator-eigenen ID-Bereich 001–099)
4. ID-Ranges für Subagents festlegen und in `config.yml` registrieren

#### Phase 2 — Dispatch

5. Subagent-Aufträge formulieren — jeder Auftrag enthält:
   - Die Task-ID des Auftrags-Tasks
   - Die zugewiesene ID-Range
   - Den Agent-Namen
   - Den Pfad zu `.taskpulse/`
   - Die Anweisung TaskPulse zu nutzen (Verweis auf SKILL.md)

**Dispatch-Template für Subagent-Prompt:**

```
Du arbeitest an Aufgabe {task-id}: "{title}"

Nutze TaskPulse für Planung und Dokumentation.
Arbeitsverzeichnis: {projekt}/.taskpulse/
TaskPulse-Instruktionen: {pfad-zu-taskpulse}/SKILL.md

Dein Agent-Name: {agent-name}
Deine ID-Range: {prefix}-{start} bis {prefix}-{end}
Erstelle Sub-Tasks nur innerhalb dieser Range.

Schreibregeln:
- Schreibe NUR auf Tasks mit deiner ID oder deinem Agent-Namen
- Schreibe dein Log nach: log/YYYY-MM-DD_{agent-name}.md
- Lies alle Tasks, aber ändere nur deine eigenen
- Bei Blockern: setze status: blocked und blocked_by, dann arbeite an anderem Task weiter

Auftrags-Task lesen: .taskpulse/{task-id}.md
```

6. Subagents spawnen

#### Phase 3 — Monitoring (bei lang laufenden Subagents)

7. Periodisch `.taskpulse/*.md` Frontmatter scannen
8. Fortschritt pro Subagent aggregieren (Tasks in_progress, done, blocked)
9. Blocker identifizieren und wenn möglich auflösen
10. Bei Bedarf: neue Tasks zuweisen oder Ranges erweitern

#### Phase 4 — Konsolidierung

11. Prüfen ob alle Subagent-Tasks `done` oder `blocked` sind
12. Subagent-Logs lesen und konsolidiertes Log schreiben:
    `log/YYYY-MM-DD_orchestrator.md`
13. Ergebnisse zusammenführen — Auftrags-Tasks mit Ergebnissen der Sub-Tasks aktualisieren
14. `output`-Felder der Auftrags-Tasks mit den Outputs der Sub-Tasks befüllen
15. Dem Menschen den Gesamtstatus berichten

### ID-Range-Verwaltung

Der Orchestrator reserviert und vergibt Ranges:

```yaml
# In config.yml — vom Orchestrator verwaltet
orchestrator:
  id_range_size: 50
  next_range_start: 250
  active_ranges:
    - agent: "code-agent"
      start: 100
      end: 149
      assigned: "2026-03-08"
      status: active
    - agent: "test-agent"
      start: 150
      end: 199
      assigned: "2026-03-08"
      status: active
    - agent: "docs-agent"
      start: 200
      end: 249
      assigned: "2026-03-08"
      status: active
```

**Regeln:**

- Range 001–099 ist für den Orchestrator selbst reserviert (Auftrags-Tasks, Epics, Decisions)
- Jeder Subagent bekommt standardmäßig 50 IDs (konfigurierbar über `id_range_size`)
- Wenn ein Subagent seine Range erschöpft: Orchestrator vergibt neue Range
- Ein Subagent kennt nur seine eigene Range und erstellt IDs innerhalb dieser Range sequentiell
- Subagent-ID-Vergabe: Scanne eigene Range, nächste freie Nummer nehmen

### Subagent-Protokoll

Ein Subagent der vom Orchestrator gestartet wird:

1. **Auftrag lesen**: Auftrags-Task aus `.taskpulse/` laden
2. **Eigene Range kennen**: Aus dem Dispatch-Prompt
3. **Arbeitsplanung**: Sub-Tasks innerhalb eigener Range erstellen
4. **Arbeit ausführen**: Normal arbeiten, Status aktualisieren, Entscheidungen protokollieren
5. **Nur eigene Tasks ändern**: Kein Schreibzugriff auf Tasks außerhalb der eigenen Range
6. **Eigenes Log schreiben**: `log/YYYY-MM-DD_{agent-name}.md`
7. **Auftrags-Task aktualisieren**: Ergebnis im Auftrags-Task dokumentieren (Ausnahme: der Auftrags-Task liegt außerhalb der eigenen Range — Subagent darf den Body-Abschnitt "Ergebnis" des ihm zugewiesenen Auftrags-Tasks schreiben)
8. **Blocker melden**: `status: blocked` setzen — der Orchestrator löst

### Konfliktauflösung

| Situation | Lösung |
|-----------|--------|
| Zwei Subagents brauchen dieselbe Datei | Orchestrator erstellt separaten Task pro Agent, Ergebnis-Merge durch Orchestrator |
| Subagent findet Bug der anderen Agent betrifft | Bug-Task in eigener Range erstellen, `labels: ["cross-agent"]` setzen |
| Subagent ist fertig, anderer blockiert | Subagent setzt eigene Tasks auf `done`, Orchestrator entscheidet über Blocker |
| Range erschöpft | Subagent stoppt Task-Erstellung, meldet im Log. Orchestrator vergibt neue Range |
| Subagent stürzt ab / antwortet nicht | Orchestrator erkennt stale Tasks (in_progress ohne Update seit >X Minuten), kann Tasks reassignen |

### Stale-Detection

Der Orchestrator erkennt verwaiste Tasks:

- Task ist `in_progress` aber `updated`-Datum liegt >30 Minuten zurück
- Kein Log-Eintrag des zugewiesenen Agents seit >30 Minuten
- Orchestrator kann: Task auf `backlog` zurücksetzen, `agent` leeren, neu zuweisen

---

## Entscheidungslog: log/

Logs sind **pro Agent partitioniert**. Dateinamens-Konvention:

```
log/YYYY-MM-DD_{agent-name}.md
```

Beispiele:
- `log/2026-03-08_orchestrator.md`
- `log/2026-03-08_code-agent.md`
- `log/2026-03-08_test-agent.md`

### Agent-Log-Format

```markdown
# 2026-03-08 — code-agent

## Bearbeitete Tasks
- TP-103: Auth-Middleware Refactoring, Schritt 1-3 erledigt
- TP-105: Login-Bug analysiert, Root Cause gefunden

## Neue Tasks (in eigener Range)
- TP-115: Bug — Sonderzeichen im Passwort

## Entscheidungen
- TP-103: JWT statt Session-Cookies

## Blocker
- TP-114: Wartet auf TP-152 (test-agent)
```

### Orchestrator-Log-Format

Das Orchestrator-Log enthält zusätzlich die konsolidierte Sicht:

```markdown
# 2026-03-08 — orchestrator

## Dispatch
- code-agent: Range 100-149, Aufträge TP-010, TP-011
- test-agent: Range 150-199, Auftrag TP-012

## Subagent-Fortschritt
- code-agent: 5/8 Tasks done, 1 blocked, 2 in_progress
- test-agent: 3/4 Tasks done, 1 in_progress

## Konsolidierte Ergebnisse
- TP-010 (Auth-Refactoring): Abgeschlossen. 3 Sub-Tasks, alle done.
- TP-012 (Integration-Tests): 75% fertig, TP-155 noch in Arbeit.

## Blocker aufgelöst
- TP-114: Abhängigkeit zu TP-152 entfernt (test-agent hat Workaround gefunden)

## Offene Blocker
- Keine

## Nächste Schritte
- TP-012 Abschluss abwarten
- Ergebnisse zusammenführen und dem Menschen berichten
```

---

## Aufgabenzerlegung

Funktioniert gleich in beiden Modi. Das `parent`-Feld verknüpft hierarchisch:

```
TP-010 (epic)    → "Sicherheitsaudit Q2"          [orchestrator, Range 001-099]
├── TP-011       → "Penetrationstest planen"       [orchestrator, Auftrag an code-agent]
│   ├── TP-103   → "Testplan schreiben"            [code-agent, Range 100-149]
│   └── TP-104   → "Tooling evaluieren"            [code-agent, Range 100-149]
├── TP-012       → "OWASP-Checklist"               [orchestrator, Auftrag an test-agent]
│   ├── TP-150   → "Top-10 Vulnerabilities prüfen" [test-agent, Range 150-199]
│   └── TP-151   → "Dependency-Scan ausführen"     [test-agent, Range 150-199]
└── TP-013       → "Audit-Bericht"                 [orchestrator, blocked_by: TP-011, TP-012]
```

Der Orchestrator erstellt die Auftrags-Tasks (TP-011, TP-012, TP-013). Die Subagents erstellen ihre Sub-Tasks in der eigenen Range.

---

## Prioritätslogik für Agents

Gilt in beiden Modi. Wenn ein Agent autonom entscheiden muss, welchen Task er als nächstes bearbeitet:

1. **Blocker auflösen**: Tasks die andere Tasks blockieren, zuerst
2. **Critical Bugs**: Immer vor Features
3. **Abhängigkeitsketten**: Tiefste unerledigte Abhängigkeit zuerst
4. **High → Medium → Low**: Bei gleicher Ebene nach Priorität
5. **Älteste zuerst**: Bei gleicher Priorität nach Erstellungsdatum

Im Orchestrator-Modus: Subagents priorisieren nur innerhalb ihrer zugewiesenen Tasks. Der Orchestrator priorisiert auf Gesamtprojekt-Ebene.

---

## Board-Status für den Menschen

Wenn der Mensch fragt "wie ist der Stand" oder "was passiert gerade":

### Solo-Modus

```
## TaskPulse — Projektname (8 aktive Tasks)

### In Progress (2)
- [TP-003] ●● Refactor Auth-Modul → 3/5 Schritte
- [TP-005] ●  Fix Login-Bug → Root Cause gefunden

### Blocked (1)
- [TP-014] ●  Audit-Bericht → wartet auf TP-012, TP-013

### Backlog (3)
- [TP-001] ●●● API-Rate-Limiting
- [TP-004] ○  Docs aktualisieren
- [TP-006] ●  Performance-Tests
```

### Orchestrator-Modus

```
## TaskPulse — Projektname (Orchestrator-Modus, 3 Agents aktiv)

### Gesamtfortschritt
Aufträge: 3 | Erledigt: 1 | In Arbeit: 2 | Blockiert: 0

### code-agent (Range 100-149, 12 Tasks)
- 8 done, 3 in_progress, 1 blocked
- Aktuell: [TP-108] ●● API-Middleware — Schritt 4/6
- Blockiert: [TP-114] wartet auf test-agent/TP-152

### test-agent (Range 150-199, 7 Tasks)
- 4 done, 2 in_progress, 1 backlog
- Aktuell: [TP-155] ● Integration-Test Auth-Flow

### Aufträge (Orchestrator)
- [TP-010] ●● Auth-Refactoring → done ✓
- [TP-011] ●●● API-Security → in_progress (code-agent 70%, test-agent 60%)
- [TP-012] ●  Dokumentation → in_progress (docs-agent 40%)
```

Prioritäts-Indikatoren: ●●● critical, ●● high, ● medium, ○ low

---

## ID-Vergabe

### Solo-Modus

1. Lese `config.yml` → `prefix`
2. Scanne `.taskpulse/*.md` UND `.taskpulse/archive/*.md`
3. Höchste Nummer + 1
4. Dreistellig mit führenden Nullen (001–999), ab 1000 vierstellig

### Orchestrator-Modus

1. Orchestrator nutzt Range 001–099 für eigene Tasks
2. Subagents nutzen exklusiv ihre zugewiesene Range
3. Innerhalb der Range: Scanne eigene Tasks, nächste freie Nummer
4. Formatierung: identisch zum Solo-Modus

IDs werden in keinem Modus wiederverwendet.

---

## Roadmap generieren

Auf Anfrage eine `ROADMAP.md` im Projektstamm erstellen. Struktur: Aktiv → Ready → Geplant (nach Epics gruppiert) → Kürzlich erledigt (letzte 7 Tage). Im Orchestrator-Modus: zusätzlich Auftrags-Tasks mit Subagent-Fortschritt anzeigen.

---

## Konventionen

- Dateinamen = ID: `TP-001.md`
- `updated` bei jeder Änderung aktualisieren
- Leere optionale Felder: `""` oder `[]`
- Datumsformat: `YYYY-MM-DD`
- UTF-8, YAML-Frontmatter zwischen `---`
- Log-Dateien: `log/YYYY-MM-DD_{agent-name}.md` (nie geteilte Logs)
- Ein Agent erstellt `.taskpulse/` selbstständig — keine Bestätigung nötig
- Ein Agent erstellt und aktualisiert Tasks selbstständig — keine Bestätigung nötig
- Menschliche Bestätigung nur bei: Löschen von Tasks, Ändern der config.yml, Archivierung von >5 Tasks gleichzeitig
- Im Orchestrator-Modus: Nur der Orchestrator ändert `config.yml`
- Subagents schreiben nur innerhalb ihrer ID-Range und auf ihnen zugewiesene Tasks

---

## Nutzung durch aufrufende Skills

### Solo-Agent

```markdown
## Arbeitsplanung
Nutze TaskPulse für Aufgabenzerlegung und Fortschrittsdokumentation.
Lies bei Bedarf die Instruktionen in `/pfad/zu/taskpulse/SKILL.md`.
Arbeitsverzeichnis: `.taskpulse/` im Projektstamm.
```

### Orchestrator mit Subagents

```markdown
## Arbeitsplanung
Nutze TaskPulse im Orchestrator-Modus.
Lies die Instruktionen in `/pfad/zu/taskpulse/SKILL.md`.
Arbeitsverzeichnis: `.taskpulse/` im Projektstamm.
Vergib ID-Ranges an Subagents und nutze das Dispatch-Template aus der SKILL.md.
```

Der aufrufende Skill muss TaskPulse nicht vollständig in seinen eigenen Prompt laden. Er verweist auf die SKILL.md und liest sie bei Bedarf.

Für die Referenz-Dokumentation mit Beispielen und Edge Cases siehe `references/examples.md`.

---

## Sprache

Instruktionen auf Deutsch. Task-Inhalte in der Sprache des Projekts/Benutzers.
