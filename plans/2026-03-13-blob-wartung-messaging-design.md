# Design: Azure Blob Storage, Wartungsmodus, Platform-Admin Messaging

**Datum:** 2026-03-13
**Status:** Erledigt

---

## Feature 1: Azure Blob Storage Abstraction

### Ueberblick

Storage-Abstraction-Layer mit zwei Backends (Local / Azure), steuerbar per `STORAGE_BACKEND=local|azure` Env-Variable. Lokal fuer Development, Azure fuer Production.

### Architektur

```
storage.py (neues Modul)
├── StorageBackend (Abstract Base Class)
│   ├── save(category, filename, data) → url
│   ├── url(path) → public_url
│   ├── delete(path)
│   └── exists(path) → bool
├── LocalStorage    → liest/schreibt in static/uploads/
└── AzureBlobStorage → liest/schreibt in Azure Container

get_storage() → gibt das konfigurierte Backend zurueck
```

- Alle bestehenden Upload-Endpoints rufen `storage.save()` auf statt direkt `open()` / `os.path`
- Azure-Config aus PlatformSettings (existiert bereits: `azure_blob_connection_string`, `azure_blob_container_name`)
- Fallback: Wenn Azure nicht konfiguriert → LocalStorage

### Betroffene Upload-Stellen

| Datei | Funktion | Upload-Typ |
|-------|----------|-----------|
| blueprints/api_auth.py | api_upload_avatar | Avatar |
| blueprints/api_customer/dogs.py | upload_dog_photo | Hundefoto |
| blueprints/api_admin/dogs.py | upload_dog_photo | Hundefoto (Admin) |
| blueprints/api_mobile.py | api_dog_file_upload | Hunde-Dateien |
| blueprints/api_messaging.py | upload_file | Chat-Anhang |
| blueprints/messaging.py | upload_file | Chat-Anhang (Web) |
| blueprints/admin/dogs.py | upload_dog_file | Hunde-Dateien (Web) |
| blueprints/saas.py | school_settings (Logo/Hero) | Schulbilder |
| blueprints/platform_admin.py | settings (Logo) | Plattform-Logo |

### Migrations-Script

`scripts/migrate-to-blob.py`:
- Liest alle Dateien aus `static/uploads/`, `uploads/`
- Uploadet nach Azure Blob mit gleichem Pfad
- Verifiziert Upload (Byte-Vergleich)
- Optionen: `--dry-run`, `--delete-local` (nach Verifizierung)
- Aktualisiert `DogFile.file_path` und `MessageAttachment` URLs in DB
- Fortschrittsanzeige mit Zaehler

### Neue Dependency

- `azure-storage-blob>=12.0.0`

---

## Feature 2: Wartungsmodus

### Datenmodell (PlatformSettings — neue Felder)

| Feld | Typ | Default | Beschreibung |
|------|-----|---------|-------------|
| maintenance_portal_active | Boolean | False | Portal-Wartung manuell aktiv |
| maintenance_portal_message | Text | null | Meldung fuer Portal |
| maintenance_website_active | Boolean | False | Website-Wartung manuell aktiv |
| maintenance_website_message | Text | null | Meldung fuer Website |
| maintenance_scheduled_start | DateTime | null | Geplanter Start |
| maintenance_scheduled_end | DateTime | null | Geplantes Ende |
| maintenance_scheduled_message | Text | null | Meldung fuer geplante Wartung |

### Logik (Middleware @app.before_request)

1. Prueft ob aktuelle Zeit zwischen `scheduled_start` und `scheduled_end` → automatisch aktiv
2. Prueft `maintenance_portal_active` / `maintenance_website_active` → manueller Override
3. Platform-Admins koennen IMMER zugreifen (Bypass)
4. `/api/auth/login` bleibt fuer Platform-Admins erreichbar
5. `/healthz`, `/static/` bleiben immer erreichbar

### Zwei Stufen

- **Portal-Wartung:** Login gesperrt, oeffentliche Seiten (Landing, Schulseiten) normal erreichbar
- **Website-Wartung:** Alles gesperrt — Wartungsseite fuer alle Besucher

### Banner-Vorwarnung

Wenn `scheduled_start` in der Zukunft liegt → gelbes Banner auf allen Seiten:
"Geplante Wartungsarbeiten am [Datum] von [Start] bis [Ende]: [Meldung]"

### UI (Platform-Admin)

Neue Seite/Abschnitt im Platform-Admin mit:
- Toggle Portal-Wartung + Textfeld
- Toggle Website-Wartung + Textfeld
- Wartungsfenster planen (Start, Ende, Meldung)
- Aktueller Status-Anzeige

### CLI

```bash
flask maintenance-on --scope portal|website|all --message "Wartungsarbeiten" --until "2026-03-14 06:00"
flask maintenance-off --scope portal|website|all
flask maintenance-status
```

### API-Endpoints (Platform-Admin)

| Methode | Pfad | Beschreibung |
|---------|------|--------------|
| GET | /api/platform/maintenance | Aktuellen Wartungsstatus abrufen |
| PUT | /api/platform/maintenance | Wartungsmodus setzen/aendern |
| POST | /api/platform/maintenance/schedule | Wartungsfenster planen |
| DELETE | /api/platform/maintenance/schedule | Geplantes Fenster loeschen |

### Template

`templates/maintenance.html` — eigenstaendige Seite mit:
- Wartungsmeldung (individueller Text)
- Geplanter Endzeitpunkt (wenn bekannt)
- PawCoach-Branding
- Kein Login-Link

---

## Feature 3: Platform-Admin ↔ Schuladmin Messaging

### Problem

`/api/messages/contacts` liefert nur Kontakte der eigenen Schule. Platform-Admins sehen keine Schuladmins, und Schuladmins sehen keinen Platform-Admin.

### Loesung

**Fuer Platform-Admins:**
- `/api/messages/contacts` liefert zusaetzlich alle Schuladmins aller Schulen
- Gruppiert nach Schule mit `school_name` pro Kontakt
- Eigene Gruppe "Platform-Admins" fuer Kollegen

**Fuer Schuladmins:**
- `/api/messages/contacts` liefert zusaetzlich alle Platform-Admins
- Angezeigt als Gruppe "PawCoach Support"

**Berechtigungen bei neuer Konversation:**
- Platform-Admin → beliebiger Schuladmin: erlaubt
- Schuladmin → Platform-Admin: erlaubt
- Schuladmin → Schuladmin anderer Schule: weiterhin gesperrt

### Betroffene Dateien

| Datei | Aenderung |
|-------|----------|
| blueprints/api_messaging.py | `list_contacts()` erweitern |
| blueprints/messaging.py | `list_contacts()` erweitern (Web) |
| blueprints/api_messaging.py | `new_conversation()` Berechtigungspruefung |
| blueprints/messaging.py | `new_conversation()` Berechtigungspruefung (Web) |
