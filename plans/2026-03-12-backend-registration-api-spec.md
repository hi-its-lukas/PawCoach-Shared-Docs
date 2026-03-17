# Backend-Anweisung: Duale Registrierungs-Flows

**Datum:** 2026-03-12
**Fuer:** Backend-Entwickler (Flask-Repo: Dog-School-Manager)
**Kontext:** Die iOS-App benoetigt neue Endpoints fuer Kunden-Selbstregistrierung bei bestehenden Hundeschulen. Bisher gibt es nur `POST /api/auth/register` (erstellt neue Schule + Admin-Account) und `POST /api/admin/users` (Admin laedt User ein).

---

## Uebersicht der Aenderungen

1. Neuer Endpoint: Kunden-Selbstregistrierung
2. Neue Endpoints: Schulsuche und Code-Validierung
3. Neue Endpoints: Admin-Genehmigungsflow
4. Erweiterung: Schuleinstellungen (Selbstregistrierungs-Konfiguration)
5. Erweiterung: Schul-Code-Verwaltung
6. Erweiterung: apple-app-site-association (Universal Links)

---

## 1. Kunden-Selbstregistrierung

### POST `/api/auth/register-customer`

**Auth:** Keine (oeffentlich)
**Rate-Limit:** 5 Anfragen/Stunde pro IP

**Request:**
```json
{
  "name": "Max Mustermann",
  "email": "max@example.de",
  "password": "sicheres-passwort",
  "school_code": "FIDO-2026"
}
```

**Logik:**
1. `school_code` validieren → Schule finden
2. Pruefen ob Schule aktiv ist und Selbstregistrierung erlaubt
3. Pruefen ob E-Mail bereits vergeben → 409 Conflict
4. User erstellen mit Rolle `customer`, Zuordnung zur Schule
5. Wenn Schule `customer_self_registration == "auto_approve"`:
   - User sofort aktivieren
   - Bestaetigungs-E-Mail senden
   - Response: `{ "status": "confirmation_sent" }`
6. Wenn Schule `customer_self_registration == "require_approval"`:
   - User als `pending_approval` markieren
   - Bestaetigungs-E-Mail senden
   - Admin per Push/E-Mail benachrichtigen (neue Beitrittsanfrage)
   - Response: `{ "status": "pending_approval" }`

**Responses:**
| Code | Body | Beschreibung |
|------|------|--------------|
| 201 | `{ "status": "confirmation_sent" }` | Erfolgreich, sofort freigeschaltet |
| 201 | `{ "status": "pending_approval" }` | Erfolgreich, wartet auf Admin |
| 400 | `{ "error": "Alle Felder sind erforderlich" }` | Fehlende Felder |
| 404 | `{ "error": "Ungueltiger Schul-Code" }` | Code nicht gefunden |
| 409 | `{ "error": "E-Mail bereits registriert" }` | Duplikat |
| 410 | `{ "error": "Diese Hundeschule nimmt keine neuen Kunden auf" }` | Schule deaktiviert oder Registrierung gesperrt |
| 429 | `{ "error": "Zu viele Versuche" }` | Rate-Limit |

---

## 2. Schulsuche und Code-Validierung

### GET `/api/schools/search?q=<suchbegriff>`

**Auth:** Keine (oeffentlich)
**Rate-Limit:** 10 Anfragen/Minute pro IP

**Query-Parameter:**
- `q` (required): Suchbegriff, min. 2 Zeichen

**Logik:**
- Suche nach Schulname (ILIKE/case-insensitive)
- Nur aktive Schulen mit aktivierter Selbstregistrierung
- Max. 10 Ergebnisse
- Sortierung: Relevanz (Prefix-Match zuerst)

**Response (200):**
```json
[
  {
    "id": 42,
    "name": "Hundeschule Fido",
    "city": "Berlin",
    "code": "FIDO-2026"
  },
  {
    "id": 15,
    "name": "Fidos Welpengruppe",
    "city": "Hamburg",
    "code": "FWELP-2026"
  }
]
```

**Response (400):** `{ "error": "Suchbegriff zu kurz (min. 2 Zeichen)" }`

---

### GET `/api/schools/join/:code`

**Auth:** Keine (oeffentlich)
**Rate-Limit:** 10 Anfragen/Minute pro IP

**Logik:**
- Code validieren (case-insensitive)
- Nur aktive Schulen

**Response (200):**
```json
{
  "id": 42,
  "name": "Hundeschule Fido",
  "city": "Berlin",
  "requires_approval": true
}
```

**Response (404):** `{ "error": "Ungueltiger Schul-Code" }`
**Response (410):** `{ "error": "Diese Hundeschule nimmt keine neuen Kunden auf" }`

---

## 3. Admin-Genehmigungsflow

### GET `/api/admin/pending-registrations`

**Auth:** Admin-Token erforderlich
**Beschreibung:** Liste aller Kunden mit Status `pending_approval` fuer die Schule des Admins.

**Response (200):**
```json
[
  {
    "id": 123,
    "name": "Max Mustermann",
    "email": "max@example.de",
    "registered_at": "2026-03-12T14:30:00Z"
  }
]
```

---

### POST `/api/admin/pending-registrations/:id/approve`

**Auth:** Admin-Token erforderlich

**Logik:**
1. User-Status auf `active` setzen
2. User bekommt Push-Benachrichtigung: "Deine Anmeldung bei {Schulname} wurde bestaetigt!"
3. Optional: Willkommens-E-Mail senden

**Response (200):** `{ "status": "approved" }`
**Response (404):** `{ "error": "Anfrage nicht gefunden" }`

---

### POST `/api/admin/pending-registrations/:id/reject`

**Auth:** Admin-Token erforderlich

**Request (optional):**
```json
{
  "reason": "Bitte kontaktiere uns telefonisch"
}
```

**Logik:**
1. User-Status auf `rejected` setzen oder Konto deaktivieren
2. User bekommt E-Mail mit Ablehnungsgrund (falls angegeben)

**Response (200):** `{ "status": "rejected" }`
**Response (404):** `{ "error": "Anfrage nicht gefunden" }`

---

## 4. Schuleinstellungen erweitern

### Datenbank-Aenderungen

**Tabelle `schools` — neue Spalten:**

| Spalte | Typ | Default | Beschreibung |
|--------|-----|---------|--------------|
| `school_code` | `VARCHAR(20)` UNIQUE | Auto-generiert | Beitrittscode (z.B. "FIDO-2026") |
| `customer_self_registration` | `VARCHAR(20)` | `"auto_approve"` | `"auto_approve"` oder `"require_approval"` |
| `self_registration_enabled` | `BOOLEAN` | `true` | Ob Selbstregistrierung ueberhaupt moeglich ist |

**Migration:**
```sql
ALTER TABLE schools ADD COLUMN school_code VARCHAR(20) UNIQUE;
ALTER TABLE schools ADD COLUMN customer_self_registration VARCHAR(20) DEFAULT 'auto_approve';
ALTER TABLE schools ADD COLUMN self_registration_enabled BOOLEAN DEFAULT true;

-- Bestehende Schulen: Codes generieren
-- Format-Vorschlag: Erste 4 Buchstaben des Schulnamens (uppercase) + "-" + Jahr
-- z.B. "Hundeschule Fido" → "HUND-2026"
-- Bei Duplikaten: Zufallssuffix anhaengen
```

**Tabelle `users` — neuer Status-Wert:**
- Bestehender Status-Enum erweitern um `pending_approval`
- Oder neues Feld `approval_status` (falls Status-Feld bereits andere Bedeutung hat)

### Erweiterung bestehender Endpoints

**GET `/api/admin/settings`** — zusaetzliche Felder in der Response:
```json
{
  "...bestehende Felder...",
  "school_code": "FIDO-2026",
  "customer_self_registration": "auto_approve",
  "self_registration_enabled": true
}
```

**PUT `/api/admin/settings`** — neue Felder akzeptieren:
```json
{
  "customer_self_registration": "require_approval",
  "self_registration_enabled": true
}
```

**POST `/api/admin/settings/regenerate-school-code`** (neu):
- Generiert neuen zufaelligen Schul-Code
- Alter Code wird sofort ungueltig
- Response: `{ "school_code": "FIDO-8X3K" }`

---

## 5. Schul-Code-Generierung

**Format-Vorschlag:**
- 4 Buchstaben (aus Schulname) + `-` + 4 alphanumerische Zeichen
- Beispiel: `FIDO-8X3K`
- Gross-/Kleinschreibung egal bei Eingabe (Backend normalisiert)
- Keine verwechselbaren Zeichen (0/O, 1/I/l)

**Auto-Generierung:**
- Bei Schulerstellung automatisch generieren
- Bei Duplikat: Neuen Code generieren (Retry-Loop)

---

## 6. Universal Links (apple-app-site-association)

Die Datei `https://app.pawcoach.de/.well-known/apple-app-site-association` muss folgende Pfade enthalten:

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "A5K5L86876.de.pawcoach.ios",
        "paths": [
          "/join/*",
          "/set-password/*",
          "/confirm/*"
        ]
      }
    ]
  },
  "webcredentials": {
    "apps": ["A5K5L86876.de.pawcoach.ios"]
  }
}
```

### Web-Fallback

Fuer Nutzer ohne installierte App sollten die URLs auf Webseiten verweisen:
- `/join/:code` → Landingpage mit App-Store-Link + "Lade die PawCoach-App"
- `/set-password/:token` → Web-Formular zum Passwort setzen (bestehend?)
- `/confirm/:token` → Bestaetigungsseite (bestehend)

---

## 7. Login-Verhalten bei pending_approval

**GET `/api/auth/me`** — wenn User Status `pending_approval` hat:
```json
{
  "...user-Felder...",
  "approval_status": "pending",
  "school_name": "Hundeschule Fido"
}
```

Die iOS-App zeigt dann einen Wartescreen statt des Dashboards. Der User ist technisch eingeloggt, hat aber keinen Zugriff auf Features.

**Alternative:** Login verweigern mit spezieller Fehlermeldung:
```json
// POST /api/auth/login Response (403)
{
  "error": "pending_approval",
  "message": "Deine Anmeldung wird noch geprueft.",
  "school_name": "Hundeschule Fido"
}
```

Empfehlung: **Alternative (403)** — einfacher, kein Spezial-State in der App.

---

## Zusammenfassung neue Endpoints

| # | Methode | Endpoint | Auth |
|---|---------|----------|------|
| 1 | `POST` | `/api/auth/register-customer` | Nein |
| 2 | `GET` | `/api/schools/search?q=...` | Nein |
| 3 | `GET` | `/api/schools/join/:code` | Nein |
| 4 | `GET` | `/api/admin/pending-registrations` | Admin |
| 5 | `POST` | `/api/admin/pending-registrations/:id/approve` | Admin |
| 6 | `POST` | `/api/admin/pending-registrations/:id/reject` | Admin |
| 7 | `POST` | `/api/admin/settings/regenerate-school-code` | Admin |

**Datenbank:** 3 neue Spalten in `schools`, 1 neuer Status-Wert fuer `users`.

**Prioritaet:** Endpoints 1-3 zuerst (Kunden-Registrierung), dann 4-7 (Admin-Genehmigung).
