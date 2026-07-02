# AGENTS.md – Arbeitsregeln für Entwicklungsagenten

Diese Datei definiert verbindliche Arbeitsregeln für KI-Agenten und Entwickler im
**neuen** Projekt `BuchungAtakan` (Mehrmandanten-Client-Server-Architektur).

Maßgebliche Quellen:
- **Autoritativ (Soll):** `docs/ENTWICKLUNGSPLAN.md`, `docs/PHASENUEBERSICHT.md`
- **Historisch (Referenz):** Altprojekt unter `C:\Users\User\PycharmProjects\BuchungAtakan`

Bei Widersprüchen gilt **immer** `docs/ENTWICKLUNGSPLAN.md`. Historische Festlegungen aus
dem Altprojekt sind nur als Referenz zu verwenden, nicht als aktuelle Vorgabe.

---

## 1. Architektur-Leitplanken (verbindlich)

- Zentrale **PostgreSQL**-Datenbank, zentrale **FastAPI** (Standort Aachen).
- **Kein direkter Datenbankzugriff** durch lokale Clients. Jeder Zugriff läuft über die
  authentifizierte API.
- **Mandantentrennung** über `unternehmen_id`. Jede fachliche Abfrage und jeder Schreibpfad
  muss sicher nach `unternehmen_id` gefiltert sein.
- **Dokumente bleiben lokal.** Original-PDFs (Rechnungen, Kontoauszüge, Anhänge) liegen auf
  dem lokalen `DATA_ROOT` des Standorts. Zentral werden nur strukturierte Daten, Metadaten,
  Hashes und **relative** Dokument-Locator gespeichert.
- Der lokale Client ist **PySide6**. Domänen- und Anwendungslogik bleibt **unabhängig** von
  PySide6 (Ports/Adapter), damit später eine andere UI andocken kann.
- Wiederverwendbare Fachlogik wandert nach `packages/core`; Infrastruktur (DB, Mail, Dateisystem)
  liegt hinter Adaptern in `apps/server` bzw. `apps/client`.

> Aufgehoben gegenüber Altprojekt: zentrale UNC-/Tailscale-Dokumentablage (Entscheidung
> 2026-06-21) ist **superseded**. Siehe `docs/OPEN_POINTS.md` → „Optional C“.

---

## 2. Code- und Liefer-Disziplin

- **Vollständige Dateien liefern.** Keine Auslassungen, keine `...`-Platzhalter in
  produktivem Code.
- Bestehende, funktionierende Fachlogik (EC/Telexpert, Gmail, Parser, Matching) wird **nicht**
  ohne nachgewiesenen Grund neu geschrieben. Wiederverwendung erfolgt **phasenweise**:
  alte Implementierung + Tests prüfen → wiederverwendbare Domänenlogik isolieren →
  Infrastrukturabhängigkeiten trennen → nach `packages/core` heben → über Ports/Adapter
  anbinden → Verhalten per Regressionstest absichern.
- Parser-Feinoptimierung (Rechnung/Kontoauszug) ist **Spätarbeit**. Nur blockierende
  Parserfehler werden sofort minimal behoben; übrige Befunde werden dokumentiert.
- Begriff **`Zuordnungen`** (nicht „Matches“) für fachliche Zuordnungsberichte/-ordner.
- `CardPayment` und `EC_Karte` sind **dieselbe** normalisierte EC-Kategorie.

---

## 3. Sicherheit und Datenschutz

- Echte Rechnungen, echte Kontoauszüge, Gmail-Zugangsdaten, OAuth-Schlüssel und
  Produktivdatenbanken werden **nie** in Chat, Git oder Projektdokumentation ausgegeben.
- Sensible Originaldateien werden nur **gelesen**; keine Umbenennung, kein Löschen,
  kein Überschreiben. Verarbeitet werden neutrale Kopien.
- Logs/Exporte enthalten **keine** sensiblen Originaldateinamen, -inhalte oder -hashes,
  sondern nur aggregierte Werte.
- Mandantenfremde Daten dürfen niemals sichtbar werden (Tenant-Isolation testen).

---

## 4. Datenbank- und Reset-Sicherheit

- **Produktivdaten dürfen niemals automatisch gelöscht werden.**
- Ein Datenbank-Reset erfolgt **nie stillschweigend**: nur über einen ausdrücklich benannten
  TEST-/Reset-Befehl mit vollständiger Pfadanzeige, klarer TEST-Modus-Kennzeichnung und
  Sicherheitsabfrage. Ein PRODUKTIONS-Modus muss den Reset **blockieren**.
- Datenmigration aus dem Altbestand (SQLite, Einzelmandant) erfolgt **nicht blind**: erst
  Inventar, Mapping, Tests und ausdrückliche Freigabe (siehe `docs/OPEN_POINTS.md` → D1).

---

## 5. Naming und Umgebungsvariablen

- Neues Projekt standardisiert auf **`BuchungAtakan`** und Präfix **`BUCHUNG_ATAKAN_*`**.
- Altnamen (`BuchungCumcu`, `BUCHUNG_CUMCU_*`) sind nur als **explizit dokumentierte,
  temporäre Kompatibilitäts-Fallbacks während der Migration** zulässig. Neuer Code und neue
  Dokumentation verwenden ausschließlich die neuen Namen.

---

## 6. Tests und Validierung

- Tests verwenden ausschließlich temporäre Datenbanken und Fakes – nie die Produktiv-DB.
- Reproduzierbarkeit: synthetische Testdaten müssen vollständig neu aufbaubar sein.
- Ein Testlauf darf bei Abweichungen nicht „grün“ klingen; er muss klar fehlschlagen und mit
  Nonzero-Exit enden.

---

## 7. Dokumentationspflichten

Bei jeder fachlich relevanten Änderung sind zu pflegen:
`CHANGELOG.md`, `docs/CURRENT_STATUS.md`, `docs/IMPLEMENTATION_HISTORY.md`,
`docs/TEST_HISTORY.md`, `docs/OPEN_POINTS.md` und – bei Schemaänderung – die
Architektur-/Datenmodellabschnitte in `docs/ARCHITECTURE.md`.

---

## 8. Aktueller Stand und Grenzen

- Das neue Repository enthält bisher **nur das technische Skelett** (siehe `docs/CURRENT_STATUS.md`).
- **Phase 1 / Anwendungscode ist nicht begonnen.** Vor Implementierungsbeginn gelten die
  Phase-0-Lieferobjekte aus `docs/ENTWICKLUNGSPLAN.md`.

---
*Sprache der Projektdokumentation: Deutsch. Technische Bezeichner, Codenamen und
API-Feldnamen dürfen englisch bleiben. (D5)*
*Stand: 2026-06-25.*
