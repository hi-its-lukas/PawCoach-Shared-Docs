# Accounting Settings API — iOS-Spec

**Datum:** 2026-03-21
**Status:** Backend implementiert, iOS ausstehend

---

## Übersicht

Neue API-Endpoints für die Buchhaltungs-Integration (SevDesk, lexoffice, BuchhaltungsButler). Alle Endpoints erfordern JWT-Auth + Admin-Rolle.

---

## Neue Endpoints

### GET `/api/admin/accounting/settings`

Buchhaltungs-Konfiguration lesen. Credentials werden **nie** zurückgegeben, nur `has_*`-Flags.

**Rate-Limit:** 60/hour
**Auth:** JWT + Admin

**Response:**

```json
{
  "school": {
    "tool": "sevdesk",
    "mode": "push",
    "has_sevdesk_token": true,
    "has_lexoffice_key": false,
    "has_bb_credentials": false
  },
  "platform": {
    "tool": "lexoffice",
    "has_sevdesk_token": false,
    "has_lexoffice_key": true,
    "has_bb_credentials": false,
    "has_webhook_secret": true
  },
  "webhook_urls": {
    "sevdesk": "https://app.pawcoach.de/webhooks/accounting/sevdesk",
    "lexoffice": "https://app.pawcoach.de/webhooks/accounting/lexoffice",
    "buchhaltungsbutler": "https://app.pawcoach.de/webhooks/accounting/buchhaltungsbutler"
  }
}
```

**`tool`-Werte:** `none`, `sevdesk`, `lexoffice`, `buchhaltungsbutler`, `both`
**`mode`-Werte:** `none`, `push`, `full`

---

### PUT `/api/admin/accounting/settings`

Buchhaltungs-Konfiguration aktualisieren. Nur gesetzte Felder werden überschrieben. Leere Strings werden ignoriert (bestehende Credentials bleiben erhalten).

**Rate-Limit:** 10/hour
**Auth:** JWT + Admin

**Request Body (Beispiel: SevDesk auf Schulebene aktivieren):**

```json
{
  "school": {
    "tool": "sevdesk",
    "mode": "push",
    "sevdesk_api_token": "abc123..."
  }
}
```

**Request Body (Beispiel: lexoffice auf Plattformebene + Webhook):**

```json
{
  "platform": {
    "tool": "lexoffice",
    "lexoffice_api_key": "xyz789...",
    "webhook_secret": "mein-secret-123"
  }
}
```

**Credential-Keys im JSON:**

| JSON-Key | DB-Feld | Ebene |
|----------|---------|-------|
| `school.sevdesk_api_token` | `school_sevdesk_api_token` | Schule |
| `school.lexoffice_api_key` | `school_lexoffice_api_key` | Schule |
| `school.bb_api_client` | `school_bb_api_client` | Schule |
| `school.bb_api_secret` | `school_bb_api_secret` | Schule |
| `school.bb_api_key` | `school_bb_api_key` | Schule |
| `platform.sevdesk_api_token` | `platform_sevdesk_api_token` | Plattform |
| `platform.lexoffice_api_key` | `platform_lexoffice_api_key` | Plattform |
| `platform.bb_api_client` | `platform_bb_api_client` | Plattform |
| `platform.bb_api_secret` | `platform_bb_api_secret` | Plattform |
| `platform.bb_api_key` | `platform_bb_api_key` | Plattform |
| `platform.webhook_secret` | `accounting_webhook_secret` | Plattform |

**Response (Erfolg):**

```json
{
  "status": "ok"
}
```

---

## Bestehende Endpoints (Referenz)

Diese existieren bereits und funktionieren mit JWT-Auth:

| Methode | Endpoint | Beschreibung |
|---------|----------|-------------|
| GET | `/api/admin/accounting/jobs` | Sync-Jobs auflisten (paginiert) |
| POST | `/api/admin/accounting/jobs/<id>/retry` | Einzelnen Job wiederholen |
| POST | `/api/admin/accounting/retry-all` | Alle fehlgeschlagenen wiederholen |
| POST | `/api/admin/accounting/sync-now` | Sofort synchronisieren |
| POST | `/api/admin/accounting/test-connection` | Verbindung testen |
| GET | `/api/customer/accounting/status` | Status (tool, mode, last_sync) |
| GET | `/api/customer/accounting/sync-jobs` | Jobs mit Pagination |

---

## iOS-Implementierung

### Settings-Screen

Die App sollte einen "Buchhaltung"-Bereich in den Schuleinstellungen zeigen:

1. **Tool-Auswahl** (Dropdown): SevDesk / lexoffice / BuchhaltungsButler / Deaktiviert
2. **Modus** (nur Schulebene): Push / Vollständig
3. **Credentials**: Eingabefelder für API-Token/Keys (Password-Felder)
4. **Verbindung testen**: Button → `POST /test-connection`
5. **Webhook-URLs**: Read-only, kopierbar (aus GET-Response)

### Sync-Dashboard

1. **Job-Liste** aus `GET /api/admin/accounting/jobs`
2. **Retry-Buttons** für fehlgeschlagene Jobs
3. **Sync-Now-Button**

### Wichtig

- Credentials NIE cachen oder loggen
- `has_*`-Flags nutzen um UI-Status anzuzeigen (✓ Gesetzt / Nicht konfiguriert)
- Plattform-Settings nur für Admins mit `platform_admin`-Berechtigung anzeigen
