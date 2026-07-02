<!-- AUTHORITY NOTE (manuell, nicht von GSD generiert) -->
> **Maßgeblich** sind `AGENTS.md` und `docs/ENTWICKLUNGSPLAN.md`. Diese Datei liefert nur
> GSD-Workflow-Kontext. Bei Widerspruch gewinnt der ENTWICKLUNGSPLAN. AGENTS.md zuerst lesen.

<!-- GSD:project-start source:PROJECT.md -->
## Project

**PROJECT — BuchungAtakan**

BuchungAtakan ist eine **Mehrmandanten-Buchhaltungs- und Dokumentabgleichsplattform**.
Sie importiert Rechnungen und Kontoauszüge, ordnet Rechnungen den passenden Bankbuchungen zu,
verarbeitet EC-/Telexpert-Sammeleinzahlungen und holt fehlende Belege per Gmail nach.

Technischer **Neuaufbau**: vom lokalen Einzelmandanten-Desktop (SQLite) hin zu zentraler
Client-Server-Architektur (lokale Windows-Clients + zentrale FastAPI/PostgreSQL in Aachen).

Nutzer:
- Mitarbeitende je Unternehmen über einen **lokalen Windows-Client (PySide6)**.
- Später authentifizierter Web-Zugriff (z. B. Steuerberater).

**Core Value:** > Rechnungen, Bankbuchungen und Belege werden je Unternehmen sicher getrennt verarbeitet und
> zuverlässig einander zugeordnet — Originaldokumente bleiben lokal, zentral liegen nur die
> strukturierten Daten.
<!-- GSD:project-end -->

<!-- GSD:stack-start source:STACK.md -->
## Technology Stack

Technology stack not yet documented. Will populate after codebase mapping or first phase.
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->

<!-- KARPATHY-GUIDELINES:project-start -->
## Karpathy-Inspired Working Rules

Diese Regeln ergänzen die projektspezifischen Vorgaben. Sie überschreiben weder `AGENTS.md`
noch `docs/ENTWICKLUNGSPLAN.md`.

### 1. Think Before Coding

- Keine Annahmen über Datenbanknamen, Rollen, Migrationsstand, Dateipfade,
  Umgebungsvariablen, Feature-Flags, installierte Pakete oder API-Verhalten.
- Vor Änderungen tatsächlichen Zustand prüfen: Branch, Arbeitsbaum, relevante Modelle,
  Migrationen, Tests und Konfiguration.
- Wichtige Annahmen ausdrücklich nennen.
- Bei mehreren möglichen Interpretationen nicht still eine auswählen.
- Widersprüche, fehlende Informationen und Trade-offs vor der Implementierung melden.
- Bei Unklarheit stoppen statt mit einer Vermutung weiterzuarbeiten.
- Vor jeder schreibenden Datenbankaktion `current_database()` und aktive Rolle prüfen.
- Niemals aus einer vorhandenen URL ableiten, dass Realtest- oder Produktivdaten verändert
  werden dürfen.

### 2. Simplicity First

- Nur die kleinste Änderung umsetzen, die das aktuelle Paket und seine Akzeptanzkriterien erfüllt.
- Keine spekulativen Features, zusätzlichen Abstraktionsschichten, Konfigurierbarkeit oder
  Fallbacks hinzufügen, die nicht verlangt wurden.
- Bestehende Parser-, Matching-, Repository-, Service-, API- und Testmuster wiederverwenden.
- Funktionierende Fachlogik nicht ohne nachgewiesenen Grund neu schreiben.
- Eine klare Implementierung mehreren halb genutzten Abstraktionen vorziehen.
- Wenn die Lösung deutlich größer als das Problem wird: stoppen und vereinfachen.

### 3. Surgical Changes

- Nur Dateien ändern, die direkt für die aktuelle Phase oder das aktuelle Paket nötig sind.
- Keine angrenzenden Refactorings, Formatierungsänderungen, Umbenennungen oder historischen
  Dokumentationsänderungen ohne direkten Bezug zum Auftrag.
- Bestehenden Projektstil beibehalten.
- Jede geänderte Zeile muss auf Nutzerauftrag, Plan, Requirement, Test oder notwendigen
  Kompatibilitätsfix zurückführbar sein.
- Nur Imports, Variablen, Funktionen oder Dateien entfernen, die durch die eigene Änderung
  unbenutzt wurden.
- Bereits vorhandenen Dead Code nicht ungefragt löschen.
- Niemals committen:
  - `.idea/workspace.xml`
  - `CODEX_PHASE5_REVIEW_BEFUNDE.txt`
  - echte Rechnungen, Kontoauszüge, Gmail-Exporte oder Produktivdaten
  - Credentials oder URLs mit Credentials
  - echte Hashes oder sensible Dateinamen

### 4. Goal-Driven Execution

Vor der Implementierung einen kurzen Plan mit überprüfbaren Punkten nennen:

1. Welche Änderung wird vorgenommen?
2. Welche Dateien sollen sich voraussichtlich ändern?
3. Welcher gezielte Test beweist das Verhalten?
4. Welche Regressionstests folgen?
5. Welche Kriterien bedeuten „fertig“?

Bei Fehlern:

- Fehler nach Möglichkeit zuerst mit einem gezielten Test reproduzieren.
- Die kleinste verantwortliche Schicht korrigieren.
- Den gezielten Test erneut ausführen.
- Danach die relevante Regressionssuite ausführen.
- Die vollständige Suite erst nach grünen gezielten Tests starten.

Ein Auftrag ist erst abgeschlossen, wenn:

- die verlangte Funktion implementiert ist,
- gezielte Tests grün sind,
- die relevante Gesamtsuite grün ist,
- keine fremden Dateien im Diff liegen,
- keine echten Daten oder Secrets enthalten sind,
- kein Push ohne ausdrückliche Freigabe erfolgt ist.

### 5. Paket- und Phasengrenzen

- Ausschließlich das ausdrücklich genannte Paket bearbeiten.
- Keine Funktionen späterer Pakete vorziehen.
- Keine zusätzlichen Migrationen, Refactorings oder Architekturänderungen außerhalb des Plans.
- Eine Phase nicht als abgeschlossen melden, solange notwendige Tests, Realtests,
  Requirement-Bewertungen oder die fachliche Abnahme offen sind.

### 6. Datenbank- und Testdisziplin

- `buchungatakan_test` ist die einzige erlaubte Pytest-Datenbank.
- `buchungatakan_realtest` darf niemals `TEST_DATABASE_URL` sein und niemals automatisch
  bereinigt oder truncated werden.
- `buchungatakan` darf niemals Testziel sein.
- `buchungatakan_app` ist nur Laufzeit-/DML-Rolle, niemals DDL- oder Alembic-Rolle.
- `buchungatakan_migrations` ist für Migrationen und DDL zuständig.
- `postgres` nur für einmalige lokale Reparaturen nach ausdrücklicher Freigabe verwenden.
- Eine Testsuite nur dann als vollständig melden, wenn Integrationstests tatsächlich laufen
  und `collected`, `passed`, `skipped` und `failed` berichtet wurden.
- Migrationstests früherer Phasen dürfen nicht erwarten, dass ihre Revision dauerhaft Head ist;
  sie prüfen stattdessen Existenz, Abstammung und benötigte Schemaobjekte.

### 7. Stop-Regel für Infrastruktur-Loops

Nach höchstens zwei fehlgeschlagenen Infrastruktur-Reparaturversuchen:

1. keine weiteren schreibenden Änderungen,
2. nur lesende Diagnose,
3. aktuellen Zustand und Blocker dokumentieren,
4. keine Owner-, Rollen-, Grant- oder Schemaänderung improvisieren,
5. Nutzerentscheidung abwarten.

### 8. Terminal- und Secret-Disziplin

- PowerShell-, SQL- und Python-Blöcke klar trennen.
- Python-Code niemals als PowerShell-Befehl ausgeben.
- Keine Secrets im Chat oder in Logs ausgeben.
- Keine ausführbaren Befehle mit ungeklärten Platzhaltern darstellen.
- Lokale URLs aus `.env` ableiten, ohne sie auszugeben.
- Vor jedem Commit ausführen:
  - `git status --short`
  - `git diff --name-only`
  - `git diff --check`

### 9. Headroom

Bei jedem Claude-Code-Auftrag ausdrücklich angeben, ob Headroom aktiviert oder deaktiviert ist.
Für dieses Projekt gilt ohne abweichende Nutzeranweisung: **Headroom deaktiviert**.
<!-- KARPATHY-GUIDELINES:project-end -->
