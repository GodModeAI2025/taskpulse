# TaskPulse Referenz — Agent-Nutzungsmuster

## Inhaltsverzeichnis

1. Solo: Code-Agent bearbeitet Feature-Request
2. Orchestrator: Drei Subagents arbeiten parallel
3. Orchestrator: Dispatch-Beispiel vollständig
4. Orchestrator: Blocker-Auflösung zwischen Subagents
5. Orchestrator: Range-Erschöpfung und Erweiterung
6. Fortführung über Konversationsgrenzen
7. ID-Vergabe-Algorithmus (beide Modi)
8. Edge Cases

---

## 1. Solo: Code-Agent bearbeitet Feature-Request

Mensch sagt: "Implementiere API-Rate-Limiting"

Agent erstellt Task, arbeitet, dokumentiert Entscheidungen, findet Bug, erstellt neuen Task, schließt ab, schreibt Log.

`.taskpulse/TP-001.md` bei Abschluss:
```markdown
---
id: TP-001
title: "API-Rate-Limiting implementieren"
type: feature
status: done
priority: high
agent: "code-agent"
labels: ["api", "security"]
epic: ""
parent: ""
blocked_by: []
output: ["src/middleware/rate-limiter.ts", "src/middleware/rate-limiter.test.ts"]
created: "2026-03-08"
updated: "2026-03-08"
---

## Plan

1. [x] Bestehende Middleware-Architektur analysieren
2. [x] Rate-Limiter Middleware erstellen
3. [x] Redis-Backend für verteiltes Rate-Limiting
4. [x] HTTP 429 Response mit Rate-Limit-Headern
5. [x] Tests schreiben

## Ergebnis

Token-Bucket-Middleware. 100 Requests/Minute pro API-Key. Redis-Backend. HTTP 429 mit Retry-After Header.

## Entscheidungen

- **Token-Bucket statt Sliding Window**: Einfacher, Redis-freundlicher. Präzisionsverlust akzeptabel.
```

Nebenbei erstellter Bug: `.taskpulse/TP-002.md`
```markdown
---
id: TP-002
title: "Auth-Middleware ignoriert OPTIONS-Requests"
type: bug
status: backlog
priority: medium
agent: ""
...
---

CORS Preflight-Requests durchlaufen Auth-Middleware. Gefunden während TP-001.
```

---

## 2. Orchestrator: Drei Subagents arbeiten parallel

Mensch sagt: "Mach ein komplettes Security-Audit unserer API"

### Phase 1 — Orchestrator erstellt Auftrags-Tasks

Orchestrator nutzt Range 001–099:

```
TP-001 (epic)     → "API Security Audit"                status: in_progress
TP-002 (task)     → "Authentifizierung prüfen"           status: ready, agent: ""
TP-003 (task)     → "Autorisierung und RBAC prüfen"      status: ready, agent: ""
TP-004 (task)     → "Input-Validierung prüfen"           status: ready, agent: ""
TP-005 (decision) → "Audit-Bericht zusammenstellen"      status: backlog, blocked_by: [TP-002, TP-003, TP-004]
```

### Phase 2 — Orchestrator vergibt Ranges und dispatcht

`config.yml` aktualisiert:
```yaml
orchestrator:
  id_range_size: 50
  next_range_start: 250
  active_ranges:
    - agent: "auth-agent"
      start: 100
      end: 149
      assigned: "2026-03-08"
      status: active
    - agent: "rbac-agent"
      start: 150
      end: 199
      assigned: "2026-03-08"
      status: active
    - agent: "input-agent"
      start: 200
      end: 249
      assigned: "2026-03-08"
      status: active
```

Orchestrator setzt die Auftrags-Tasks:
```
TP-002 → agent: "auth-agent",  status: in_progress
TP-003 → agent: "rbac-agent",  status: in_progress
TP-004 → agent: "input-agent", status: in_progress
```

### Phase 3 — Subagents arbeiten parallel

**auth-agent** (Range 100–149) erstellt:
```
TP-100 → "JWT-Implementierung prüfen"       parent: TP-002, status: done
TP-101 → "Session-Management analysieren"    parent: TP-002, status: done
TP-102 → "Finding: JWT ohne Expiry"          parent: TP-002, type: finding, status: done
```

**rbac-agent** (Range 150–199) erstellt:
```
TP-150 → "Rollen-Hierarchie analysieren"     parent: TP-003, status: done
TP-151 → "Endpoint-Permissions prüfen"       parent: TP-003, status: in_progress
TP-152 → "Finding: Admin-Endpoint ohne Auth" parent: TP-003, type: finding, status: done
```

**input-agent** (Range 200–249) erstellt:
```
TP-200 → "SQL-Injection Tests"               parent: TP-004, status: done
TP-201 → "XSS-Prüfung"                      parent: TP-004, status: done
TP-202 → "Finding: Unescaped User-Input"     parent: TP-004, type: finding, status: done
```

Keine ID-Kollisionen. Kein Agent berührt Tasks eines anderen.

### Phase 4 — Orchestrator konsolidiert

Orchestrator liest alle Subagent-Logs und Task-Stati:
```
auth-agent:  3/3 done ✓
rbac-agent:  2/3 done, 1 in_progress
input-agent: 3/3 done ✓
```

Orchestrator wartet auf rbac-agent, dann:
- TP-002, TP-003, TP-004 → status: done
- TP-005 (Audit-Bericht) → Blocker aufgelöst, status: in_progress
- Orchestrator erstellt Audit-Bericht aus allen Findings

---

## 3. Orchestrator: Dispatch-Beispiel vollständig

Prompt den der Orchestrator an einen Subagent sendet:

```
Du arbeitest an Aufgabe TP-002: "Authentifizierung prüfen"

Nutze TaskPulse für Planung und Dokumentation.
Arbeitsverzeichnis: /home/projekt/.taskpulse/
TaskPulse-Instruktionen: /mnt/skills/user/taskpulse/SKILL.md

Dein Agent-Name: auth-agent
Deine ID-Range: TP-100 bis TP-149
Erstelle Sub-Tasks nur innerhalb dieser Range.

Schreibregeln:
- Schreibe NUR auf Tasks mit deiner ID-Range (TP-100 bis TP-149) oder auf TP-002 (dein Auftrags-Task, nur Ergebnis-Abschnitt)
- Schreibe dein Log nach: .taskpulse/log/2026-03-08_auth-agent.md
- Lies alle Tasks, aber ändere nur deine eigenen
- Bei Blockern: setze status blocked und blocked_by, dann arbeite an anderem Task weiter

Auftrags-Task lesen: .taskpulse/TP-002.md

Kontext: API-Security-Audit. Fokus auf Authentifizierungsmechanismen.
Prüfe JWT-Implementierung, Session-Management, Passwort-Hashing, MFA-Status.
Dokumentiere Findings als separate Tasks mit type: finding.
```

---

## 4. Orchestrator: Blocker-Auflösung zwischen Subagents

**Situation**: `input-agent` findet dass ein Endpoint den `rbac-agent` prüfen sollte, gar keine Input-Validierung hat.

**input-agent** erstellt in eigener Range:
```markdown
---
id: TP-203
title: "Cross-Agent: /admin/users Endpoint ohne Input-Validierung"
type: finding
status: done
priority: critical
agent: "input-agent"
labels: ["cross-agent"]
parent: "TP-004"
...
---

Endpoint /admin/users akzeptiert beliebige JSON-Payloads ohne Schema-Validierung.
Betrifft auch RBAC-Prüfung (TP-003/rbac-agent) — Endpoint ist möglicherweise ungeschützt.
```

**Orchestrator** liest `labels: ["cross-agent"]`, erkennt Querverbindung, erstellt in eigener Range:
```markdown
---
id: TP-006
title: "/admin/users Endpoint — kombiniertes Security-Problem"
type: bug
status: ready
priority: critical
agent: ""
labels: ["cross-agent", "security"]
parent: "TP-001"
blocked_by: []
...
---

Aus Finding TP-203 (input-agent) und TP-152 (rbac-agent):
Endpoint hat weder Input-Validierung noch Auth-Prüfung. Kritisch.
```

Orchestrator entscheidet wer den Fix übernimmt.

---

## 5. Orchestrator: Range-Erschöpfung und Erweiterung

**Situation**: `code-agent` hat Range 100–149 und erreicht TP-149.

**code-agent** Log-Eintrag:
```markdown
## Range-Status
ID-Range 100-149 erschöpft. Kann keine neuen Tasks erstellen.
Bestehende Tasks werden weiter bearbeitet.
Neue Range beim Orchestrator angefordert.
```

**Orchestrator** erkennt die Meldung (oder sieht dass der Agent keine neuen Tasks mehr erstellt), vergibt neue Range:

```yaml
# config.yml Update
active_ranges:
  - agent: "code-agent"
    start: 100
    end: 149
    status: exhausted
  - agent: "code-agent"
    start: 250
    end: 299
    assigned: "2026-03-08"
    status: active
```

Orchestrator informiert code-agent über neue Range.

---

## 6. Fortführung über Konversationsgrenzen

Neue Konversation, neuer Agent. Liest Board-State:

**Solo-Modus:**
1. `config.yml` laden
2. Alle `.taskpulse/*.md` Frontmatter scannen
3. `in_progress`-Tasks identifizieren → Plan-Checkboxen zeigen Fortschritt
4. Agent setzt bei nächstem offenen Schritt fort

**Orchestrator-Modus:**
1. `config.yml` laden — `active_ranges` zeigen welche Subagents existierten
2. Alle Tasks scannen — Fortschritt pro Range/Agent aggregieren
3. Logs lesen — letzte Einträge pro Agent
4. Entscheiden: Offene Auftrags-Tasks neu dispatchen oder selbst bearbeiten

---

## 7. ID-Vergabe-Algorithmus (beide Modi)

### Solo-Modus
```
1. Lese config.yml → prefix
2. Scanne .taskpulse/*.md UND archive/*.md
3. Höchste Nummer + 1
4. Format: dreistellig (001–999), ab 1000 vierstellig
```

### Orchestrator-Modus — Orchestrator
```
1. Nutze Range 001–099
2. Scanne nur Tasks in eigener Range
3. Nächste freie Nummer in 001–099
```

### Orchestrator-Modus — Subagent
```
1. Kenne eigene Range (z.B. 100–149) aus Dispatch-Prompt
2. Scanne .taskpulse/*.md — filtere auf eigene Range
3. Nächste freie Nummer innerhalb der Range
4. Falls Range erschöpft: im Log melden, auf neue Range warten
```

IDs werden nie wiederverwendet.

---

## 8. Edge Cases

### Board existiert nicht
Agent erstellt `.taskpulse/`, `config.yml`, `log/`, `archive/` selbstständig.

### Subagent findet kein Board
Subagent muss `.taskpulse/` vorfinden (Orchestrator hat es erstellt). Falls nicht: Fehler im Log melden, Arbeit mit reiner Konversation fortsetzen.

### Subagent kennt seine Range nicht
Fehler im Dispatch. Subagent erstellt keine Tasks, arbeitet nur am zugewiesenen Auftrags-Task, meldet Problem im Log.

### Orchestrator stürzt ab
Tasks bleiben im Dateisystem. Neuer Orchestrator kann aus `config.yml` und Task-Frontmatter den Zustand rekonstruieren. `active_ranges` zeigen welche Subagents existierten, Task-Status zeigt Fortschritt.

### Subagent schreibt auf fremde Tasks
Verletzung des Protokolls. Der Schreibschutz ist konventionsbasiert, nicht technisch erzwungen. Bei der nächsten Konsolidierung erkennt der Orchestrator inkonsistente `agent`-Felder und kann korrigieren.

### Sehr viele parallele Subagents (>5)
Ranges verkleinern (z.B. `id_range_size: 20`). Orchestrator-Log wird wichtiger für Übersicht. Board-Status-Anzeige gruppiert nach Agent.

### Verschachtelte Orchestrierung
Ein Subagent kann theoretisch selbst Orchestrator werden und Sub-Sub-Agents starten. Eigene Range wird zur Orchestrator-Range, Sub-Ranges werden daraus geschnitzt. In der Praxis selten nötig und nicht empfohlen wegen Komplexität.

### Solo-Agent will nachträglich Subagents nutzen
Wechsel zu Orchestrator-Modus: `orchestrator`-Sektion in `config.yml` anlegen. Bestehende Tasks (sequentiell vergeben) liegen wahrscheinlich in Range 001–0XX — Orchestrator deklariert diese als seine eigene Range und vergibt Subagent-Ranges ab der nächsten freien Hunderterstelle.
