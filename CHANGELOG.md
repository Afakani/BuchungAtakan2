# Changelog

Format nach [Keep a Changelog](https://keepachangelog.com/de/1.1.0/).
Dieses Projekt ist der technische Neuaufbau von `BuchungAtakan` als
Mehrmandanten-Client-Server-System.

> **Hinweis:** Der vollständige Funktionsverlauf des Vorgängersystems steht unter
> [Altprojekt (Referenz)](#altprojekt-referenz) und in
> `docs/IMPLEMENTATION_HISTORY.md`.

---

## [Unreleased] — Phase 7 Pakete 07-01..07-04 + Gap-Pakete HYB-07/Testinfrastruktur, Paket 07-06 Pre-Realtest-Härtung

**Branch:** `phase/07-hybride-rechnungserkennung-qualitaetspruefung`
**GSD-Phase:** 7 | **ENTWICKLUNGSPLAN-Phase:** 6 (Pakete 07-01..07-04, Gap-Pakete HYB-07 + Testinfrastruktur, Paket 07-06)
**Anforderungen:** HYB-01..24 vollständig erfüllt (Details in `.planning/REQUIREMENTS.md`)
**Phase 7: formal abnahmefähig** (Commit `cd81fbd`) — vollständige Suite mit `TEST_DATABASE_URL` + `APP_DATABASE_URL` gegen `buchungatakan_test`: 1088 passed, 1 skipped (P-10), 0 failed. Paket 07-06 ist eine zusätzliche Härtung **nach** dieser Abnahme, keine Korrektur der Abnahme selbst.

### Paket 07-06-PRE-REALTEST-QUALITAETSHAERTUNG

Ausgelöst durch einen Read-only-Audit von Phase 7: zwei bestätigte Bugs (Faktor-1000-Betragsfehler, unmögliche Kalenderdaten passieren als OK) und eine tote Prüfung (Währungskonsistenz konnte nie fehlschlagen) behoben; Testlücke bei den Plausibilitätsregeln geschlossen.

**Fixed:**

- **packages/core/invoice_extraction/betrag_utils.py** (neu) — zentrale, abhängigkeitsfreie `bereinige_betrag_string()`; erkennt deutsches Tausenderformat ohne Dezimalteil (`"1.000"` → `1000`, `"2.500"` → `2500`), `"0.500"` bleibt korrekt `0.50`. `invoice_parser.py::_parse_amount` und `hybrid_normalisierung.py::normalisiere_betrag` nutzen dieselbe Funktion — keine Divergenz mehr, keine private Parser-Methode mehr aus Hybrid-Code aufgerufen.
- **packages/core/invoice_extraction/invoice_parser.py** — `_extract_date_from_string()` validiert jetzt mit `datetime.date(...)`; `31.02.`/`29.02.` (Nicht-Schaltjahr) werden abgelehnt, gültige Formate inkl. Schaltjahre bleiben erkannt.
- **packages/core/invoice_extraction/hybrid_plausibilitaet.py** — `pruefe_rechnungsdatum_plausibel()` validiert ebenfalls per `datetime.date(...)` statt nur den Jahresbereich.
- **packages/core/invoice_extraction/hybrid_auswertung.py** — `_berechne_plausibilitaet()` prüft Währungskonsistenz jetzt gegen **alle** normalisierten Currency-Kandidaten (nicht nur den Auswahl-Gewinner); ein Providerwiderspruch (EUR/USD) blockiert `OK` nachweislich und bleibt auditierbar.

**Added:**

- **tests/unit/test_hybrid_plausibilitaet.py** (neu, 33 Tests) — direkte Tests für alle 7 Plausibilitätsregeln inkl. Toleranzgrenzen, Storno-Fällen, Schaltjahren.
- **tests/unit/test_hybrid_normalisierung.py** — 12 neue Tests (Tausenderformat, Kalendervalidierung).
- **tests/unit/test_hybrid_auswertung_anti_false_ok.py** (neu, 10 Tests) — End-to-End-Tests durch die volle Auswertungskette: Happy Path, ungültiges Datum, Faktor-1000, Währungskonflikt, fehlende/widersprüchliche Pflichtfelder, Determinismus.

**Nicht geändert (für Realtest-Auswertung zurückgestellt):** Netto-/Steuer-Pflichtlogik, automatisches OK bei aufgelösten Mehrheitskonflikten, Confidence-Kalibrierung, Schwellenwerte, Azure, PaddleOCR-Konfiguration, alte Status-Engine, Architektur.

**Realtest-Vorgaben (dokumentiert):** invoice2data bewusst aktivieren; PaddleOCR nur mit lokal gecachten Modellgewichten + nachgewiesenem No-Network; jeder Lauf mit stabiler `erkennungslauf_id` (sonst keine Idempotenz, Statistik verfälscht).

**Verifiziert:** Unit+Security 910 passed, 4 skipped, 0 failed (vorher 855 — 55 neue Tests). Volle Suite ohne DB: 911 passed, 233 skipped, 0 failed. InvoiceParser-Regression (golden/adapter/extraction-result): 38 passed.

---

### Gap-Paket Testinfrastruktur — App-Rollen-Fixtures für echte RLS-Isolationstests

**Befund:** Der erste DB-Verifikationslauf für HYB-07 (siehe unten) zeigte 3 fehlgeschlagene Tenant-Isolationstests. Ursache: `test_hybrid_extraktion.py::test_tenant_isolation_hybrid_erkennungslaeufe`, `test_hybrid_auswertung_persistenz.py::test_tenant_isolation` und `test_hybrid_audit_bewertungen.py::test_haba_tenant_b_sieht_bewertungen_von_tenant_a_nicht` liefen über `TEST_DATABASE_URL` (`db_session`/`sync_engine`), das in dieser Konfiguration als PostgreSQL-Superuser verbindet — Superuser bypassen RLS grundsätzlich, auch mit FORCE ROW LEVEL SECURITY. Kein Policy-Fehler, sondern eine Testarchitektur-Lücke (fehlende Non-Superuser-Fixture).

**Added:**

- **tests/conftest.py** — `APP_DB_URL` (aus `APP_DATABASE_URL`), `_validate_app_db_url()` (P-10-Skip ohne die Variable; Namensguard exakt `buchungatakan_test`; lehnt eine `APP_DATABASE_URL` ab, die identisch mit `TEST_DATABASE_URL` ist — verhindert blinde Wiederverwendung einer potenziell privilegierten URL), `_pruefe_rolle_nicht_privilegiert()` (Laufzeitprüfung gegen `pg_roles.rolsuper`/`rolbypassrls`, harter `pytest.fail()` statt falsch-grünem RLS-Nachweis bei privilegierter Rolle), neue Fixtures `app_role_sync_engine`, `app_role_async_engine`, `app_role_session`.
- Die drei genannten Tests auf die neue Non-Superuser-Fixture umgestellt. Scharfe Assertions (`assert row is None`, "Tenant B darf Tenant A nicht sehen") unverändert beibehalten — keine Umwandlung in tolerante Datenintegritätstests.
- **tests/integration/test_rls.py** — Skip-Begründung präzisiert: `test_rls_blocks_benutzer_without_tenant_context` bleibt ein **dauerhafter** Skip, auch mit korrekt konfigurierter `APP_DATABASE_URL`. Grund: Migration 003 entzieht `buchungatakan_app` jeden Direktzugriff auf `benutzer` (REVOKE ALL) — Auth läuft ausschließlich über SECURITY-DEFINER-Funktionen. Eine direkte Query auf `benutzer` unter dieser Rolle scheitert mit `permission denied`, das wäre kein RLS-Nachweis.

**Scope-Grenzen (eingehalten):** keine RLS-Policy-Änderung, keine Migrationsänderung, keine neue Rolle/Datenbank angelegt, kein Passwort gesetzt, keine Hybrid-Fachlogik/Provider-Flags/Azure/Gmail/UI berührt.

**Offener manueller Schritt (P-09, unverändert):** `buchungatakan_app` wird von `apply_migrations` bereits automatisch angelegt (`CREATE ROLE ... LOGIN`, ohne Passwort) — ein Passwort muss weiterhin einmalig der DB-Admin setzen, damit `APP_DATABASE_URL` sich authentifizieren kann. Doku-Inkonsistenz vermerkt: `docs/OPEN_POINTS.md` P-09 beschreibt die Rollenanlage als rein manuellen Schritt, tatsächlich übernimmt `apply_migrations` das bereits (nur das Passwort bleibt manuell).

**Verifiziert (Nutzerlauf, `TEST_DATABASE_URL` + `APP_DATABASE_URL` gegen `buchungatakan_test`):** 1088 passed, 1 skipped (P-10), 0 failed, 81,45 s. Die drei Tenant-Isolationstests liefen scharf über die Non-Superuser-App-Rolle: Tenant B sieht Tenant-A-Daten nicht.

---

### Gap-Paket HYB-07 — Hybridpipeline-Flag-Gate

**Added:**

- **packages/core/invoice_extraction/hybrid_auswertung.py** — `werte_lokale_erkennung_aus()` gatet sich selbst über das bereits vorhandene `ist_hybrid_pipeline_aktiviert()`: bei `BUCHUNG_ATAKAN_FEATURE_HYBRID_PIPELINE=False` (Default) wirft die Funktion vor jeder Normalisierungs-/Konflikt-/Auswahl-/Statuslogik die neue Exception `HybridPipelineDeaktiviertFehler`; bei `True` läuft die Auswertung unverändert weiter.
- **packages/core/invoice_extraction/__init__.py** — `HybridPipelineDeaktiviertFehler` exportiert.
- **tests/unit/test_hybrid_auswertung.py** — 10 neue Unit-Tests: Default/explizit `false` blockiert, `true` erlaubt, Mock-Beweis dass bei deaktiviertem Flag keine nachgelagerte Auswertungslogik erreicht wird, AST-Strukturbeleg (Gate ist erste Anweisung der Funktion), Unabhängigkeit des Pipeline-Flags von den Provider-Flags invoice2data/PaddleOCR/Azure (je einzeln getestet, keine automatische Aktivierung, keine Beeinflussung bestehender Provider-Flag-Auswertung).

**Fixed:**

- **.planning/REQUIREMENTS.md**, **docs/CURRENT_STATUS.md** — HYB-05, HYB-07, HYB-08 von TEILWEISE auf ERFÜLLT korrigiert. HYB-05/HYB-08 waren bereits seit Paket 07-03 fachlich erfüllt (Konflikterkennung, `auswahl_grund`) — die Statusdokumente waren veraltet, kein Code dafür in diesem Gap-Paket geändert. HYB-07 war der einzige echte Implementierungs-/Test-Gap: `ist_hybrid_pipeline_aktiviert()` existierte, wurde aber von keinem Aufrufer genutzt.

**Scope-Grenzen (eingehalten):** keine neuen Provider, keine Azure-Aktivierung, keine Änderung an `local_provider_orchestrator.py` oder den separaten Provider-Flags, kein Gmail, keine UI, keine echten Rechnungen/Produktivdaten.

**Verifiziert:** Unit+Security gesamt (ohne TEST_DATABASE_URL) 855 passed, 4 skipped, 0 failed. Vollständige Suite mit `TEST_DATABASE_URL` + `APP_DATABASE_URL` gegen `buchungatakan_test`: 1088 passed, 1 skipped (P-10), 0 failed (siehe Gap-Paket Testinfrastruktur oben — die drei ursprünglich fehlgeschlagenen Tenant-Isolationstests liefen nach dessen Fix scharf grün).

---

### Paket 07-04 — Azure-Adapter vorbereitet, aber deaktiviert (HYB-20..24)

**Added:**

- **packages/core/invoice_extraction/azure_document_intelligence_provider.py** — `AzureDocumentIntelligenceClient` (Protocol, kein SDK), `AzureAnalyseErgebnis`/`AzureFeldErgebnis` (synthetische Dataclasses, keine SDK-Typen), `AzureDocumentIntelligenceProvider` (implementiert `CloudInvoiceProvider`, fail-closed: bei `enabled=False` kein Clientaufruf), `FakeAzureDocumentIntelligenceClient`, `erstelle_provider()` (Factory — `client_factory` wird bei deaktiviertem Flag nie aufgerufen). Mapping deterministisch, auf `RECHNUNGS_FELDER` begrenzt, nutzt bestehende `normalisiere_kandidat()`. Clientfehler liefern nur sanitisierten Fehlertyp.
- **apps/server/config.py** — kanonisches Flag `BUCHUNG_ATAKAN_FEATURE_AZURE_DOCUMENT_INTELLIGENCE` (default `false`); Legacy-Flag `BUCHUNG_ATAKAN_FEATURE_AZURE_DI` (07-01) bleibt bewusst inert.
- **packages/core/invoice_extraction/hybrid_qualitaetsentscheidung.py** — Azure-only-Statusguard: Pflichtfelder ausschließlich durch Azure-Kandidaten bestätigt → höchstens `MUSS_GEPRUEFT_WERDEN`, nie `OK`.
- **packages/core/invoice_extraction/__init__.py** — Azure-Exporttypen ergänzt.
- **tests/unit/test_azure_document_intelligence_provider.py** — 17 neue Unit-Tests (Flag/Factory fail-closed, Fake-Client-Mapping, Determinismus, Feldbegrenzung, Confidence-Validierung, sanitisierte Fehlercodes).
- **tests/security/test_azure_document_intelligence_sanitization.py** — 11 neue Security-Tests (kein SDK-Import, kein `DocumentAnalysisClient`/`AzureKeyCredential`, keine Secrets/Endpunkte, `.env.example` unverändert).
- **tests/unit/test_hybrid_qualitaetsentscheidung.py**, **tests/unit/test_feature_flags.py**, **tests/security/test_hybrid_auswertung_sanitization.py** — je 2–3 neue Tests für Azure-only-Guard, kanonisches Flag und die präzisierte Azure-Ausnahme im Statusmodul.

**Sicherheitsregeln (eingehalten):** kein Azure-SDK, kein Netzwerkzugriff, keine Credentials, keine echte Aktivierung; Adapter ist nicht produktionsfähig — spätere Aktivierung braucht ein eigenes Paket und ausdrückliche Nutzerfreigabe (`docs/OPEN_POINTS.md` HYB-O-02).

**Verifiziert gegen `buchungatakan_test`:** 61 gezielte Azure-/Feature-Flag-/Status-/Security-Tests passed; vollständige Suite 1078 passed, 1 skipped (P-10), 0 failed.

**Korrektur während der Umsetzung:** der Azure-only-Statusguard war zunächst fälschlich als „HYB-24" beschriftet. `07-04-PLAN.md` zeigt, dass HYB-24 tatsächlich das Aktivierungs-/Freigabeverfahren ist (`docs/OPEN_POINTS.md` HYB-O-02), nicht die Statuslogik. Der Guard ist ein allgemeines 07-04-Akzeptanzkriterium ohne eigene HYB-Nummer — alle betroffenen Kommentare/Docstrings korrigiert.

---

### Paket 07-03B — Lokaler Excel-Export der Qualitätsstatistik (HYB-19)

**Added:**

- **packages/core/invoice_extraction/hybrid_statistik_export.py** — `erstelle_hybrid_statistik_workbook_bytes(gruppen, dimensionen)`: erzeugt `.xlsx`-Bytes vollständig im Speicher (kein DB-, kein Dateisystemzugriff) mit drei Sheets `Uebersicht`/`Statistik`/`Metriken`; nur aggregierte Statistikdaten, Prozentformat auf Quoten-Spalten, `None`-Quote als leere Zelle, leere Statistik erzeugt gültiges Workbook mit Status `LEER`.
- **apps/server/routers/hybrid.py** — `GET /api/v1/hybrid/statistik/export.xlsx`; Queryvalidierung (`_dimensionen_aus_query`) aus dem bestehenden JSON-Statistikendpoint extrahiert und von beiden Endpunkten geteilt statt dupliziert; `Content-Disposition` mit festem Dateinamen, `Cache-Control: no-store`, korrekter Excel-MIME-Type.
- **pyproject.toml** — `openpyxl==3.1.5` reproduzierbar ergänzt.
- **tests/unit/test_hybrid_statistik_export.py** — 16 neue Unit-Tests (Sheet-Struktur, Spaltenreihenfolge, Prozentformat, Determinismus, kein Dateisystemzugriff).
- **tests/security/test_hybrid_statistik_export_sanitization.py** — 6 neue Security-Tests (synthetisch sensible Strings, Anbieter nur als angeforderter Gruppenschlüssel).
- **tests/integration/test_hybrid_statistik_export_api.py** — 7 neue Integrationstests (HTTP 200, Header, `openpyxl.load_workbook`, Queryparität mit JSON-Endpoint, Tenant-Isolation, leere Statistik).

**Sicherheitsregeln (eingehalten):** keine Rechnungsnummern/-daten/Einzelbeträge/Empfängerdaten/Dateinamen/Pfade/Hashes/Volltexte/OCR-Rohdaten/Kandidaten-Rohwerte/Secrets im Workbook; Anbietername nur als aggregierter Gruppenschlüssel, keine Pseudonymisierung in diesem Paket; kein Azure, kein Gmail, keine echten Rechnungen, kein Push.

---

### Paket 07-03A — Audit-Booleanmodell und Qualitätsstatistik (HYB-17/18)

**Added:**

- **migrations/versions/012_hybrid_audit_bewertungen.py** — nullable `dokument_klasse` auf `hybrid_statusentscheidungen`; neue Tabelle `hybrid_audit_bewertungen` (append-only, RLS, FORCE RLS, Check-Constraint auf `DokumentKlasse`-Werte).
- **packages/core/invoice_extraction/hybrid_audit.py** — `AuditBewertungFelder` (6 strukturierte Booleanfelder), `berechne_gesamtergebnis_korrekt()` (abgeleitet, nicht gespeichert), `waehle_neueste_bewertung()` (bewertet_am DESC, id DESC).
- **packages/core/invoice_extraction/hybrid_statistik.py** — Vollständigkeits-, Richtigkeits-, Automatisierungs-, Prüf-, Nichterkennungsquote je Gruppe; Nenner 0 → `null`.
- **apps/server/routers/hybrid.py** — `POST/GET /api/v1/hybrid/statusentscheidungen/{id}/auditbewertungen[/latest]`, `GET /api/v1/hybrid/statistik`.

Details siehe `docs/CURRENT_STATUS.md` §Paket 07-03A und Commit `00abaaa042acdd472b679eb7bfe3969a513695e2`.

---

### Paket 07-02 — Lokale Erkennungsschienen (abgeschlossen)

**Added:**

- **packages/core/invoice_extraction/document_classifier.py** — `DokumentKlasse` (TEXT_PDF/SCAN_PDF/SONDERFORMAT/UNBEKANNT), deterministische Schwellenklassifikation.
- **packages/core/invoice_extraction/local_text.py** — `LocalExtractedText`: einheitliches Text-/Seitenmodell (Text, Seite, Quelle, Confidence, Methode, Provider-Version, HYB-12).
- **packages/core/invoice_extraction/invoice2data_provider.py** — `Invoice2DataProvider`: nutzt bereits extrahierten pdfplumber-Text statt invoice2data-eigenem Inputreader (vermeidet poppler-Systemabhängigkeit); Template-Version/Anbieterkennung je Kandidat nachvollziehbar; unbekanntes Layout liefert kontrolliertes Leerergebnis; Templatefehler isoliert.
- **config/invoice2data_templates/synthetisch_muster_ag_v1.yml** — synthetisches Test-Template, keine echten Rechnungsinhalte.
- **packages/core/invoice_extraction/paddleocr_provider.py** — `PaddleOCRProvider`: Lazy Import, kein Netzwerk/Cloud-Fallback; PaddleOCR im Repo nicht installiert, Adapter vollständig implementiert und mit Fake-Engine getestet.
- **packages/core/invoice_extraction/local_provider_orchestrator.py** — `LocalInvoiceProviderOrchestrator`: wählt aktivierte lokale Provider nach Dokumentklasse und Feature-Flag, isoliert Providerfehler zusätzlich, trifft keine finale fachliche Auswahl, instanziiert nie einen Cloud-Provider.
- **tests/unit/test_invoice_document_classifier.py**, **test_invoice2data_provider.py**, **test_paddleocr_provider.py**, **test_local_invoice_provider_orchestrator.py** — 47 neue Unit-Tests.
- **tests/security/test_invoice_provider_sanitization.py** — Security-Tests: keine echten Pfade/Volltexte/Credentials in Fehlern, keine `unternehmen_id` auf Provider-Ebene, kein Netzwerkzugriff.
- **pyproject.toml** — optionale Abhängigkeiten `invoice2data==1.0.0` (installiert), `paddleocr` (nicht installiert, nur deklariert).

**HYB-09..13 Akzeptanzkriterien:**

| ID | Status |
|----|--------|
| HYB-09 | **ERFÜLLT** — pdfplumber-Pfad unverändert übernommen |
| HYB-10 | **ERFÜLLT** — invoice2data-Templates versioniert, synthetisch getestet |
| HYB-11 | **ERFÜLLT** — PaddleOCR lokal integrierbar, gleiche Parser-/Validierungspipeline |
| HYB-12 | **ERFÜLLT** — einheitliche Textübergabe über `LocalExtractedText` |
| HYB-13 | **ERFÜLLT** — Providerfehler isoliert, sanitisiert, blockieren andere Provider nicht |

**Tests (vom Nutzer ausgeführt, vollständiger Lauf mit `TEST_DATABASE_URL` auf `buchungatakan_test`):**

| Kennzahl | Wert |
|----------|------|
| Passed | **869** |
| Skipped | **1** (P-10 — bekannter Nicht-Blocker) |
| Failed | **0** |

**Security (Paket 07-02):**

- invoice2data läuft ausschließlich lokal auf bereits extrahiertem Text; kein Netzwerk.
- PaddleOCR nicht installiert; Lazy Import verhindert jede Modellinitialisierung ohne Paket.
- Kein Azure-Aufruf, keine echten Rechnungen, kein Gmail, keine UI, kein Push.

---

### Added

- **packages/core/invoice_extraction/hybrid_kandidat.py** — `HybridKandidat` (frozen dataclass), `AuswahlStatus` (AUSGEWAEHLT/ABGELEHNT/KONFLIKT/UNSICHER), `from_kandidat()` (Legacy-Adapter).
- **packages/core/invoice_extraction/hybrid_ergebnis.py** — `HybridStatus` (OK/MUSS_GEPRUEFT_WERDEN/NICHT_GEFUNDEN), `HybridExtractionResult`, `berechne_hybrid_status()`.
- **packages/core/invoice_extraction/provider_ports.py** — `InvoiceTextProvider`, `InvoiceCandidateProvider`, `CloudInvoiceProvider` (Protocol), `InvoiceProviderResult`.
- **packages/core/invoice_extraction/pdfplumber_provider.py** — `PdfplumberProvider`: bestehenden Parser als ersten und einzigen Provider eingebunden (kein invoice2data, kein PaddleOCR, kein Azure-Adapter/SDK in diesem Paket).
- **apps/server/config.py** — 5 Feature-Flags (`BUCHUNG_ATAKAN_FEATURE_HYBRID_PIPELINE`, `INVOICE2DATA`, `PADDLE_OCR`, `AZURE_DI`, `QUALITAETSAUDIT`), alle default `False`; noch kein Code liest/verzweigt auf diesen Flags (folgt mit Pipeline-Anbindung).
- **apps/server/models/hybrid_extraktion.py** — `HybridErkennungslauf`, `HybridKandidatOrm`, `HybridKonflikt`, `HybridStatusentscheidung`.
- **migrations/versions/011_hybrid_invoice_extraction.py** — 4 neue Tabellen; ENABLE RLS, FORCE RLS, `tenant_isolation`-Policy auf allen; `down_revision=010`.
- **tests/unit/test_hybrid_kandidat.py**, **test_hybrid_ergebnis.py**, **test_provider_ports.py**, **test_feature_flags.py** — 49 neue Unit-Tests.
- **tests/integration/test_hybrid_extraktion.py** — 18 Integrationstests (Persistenz, RLS, FORCE RLS, Tenant-Isolation, alembic check).

### Changed

- `packages/core/invoice_extraction/__init__.py` — neue Typen exportiert; bestehende Exporte unverändert.
- `apps/server/models/__init__.py` — neue ORM-Klassen importiert.
- `tests/conftest.py` — 4 neue Tabellen in `_TRANSAKTIONALE_TABELLEN`.

### Fixed

- **packages/core/ec_sammlung/gruppierung.py** (Phase 6 Regression) — Sortierschlüssel bei EC-Beleggruppierung um `betrag_cent` erweitert. Zwei Belege mit identischem Buchungsdatum wurden bisher per zufälliger UUID sortiert; bei Überdeckung konnte das den exakt passenden Beleg statt des zu großen entfernen (`test_e04_entfernte_belege_bleiben_verfuegbar` flakte ca. 40 % der isolierten Läufe). Jetzt deterministisch: bei Datumsgleichstand bleibt der kleinere Betrag stehen.

### Verified

- Tenant-Isolation (HY05, `test_tenant_isolation_hybrid_erkennungslaeufe`) mit nicht privilegierter Testrolle (`buchungatakan_migrations`; kein Superuser, kein BYPASSRLS) nachgewiesen — mit einer Superuser-Verbindung wird RLS grundsätzlich umgangen (bekanntes Verhalten, siehe P-10) und der Test liefert falsch-positive Ergebnisse.
- Vollständiger Teststand mit `buchungatakan_test` und nicht privilegierter Rolle, zweifach reproduziert: **810 passed, 1 skipped (P-10), 0 failed**.

### Security

- `BUCHUNG_ATAKAN_FEATURE_AZURE_DI` default `False` — kein Azure-Aufruf ohne explizite Aktivierung.
- `InvoiceProviderResult.fehlercode` sanitisiert: nur Fehlertyp, kein Volltext, kein Dateiname.
- Keine echten Rechnungen; keine Azure-Credentials; kein Realtest; kein Push.

---

## [0.6.0] — 2026-07-01 — Phase 6: EC-/Telexpert-Sammelzuordnung — fachlich abgenommen

**Branch:** `phase/06-ec-telexpert-sammelzuordnung`
**GSD-Phase:** 6 | **ENTWICKLUNGSPLAN-Phase:** 5
**Anforderungen:** EC-01..10 (alle erfüllt)

### Added

- **apps/server/models/ec_sammlung.py** — `EcBeleg`, `EcSammellauf`, `EcSammelzuordnung`, `EcSammelzuordnungBeleg`; Status-Enums `EC_SCORE`, `EC_ZUORDNUNG_STATUS`, `EC_BELEG_ROLLE`.
- **migrations/versions/010_ec_sammelzuordnung.py** — ec_belege, ec_sammellaeufe, ec_sammelzuordnungen, ec_sammelzuordnung_belege + RLS + Partial-Unique-Indices.
- **packages/core/ec_sammlung/normalisierung.py** — `normalisiere_buchungsart()`: `CardPayment`/`EC_Karte` in gemeinsame Normkategorie; kein stilles Downgrade unbekannter Begriffe (EC-01).
- **packages/core/ec_sammlung/scorer.py** — `berechne_ec_score()`: STARK bei exakter Summe (EC-02), UNSICHER bei Differenz (EC-04), OFFENE_PERIODE ohne nächste Einzahlung (EC-06).
- **packages/core/ec_sammlung/gruppierung.py** — `gruppiere_einzahlungen()`: greedy-pop bei Überdeckung (neueste Belege zuerst, EC-03); Idempotenz (EC-08).
- **apps/server/repositories/ec_sammelzuordnung.py** — `EcSammelzuordnungRepository`: bestaetigen, ablehnen, MANUELL_*-Unveränderlichkeit (EC-05), rematch-Schutz.
- **apps/server/services/ec_sammelzuordnung_service.py** — rematch-Algorithmus, offene Periode (EC-06), ohne_vorschlag-Logik.
- **apps/server/routers/ec_sammelzuordnungen.py** — `POST /{id}/bestaetigen`, `POST /{id}/ablehnen`; begin()-konformes Session-Pattern.
- **apps/server/auth/dependencies.py** — `CurrentUserId`-Dependency ergänzt.
- **tests/db_guard.py**, **tests/test_db_guard.py** — Pytest-Guard (commit `2f749bd`): 14 Guard-Tests, blockiert TRUNCATE gegen `buchungatakan_realtest` und `buchungatakan`; doppelte Absicherung via `pytest_configure` + Pre-TRUNCATE-Check (EC-10, Sicherheitsinfrastruktur).

### Tests (Stand 2026-07-01)

| Testsuite | Ergebnis |
|-----------|----------|
| Integration + Security (buchungatakan_test) | **175 passed**, 1 skipped (P-10), 0 failed |
| Gesamt (buchungatakan_test, TEST_DATABASE_URL) | **743 passed**, 1 skipped, 0 failed (~75 s) |
| Gesamtsammlung inkl. Guard-Tests | **744 gesammelt** (572 passed / 172 skipped ohne TEST_DATABASE_URL) |

### EC-01..10 Akzeptanzkriterien

| ID | Status |
|----|--------|
| EC-01 | **ERFÜLLT** — CardPayment/EC_Karte normalisiert |
| EC-02 | **ERFÜLLT** — exakte Summe → STARK |
| EC-03 | **ERFÜLLT** — Überdeckung: neueste Belege zuerst entfernt |
| EC-04 | **ERFÜLLT** — Unterdeckung: kein falscher STARK-Treffer |
| EC-05 | **ERFÜLLT** — final verwendeter Beleg gesperrt (Partial-Unique-Index) |
| EC-06 | **ERFÜLLT** — offene Periode löscht keine Kandidaten |
| EC-07 | **ERFÜLLT** — jeder Beleg im Protokoll (ec_sammelzuordnung_belege, AKTIV/ENTFERNT) |
| EC-08 | **ERFÜLLT** — zweiter identischer Lauf: 0 Zusatzzahlungen (UPSERT ON CONFLICT) |
| EC-09 | **ERFÜLLT** — alle Abfragen auf unternehmen_id + Bankkonto begrenzt |
| EC-10 | **ERFÜLLT** — 744 gesammelt, 743 passed/1 skipped (buchungatakan_test, 0 failed), kein Rückschritt |

### EC-Realtest März 2026

- MT940-Reimport: 17 Dateien, 2 Bankkonten, 819 Bankbuchungen, 69 EC_EINZAHLUNG; 0 Parsefehler; Idempotenz-Import: 0 neu, 819 dedupliziert.
- EC-Belegimport: 18 Belege akzeptiert (1 negative Kulanzauszahlung fachlich ausgeschlossen); Idempotenz: 0 neu, 18 dedupliziert.
- Vorschläge: 0 STARK, 3 UNSICHER, 1 OFFENE_PERIODE (begrenzter Belegzeitraum — kein falsches Hochstufen).
- Manuelle Entscheidungen: 1 UNSICHER bestätigt, 1 abgelehnt; nach Rematch Schutz beider Gruppen nachgewiesen.
- Datenschutz: keine echten Inhalte (Dateinamen, IBANs, Kontonummern, Verwendungszwecke, Hashes) in Git oder Dokumentation.

### Security

- `unternehmen_id` ausschließlich aus JWT; fremdes Objekt → 404.
- MANUELL_BESTAETIGT/MANUELL_ABGELEHNT niemals änderbar (Partial-Unique-Index + Applikationslogik).
- Keine doppelte aktive Belegverwendung (Partial-Unique-Index `uix_ec_szb_aktive_belegverwendung`).
- Pytest-Guard (commit `2f749bd`) verhindert dauerhaft Zugriff auf Realtest- und Produktivdatenbanken in der Testsuite.

### Bewusste Verschiebungen

- **Gmail-Nachsuche:** abhängig von Mail-Infrastruktur — GSD-Phase 8.
- **GUI für EC-Workflow:** PySide6-Client — GSD-Phase 9.
- **P-10 (Skip):** App-Role-RLS-Test mit separater buchungatakan_app-Verbindung — kein Blocker.
- **Azure:** vollständig deaktiviert; Aktivierung nur nach ausdrücklicher Nutzerfreigabe.

---

## [0.5.0] — 2026-06-29 — Phase 5: Bankimport und allgemeines Rechnungs-Matching

**Branch:** `phase/05-bankimport-allgemeines-matching`
**GSD-Phase:** 5 | **ENTWICKLUNGSPLAN-Phase:** 4
**Anforderungen:** BANK-01..11 (alle erfüllt)

### Added

- **apps/server/models/bankbuchung.py** — `Bankkonto`, `Kontoauszugsimport`, `Bankbuchung` (betrag_cent als INTEGER, BANK-10), `BuchungsKlassifikation` StrEnum (MATCHBAR/NICHT_ZU_MATCHEN/EC_EINZAHLUNG/MANUELL_PRUEFEN, BANK-11); Dedup-Fingerprint UNIQUE-Constraint (BANK-02).
- **apps/server/models/zuordnung.py** — `ZuordnungsVorschlag`: score, status, positive_signale, negative_signale (JSON), matcher_version, entschieden_at, entschieden_von_benutzer_id, entscheidungsnotiz; Partial Unique Index `uix_zuordnung_eine_bestaetigte_pro_buchung` (BANK-05).
- **packages/core/matching/signale.py** — `Signal`, `MatchErgebnis` Datenklassen.
- **packages/core/matching/scorer.py** — `berechne_match()`: Betragsvergleich, Datumsabstand, Rechnungsnummer im Verwendungszweck, Lieferantenname; positive/negative Signale (BANK-04); Score STARK (≥80) / UNSICHER (40–79).
- **apps/server/repositories/zuordnung.py** — `ZuordnungRepository`: `upsert_vorschlaege()` (ON CONFLICT DO UPDATE WHERE status NOT IN geschützte, BANK-06/07), `get_by_id()` (mandantensicher), `bestaetigen()` (BANK-05/07, idempotent, IntegrityError → 409), `ablehnen()` (BANK-06, idempotent), `get_bestaetigte_buchung_ids()`, `get_abgelehnte_paare()`, `list_vorschlaege()`.
- **apps/server/services/matching_service.py** — `MatchingService.rematch()`: lädt bestätigte Buchungs-IDs (BANK-07) + abgelehnte Paare (BANK-06) vor Scoring; nur MATCHBAR-Buchungen und aktive Rechnungen des Mandanten (BANK-09/11).
- **apps/server/routers/bankbuchungen.py** — `POST /api/v1/bankkonten`, `POST /api/v1/bankbuchungen/importe`, `GET /api/v1/bankbuchungen`.
- **apps/server/routers/zuordnungen.py** — `POST /api/v1/zuordnungen/rematch` (BANK-08), `GET /api/v1/zuordnungen` (Filter), `POST /api/v1/zuordnungen/{id}/bestaetigen` (BANK-05/07), `POST /api/v1/zuordnungen/{id}/ablehnen` (BANK-06); `EntscheidungRequest` mit notiz (max 500), benutzer_id nur aus JWT.
- **migrations/versions/006_bankbuchungen.py** — bankkonten, kontoauszugsimporte, bankbuchungen + GRANT + FORCE RLS.
- **migrations/versions/007_zuordnungen.py** — zuordnungsvorschlaege + UNIQUE(unternehmen_id,bankbuchung_id,rechnung_id) + GRANT + FORCE RLS.
- **migrations/versions/008_zuordnung_entscheidungen.py** — entschieden_at, entschieden_von_benutzer_id, entscheidungsnotiz + FK benutzer.id + Partial Unique Index (BANK-05).
- **tests/unit/test_bankbuchungen.py**, **test_bankbuchungen_klassifikation.py**, **test_matching_scorer.py**, **test_zuordnung_entscheidungen.py** — Unit-Tests BANK-01..11 (359 Unit+Security gesamt).
- **tests/integration/test_bankbuchungen.py**, **test_zuordnungen.py**, **test_zuordnung_entscheidungen_api.py** — Integrationstests BANK-01..11 (73 Integration gesamt).

### Tests (Stand 2026-06-29)

| Testsuite | Ergebnis |
|-----------|----------|
| Unit + Security | **359 passed**, 0 failed |
| Integration | **73 passed**, 1 skipped (P-10), 0 failed |
| Gesamt | **432 passed**, 1 skipped, 0 failed |

### Bewusste Verschiebungen

- **EC-/Telexpert-Sammelzuordnung (EC-01..10):** Eigene Matching-Logik mit Zeitfenstern und Belegsperre — Phase 6.
- **Gmail-Nachsuche:** Abhängig von Mail-Infrastruktur — spätere Phase.
- **GUI für Zuordnungsworkflow:** PySide6-Client — Phase 9.
- **RECH-10 AuditLog:** Wartet auf PATCH-Korrekturendpunkt.
- **P-10 (Skip):** App-Role-RLS-Test mit separater `buchungatakan_app`-Verbindung — kein Blocker.

---

## [0.4.0] — 2026-06-28 — Phase 4: Rechnungsimport, Extraktion, Validierung

**Branch:** `phase/04-rechnungsimport-extraktion-validierung`
**GSD-Phase:** 4 | **ENTWICKLUNGSPLAN-Phase:** 3
**Anforderungen:** RECH-01..09, RECH-11 (erfüllt); RECH-07 (teilweise, s. u.); RECH-10 (bewusst verschoben)

### Added

- **packages/core/invoice_extraction/** — Neues Paket:
  - `kandidat.py` — `Kandidat` (frozen dataclass, RECH-01): feld, rohwert, normwert, methode, quelle, confidence; `KandidatQuelle` StrEnum.
  - `extraction_result.py` — `ExtractionResult`, `ExtraktionsStatus` (OK/PRUEFFALL/UNVOLLSTAENDIG/MATH_FEHLER).
  - `text_cleaner.py` — `PDFTextCleaner` (portiert aus Altprojekt).
  - `pdf_reader.py` — `PDFReader` (portiert aus Altprojekt, pdfplumber).
  - `invoice_data.py` — `InvoiceData` (portiert aus Altprojekt, Zwischenrepräsentation).
  - `invoice_parser.py` — `InvoiceParser` (portiert aus Altprojekt): regex-basierte Rechnungsextraktion.
  - `invoice_parser_adapter.py` — `invoice_data_to_kandidaten`, `extract_from_text`, `extract_from_pdf` (RECH-01). `validate()` verbindlich eingebunden — kein hardcoded OK.
  - `invoice_validator.py` — `validate(result)` (RECH-04/05): Math-Check ±0.02 EUR; USt-ID≠Rechnungsnummer-Sperre; Pflichtfelder-Check.
  - `ports.py` — `InvoiceExtractionPort` Protocol (ENTWICKLUNGSPLAN §5).
- **apps/server/models/rechnung.py** — `Rechnung` (version_lfd_nr RECH-08, ist_aktiv RECH-09), `Erkennungslauf` (RECH-03), `RechnungsKandidat` (RECH-01), `RechnungsStatus`. Numeric-Felder als `Decimal`.
- **apps/server/repositories/rechnung.py** — `RechnungRepository`: `_neue_version_ist_aktiv` (RECH-08 Versionierungslogik), `create_version`, `list_aktive` (RECH-09), `get_aktive_by_nummer`, `list_erkennungslaeufe`, `list_kandidaten_fuer_rechnung`, mandantensichere `_get_erkennungslauf_by_id` (404 bei fremdem Lauf).
- **apps/server/routers/rechnungen.py** — 5 Endpunkte (RECH-03/06/08/09/11): POST (Anlegen mit RECH-06-Guard), GET Liste, GET Detail, GET Erkennungsläufe, GET Kandidaten. `RechnungCreateRequest` mit `extra="forbid"`, kein `unternehmen_id`-Feld (SRV-05).
- **Auth/RLS-Härtung** (`apps/server/auth/dependencies.py`): SECURITY DEFINER-Funktionen für benutzer-Lookups (`auth_benutzer_by_name`, `auth_benutzer_by_id`). `get_rls_session` setzt `app.current_tenant` transaktionslokal. `CurrentTenantId`-Dependency (unternehmen_id nur aus JWT).
- **migrations/versions/005_rechnungen.py** — rechnungen, erkennungslaeufe, rechnungs_kandidaten + GRANT + FORCE RLS + FK unternehmen_id→unternehmen(id). **Applied** (`buchungatakan_test`).
- **tests/fixtures/*.txt** — Synthetische Rechnungs-Fixtures: rechnung_standard, rechnung_amazon, rechnung_kleinunternehmer (§19 UStG), rechnung_doppelzeile.
- **tests/unit/test_invoice_parser_golden.py** — 20 Golden-File-Tests (4 Fixtures × Felder + Robustheit).
- **tests/integration/test_rechnungen.py** — 6 Integrationstests (RECH-03/06/08/09/11 + Cross-Tenant 404): Anlegen, Liste/Detail, Versioning neueres Datum, Versioning gleiches Datum, Erkennungsläufe-Isolation, Kandidaten-Isolation.

### Fixed

- **invoice_parser.py A-02:** `_line_is_inside_recipient_block` — indexbasiert statt `list.index()`.
- **invoice_parser.py A-06:** Englische Monate `april`, `august`, `september`, `november` in MONTHS-Dict ergänzt.
- **invoice_parser.py A-08:** `parse()` mit try/except — gibt `InvoiceData(confidence=0.0)` statt Exception.
- **invoice_parser_adapter.py:** `extract_from_text()` / `extract_from_pdf()` rufen jetzt `validate()` auf.
- **apps/server/repositories/rechnung.py** (Produktivfix `5eff878`): `await session.refresh(neue)` nach `session.commit()` in `session.begin()`-Kontext entfernt. SQLAlchemy 2.x lässt keine Session-Operationen nach explizitem `commit()` im `begin()`-Kontext zu. `flush()` lädt server-generierte Werte via RETURNING; `expire_on_commit=False` hält sie verfügbar. Fehler durch PostgreSQL-Integration reproduziert.
- **tests/integration/conftest.py** (Infrastruktur-Fixes): NullPool-Patch für App-Engine (Windows ProactorEventLoop + asyncpg-Pool-Kompatibilität); idempotente Seed-Logik FORCE-RLS-konform via `set_config`-Transaktion.

### Tests (Stand 2026-06-28)

| Testsuite | Ergebnis |
|-----------|----------|
| Unit + Security | **253 passed**, 0 failed |
| Integration | **25 passed**, 1 skipped (P-10), 0 failed |

Alle Blocker B-01/B-02/B-03 gelöst. Integrationstests vollständig lauffähig.

### Bewusste Verschiebungen

- **RECH-10 — AuditLog manueller Korrekturen:** Verschoben bis ein `PATCH`-Korrekturendpunkt
  implementiert wird. Keine AuditLog-Tabelle, keine Migration 006, keinen PATCH-Endpunkt
  in Phase 4. Details: `.planning/phases/04-.../04-VERIFY.md` §6.
- **RECH-07 (teilweise):** PRUEFFALL-Mechanismus implementiert (RECH-06); widersprüchliche
  Mehrfachsignale nicht testbar da nur ein starkes Signalfeld (`rechnungsnummer`) vorhanden.
- **P-10 (Skip):** App-Role-RLS-Test erfordert separaten Engine-Fixture mit
  `buchungatakan_app`-Credentials — kein Phase-4-Blocker.

---

## [0.3.0] – 2026-06-28 – Lokaler Client, DATA_ROOT, Dokumentinventar-Sync (Kern)

**Branch:** `phase/02-zentrale-api-postgresql-mandantensicherheit`
**GSD-Pläne:** 03-01 (Server-Schema), 03-02 (Core-Utils), 03-03 (Client-Infrastruktur)
**Anforderungen:** CLI-01..10

### Added

- **packages/core/path_config.py** — `PathConfig` adaptiert aus Altprojekt (BUCHUNG_ATAKAN_*,
  ohne network_document_root/D6, ohne SQLite-Pfade); `DataRootSecurityError`, `guard_data_root`,
  `load_path_config_with_source`; Windows/POSIX-Kompatibilität via `PureWindowsPath`.
- **packages/core/hash_utils.py** — `sha256_of_file` (chunked 64 KB), `sha256_of_bytes` (stdlib).
- **packages/core/mime_utils.py** — Magic-Byte-Erkennung (PDF/PNG/JPEG/ZIP) + Extension-Fallback.
- **packages/core/document_locator.py** — `DokumentLocator` (frozen dataclass, CLI-02-Validierung:
  kein absoluter Pfad), `DokumentVerfuegbarkeit`, `DokumentQuelle` StrEnums.
- **apps/client/config.py** — `ClientConfig` (JSON-Persistenz, ENV-Override, kein Token gespeichert).
- **apps/client/document_scanner.py** — `scan_for_documents` (rglob, sorted), `to_relative_posix`
  (POSIX-Pfade, CLI-02).
- **apps/client/outbox.py** — `DokumentOutbox` (SQLite lokal, CLI-06/07): enqueue/pending/
  mark_sent/mark_failed/reset_failed.
- **apps/client/api_client.py** — `BuchungAtakanClient` (httpx async, kein File-Upload CLI-08).
- **apps/client/sync_service.py** — `DokumentSyncService`: sync_alle_dokumente (scan→hash→POST
  oder Outbox), outbox_verarbeiten (idempotent CLI-07).
- **apps/server/models/dokument.py** — `Dokument` ORM (TenantMixin, TimestampMixin,
  UNIQUE(unternehmen_id,sha256) CLI-03, FK zu installationen).
- **apps/server/routers/installationen.py** — POST /api/v1/installationen (idempotent, JWT-geschützt).
- **apps/server/routers/dokumente.py** — POST /api/v1/dokumente (Metadaten-only CLI-08, idempotent
  CLI-07), PATCH /{id}/verfuegbarkeit (CLI-10), GET (RLS = CLI-09).
- **migrations/versions/004_documents_table.py** — `dokumente`-Tabelle + GRANT + RLS
  (down_revision=003; noch nicht applied — B-01).
- **httpx** in main-Dependencies verschoben (Client benötigt es zur Laufzeit).

### Tests

- 102 passed (95 Unit+Security aus Phase 02, +7 neue Sync-Tests), 0 failed.
- 20 Integration-Tests: weiter skipped (TEST_DATABASE_URL fehlt).
- Neue Unit-Tests: test_hash_utils, test_mime_utils, test_document_locator,
  test_path_config_core, test_document_scanner, test_client_config,
  test_outbox, test_server_models_phase3, test_sync_service.

### Blocker

- **B-01:** Migration 004 applyen → wartet auf P-11 (Migration 003)
- **B-02:** Integrationstests → wartet auf TEST_DATABASE_URL

---

## [0.2.0] – 2026-06-27 – Zentrale API, PostgreSQL, Mandantensicherheit

**Branch:** `phase/02-zentrale-api-postgresql-mandantensicherheit`
**GSD-Pläne:** 02-01 (Datenbankschicht), 02-02 (Auth & Repositories), 02-03 (RLS, Tests, Docs)

### Added

- **FastAPI-Grundstruktur** (`apps/server/main.py`, `apps/server/config.py`): `create_app()`-Factory,
  pydantic-settings-Konfiguration (DATABASE_URL, SECRET_KEY, ALGORITHM, ACCESS_TOKEN_EXPIRE_MINUTES).
- **Datenbankschicht** (`apps/server/database.py`): async SQLAlchemy 2.x Engine (asyncpg),
  `AsyncSession`-Factory, `DBSession`-Dependency.
- **ORM-Modelle** (`apps/server/models/`): `DeclarativeBase`, `TenantMixin` (unternehmen_id + Index),
  `Unternehmen`, `Benutzer` (mit `Rolle`-Enum), `Installation`.
- **Alembic-Migrationen** (`migrations/versions/`):
  - `001_initial_schema.py` — Tabellen unternehmen, benutzer, installationen; gen_random_uuid() Default.
  - `002_seed_test_data.py` — ENV-Guard `BUCHUNG_ATAKAN_SEED_TEST_DATA=1`; 2 Mandanten, 3 Benutzer.
  - `003_rls_policies.py` — buchungatakan_app-Rolle, GRANT, ENABLE/FORCE ROW LEVEL SECURITY,
    Tenant-Isolation-Policies für benutzer + installationen.
- **Auth-Schicht** (`apps/server/auth/`):
  - `service.py` — Argon2-Passwort-Hash (pwdlib), Timing-sicherer Dummy-Verify-Pfad, PyJWT-Token.
  - `schemas.py` — `TokenResponse`, `TokenData` (extra="forbid").
  - `dependencies.py` — `get_current_unternehmen_id`, `get_current_user`, `get_rls_session`
    (transaktionslokal via `set_config('app.current_tenant', :tid, true)`), `RoleChecker`.
  - `router.py` — `POST /api/v1/auth/login` (OAuth2PasswordRequestForm, form-data).
- **Repositories** (`apps/server/repositories/`): `BaseRepository[T]`, `BenutzerRepository`
  (global-unique benutzername), `UnternehmenRepository`.
- **Endpunkte** (`apps/server/routers/`): `GET /health`, `GET /api/v1/unternehmen/me`.
- **Exception-Handler** (`apps/server/exceptions.py`): 4xx → `{"detail": ...}`,
  5xx → `{"detail": "Interner Serverfehler", "reference_id": uuid4}`, 422 ohne interne Details.
- **Testsuite**: 19 Unit-Tests (auth, schemas, role-checker), 5 Security-Tests (Error-Sanitisierung),
  20 Integrationstests (migrations, auth-flow, tenant-isolation, RLS) — alle skip ohne TEST_DATABASE_URL.

### Security

- RLS (SRV-03): `ENABLE ROW LEVEL SECURITY` + `FORCE ROW LEVEL SECURITY` auf benutzer, installationen.
  Policy prüft `unternehmen_id::text = current_setting('app.current_tenant', true)`.
- Timing-sicherer Login: Dummy-Hash verhindert User-Enumeration via Timing-Angriff.
- JWT enthält keine DB-Zugangsdaten (SRV-10); `sub` = Benutzer-UUID, nicht Benutzername.
- Error-Sanitisierung (SRV-08): 5xx ohne SQL/Stacktrace im Response-Body; interne `reference_id` im Log.

### Open (nach Phase 2)

- Migration 003 (RLS) muss mit buchungatakan_migrations-Rolle manuell applied werden.
- `buchungatakan_app`-Passwort in Migration 003 ist `PLACEHOLDER` — vor Produktion setzen.
- Vollständige RLS-Verifikation (als buchungatakan_app) erfordert separaten Engine-Fixture.
- Phase 3 und folgende: nicht begonnen.

---

## [Phase 0] – 2026-06-26 – Bestandsaufnahme, Architektur-Freeze, Migrationsplan

**GSD-Pläne:** 01-01 (Wave 1), 01-02 (Wave 2), 01-03 (Wave 3)
**Kein Anwendungscode. Rein dokumentarisch.**

### Added

- Phase-0-Lieferobjekte (alle unter `docs/phase0/`):
  - `BESTANDSAUFNAHME.md` — Inventar aller 14 Altmodule, 11 externen Abhängigkeiten,
    14 sqlite3.connect()-Stellen, Pfadannahmen inkl. D6-SUPERSEDED-Stellen
  - `MODULKLASSIFIKATION.md` — Client/Server/Shared-Core-Zuordnung, Wiederverwendungsstrategie,
    technische Schulden (6 HOCH, 6 MITTEL, 5 NIEDRIG)
  - `MIGRATIONSPLAN.md` — Tabellen-Mapping (9 ZENTRAL + 3 ergänzend + 1 LOKAL), reversibler
    6-Schritte-Migrationspfad Ist-SQLite → Ziel-PostgreSQL-Multi-Tenant
  - `API_GROBVERTRAG.md` — Identifier-Definitionen (unternehmen_id, installation_id,
    Dokument-Lokator), API /api/v1/, Feature-Flags BUCHUNG_ATAKAN_FEATURE_*, PySide6-Bindung,
    Mandantentrennungsregeln
  - `BASELINE_TESTS.md` — Altprojekt-Testlauf 2026-06-26: 162 passed / 0 failed, read-only
  - `AKZEPTANZKRITERIEN.md` — ARCH-01..11 bewertet: 10 ERFÜLLT, 1 NICHT GETESTET (ARCH-11)

- Projektdokumentation aus den autoritativen Planungsdokumenten und dem Altprojekt
  übernommen und neu strukturiert (2026-06-25):
  - `AGENTS.md`
  - `docs/PROJECT_CONTEXT.md`
  - `docs/BUSINESS_RULES.md`
  - `docs/ARCHITECTURE.md`
  - `docs/CURRENT_STATUS.md`
  - `docs/OPEN_POINTS.md`
  - `docs/IMPLEMENTATION_HISTORY.md`
  - `docs/TEST_HISTORY.md`

### Decided (Phase 0)

- D1 Datenmigration: keine sofortige Blindmigration; Phase 0 definiert einen konkreten,
  testbaren und reversiblen Migrationspfad. Keine Löschung von Produktiv-/Altdaten.
- D2 Code-Wiederverwendung: alte Fachmodule werden **phasenweise** über Ports/Adapter nach
  `packages/core` gehoben, kein Massen-Copy, keine grundlose Neuschreibung.
- D3 GUI: lokaler Client mit **PySide6**; Domänenlogik UI-unabhängig.
- D4 Naming: `BuchungAtakan` / `BUCHUNG_ATAKAN_*`; Altnamen nur als dokumentierte
  Migrations-Fallbacks.
- D5 Doku-Sprache: Deutsch.
- D6 Zentrale UNC-/Tailscale-Dokumentablage (2026-06-21) **superseded** → Originale bleiben
  lokal, zentral nur Metadaten/Hash/relative Locator; alte Intention nur als „Optional C”
  geparkt.

### Open (nach Phase 0, in Folgephasen)

- Phase 1 (Server & Mandantentrennung) und jeglicher Anwendungscode: nicht begonnen.
- ARCH-11 (Nutzerfreigabe): ausstehend — blocking Checkpoint.

---

## Altprojekt (Referenz)

Zusammenfassung des Vorgängersystems `BuchungAtakan` (Einzelmandant, lokale **SQLite**,
Python-Desktop). Quelle: `C:\Users\User\PycharmProjects\BuchungAtakan` und dessen
`CHANGELOG.md` / Kontextpaket `BuchungAtakan_Codex_Project_Context_v2`.
Die dortige Phasennummerierung ist **eigenständig** und nicht mit den neuen
ENTWICKLUNGSPLAN-Phasen 0–8 identisch.

Wesentliche, im Altprojekt **umgesetzte** Funktionsblöcke:
- Rechnungs-PDF-Erkennung, -Extraktion, -Import und -Ablage; Duplikat-/Versionsregel.
- Kontoauszugsimport und Speicherung von Bankbuchungen.
- Matching Rechnung ↔ Bankbuchung (sichere/unsichere/nicht gefundene Fälle); manuelle Prüfung.
- `NICHT_ZU_MATCHEN` für Bankbuchungen.
- EC-/Telexpert-Sammelzuordnung (dynamische Zeitfenster, Centwerte, Belegsperre, Protokoll,
  eigenes Excel-Blatt).
- Gmail-Mehrpostfachbetrieb, Raw-PDF-Ablage mit Dedup, Gmail-Nachsuche für offene
  Bankbuchungen, Scan-Intervalle.
- Frei wählbare Datenwurzel (`DATA_ROOT`) mit relativen Pfaden; DATA_ROOT-Sicherheits-Guards.
- Excel-Bericht mit Übersicht/Status-Blättern.
- Reproduzierbare synthetische Testbasis; zuletzt **162** Pytests grün.

Im Altprojekt **offen/spätere** Blöcke: GUI, Zeitsteuerung/Scheduler, umfassende
Parseroptimierung, optionale KI-Hilfe.

Details: `docs/IMPLEMENTATION_HISTORY.md`, `docs/TEST_HISTORY.md`, `docs/CURRENT_STATUS.md`.
