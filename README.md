# BuchungAtakan

Technischer Neustart des BuchungAtakan-Systems.

## Zielarchitektur

- lokaler Windows-Client je Unternehmen
- zentraler FastAPI-Server
- zentrale PostgreSQL-Datenbank
- zentrale Dokumentablage auf dem Server
- Mandantentrennung über `unternehmen_id`
- Steuerberaterzugriff über authentifizierte Weblinks
- kein direkter Datenbank- oder Dateifreigabezugriff

## Entwicklungsstatus

Die neue Projektgrundstruktur wurde erstellt.