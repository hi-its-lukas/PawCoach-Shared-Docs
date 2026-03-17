# Design: iOS Kundenregistrierung mit Schulsuche

**Datum:** 2026-03-13
**Status:** Erledigt

---

## Ueberblick

Drei neue oeffentliche API-Endpoints fuer die iOS-App: Schulsuche, Code-Validierung, Kundenregistrierung. Dazu ein Freigabe-System pro Schule und Join-Code-Verwaltung.

---

## Datenmodell

### SchoolProfile — 2 neue Felder

| Feld | Typ | Default | Beschreibung |
|------|-----|---------|-------------|
| join_code | String(9), unique | Auto-generiert | Format: XXXX-0000 (4 Grossbuchstaben + 4 Ziffern) |
| customer_approval_required | Boolean | False | Wenn True: neue Kunden muessen vom Admin freigeschaltet werden |

### UserSchoolRole — 1 neues Feld

| Feld | Typ | Default | Beschreibung |
|------|-----|---------|-------------|
| approved | Boolean | True | False = wartet auf Freigabe |

---

## Neue Endpoints

### 1. GET /api/public/schools/search?q=...

- Oeffentlich, kein JWT
- Rate-Limit: 30/minute
- Sucht per `SchoolProfile.name.ilike(f"%{q}%")`, min. 2 Zeichen
- Nur Schulen mit aktivem Abo
- Response:
```json
{
  "schools": [
    {"id": 1, "name": "Hundeschule Beispiel", "city": "Berlin", "join_code": "FIDO-2026"}
  ]
}
```

### 2. GET /api/public/schools/by-code/<code>

- Oeffentlich, kein JWT
- Rate-Limit: 30/minute
- Sucht per join_code (case-insensitive)
- Response:
```json
{"id": 1, "name": "Hundeschule Beispiel", "city": "Berlin", "requires_approval": true}
```
- 404 wenn Code ungueltig oder Schule inaktiv

### 3. POST /api/auth/register-customer

- Oeffentlich, kein JWT
- Rate-Limit: 5/minute
- Request:
```json
{
  "email": "...",
  "password": "...",
  "name": "...",
  "join_code": "FIDO-2026",
  "gdpr_consent": true,
  "agb_consent": true
}
```
- Erstellt User + UserSchoolRole(role="customer")
- Wenn customer_approval_required: approved=False, Response: `{"status": "pending_approval"}`
- Wenn nicht: approved=True, Response: `{"status": "registered", "user": {...}}`
- Sendet Bestaetigungs-E-Mail
- Passwort: min. 12 Zeichen
- Fehler: 409 wenn E-Mail existiert, 404 wenn Code ungueltig

---

## Login-Verhalten bei pending_approval

In POST /api/auth/login: Wenn User eine Rolle mit approved=False hat und keine andere freigegebene Rolle:
```json
{"error": "pending_approval", "message": "Dein Konto wartet auf Freigabe durch die Hundeschule."}
```
Status 403.

---

## Admin-Freigabe

- Neue Kunden mit approved=False werden in der Kundenliste markiert
- Admin kann per Klick freischalten (setzt approved=True)
- PUT /api/admin/users/{id}/approve — neuer Endpoint
- E-Mail an Kunden bei Freigabe

---

## Join-Code Verwaltung

- Auto-generiert bei Schul-Erstellung
- Migration generiert Codes fuer bestehende Schulen
- Sichtbar in Schuleinstellungen (Admin)
- Neu-generieren-Button fuer den Admin
- customer_approval_required Toggle in Schuleinstellungen

---

## Betroffene Dateien

| Datei | Aenderung |
|-------|----------|
| models/schools.py | join_code, customer_approval_required Felder |
| models/auth.py | approved Feld auf UserSchoolRole |
| migrations/versions/ | Neue Migration fuer alle 3 Felder |
| blueprints/api_public.py | Neuer Blueprint: search + by-code |
| blueprints/api_auth.py | register-customer Endpoint + Login-Check |
| blueprints/api_admin/users.py | approve Endpoint |
| templates/admin/users.html | Freigabe-Anzeige + Button |
| blueprints/saas.py oder admin/settings | Join-Code Anzeige + Regenerate |
