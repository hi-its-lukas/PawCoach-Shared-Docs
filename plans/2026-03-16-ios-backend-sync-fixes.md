# iOS-Backend Sync Fixes — Vollstaendige Uebereinstimmung herstellen

**Datum:** 2026-03-16
**Ziel:** 100% Uebereinstimmung zwischen iOS-App, Backend-API und Cross-Platform-Spec
**Prioritaet:** KRITISCH — mehrere iOS-Calls liefern aktuell 404/400

---

## Uebersicht

| # | Problem | Schweregrad | Auswirkung |
|---|---------|-------------|------------|
| 1 | `markDelivered` falscher Pfad | KRITISCH | 404 bei jedem Zustellstatus |
| 2 | `messageInfo` falscher Pfad | KRITISCH | 404 bei Nachrichteninfo |
| 3 | TOTP-Setup fehlt `setup_token` | KRITISCH | 400 bei 2FA-Einrichtung |
| 4 | Forum-Thread `content` statt `body` | HOCH | 400 bei Thread-Erstellung |
| 5 | Poll falsche Feldnamen | HOCH | 400 bei Umfrage-Erstellung |
| 6 | Register fehlt Consent-Felder | HOCH | 400 bei Registrierung |
| 7 | Undokumentierte Socket-Events | MITTEL | Fehlende Echtzeit-Features |
| 8 | Fehlende Spec-Eintraege | NIEDRIG | Doku-Luecke |

---

## 1. KRITISCH: `markDelivered` — Falscher Pfad

### Ist-Zustand (iOS)
```swift
// Endpoint+Messaging.swift
case .markDelivered(let conversationId):
    // iOS ruft: POST /api/messages/{conversationId}/mark-delivered
```

### Soll-Zustand (Backend)
```
POST /api/messages/{conv_id}/delivered
```
Pfad ist `/delivered`, NICHT `/mark-delivered`.

### Fix in iOS
```swift
case .markDelivered(let conversationId):
    return "/api/messages/\(conversationId)/delivered"
```

### Request/Response
- **Method:** POST
- **Auth:** JWT oder Session
- **Body:** leer (kein JSON noetig)
- **Response 200:**
```json
{
  "status": "ok",
  "conversation_id": 42,
  "delivered_count": 5
}
```

---

## 2. KRITISCH: `messageInfo` — Falscher Pfad

### Ist-Zustand (iOS)
```swift
// Endpoint+Messaging.swift
case .messageInfo(let messageId):
    // iOS ruft: GET /api/messages/{messageId}/info
```

### Soll-Zustand (Backend)
```
GET /api/messages/info/{message_id}
```
`/info/` kommt VOR der Message-ID, nicht danach.

### Fix in iOS
```swift
case .messageInfo(let messageId):
    return "/api/messages/info/\(messageId)"
```

### Response 200
```json
{
  "message_id": 123,
  "sender_id": 5,
  "sender_name": "Max Mustermann",
  "sent_at": "2026-03-16T10:30:00+00:00",
  "delivered_to": [
    {"user_id": 8, "name": "Anna", "delivered_at": "2026-03-16T10:30:05+00:00"}
  ],
  "read_by": [
    {"user_id": 8, "name": "Anna", "read_at": "2026-03-16T10:31:00+00:00"}
  ]
}
```

---

## 3. KRITISCH: TOTP Verify-Setup — Fehlendes `setup_token`

### Ist-Zustand (iOS)
```swift
// Endpoint+Auth.swift
case .totpVerifySetup(let code):
    // iOS sendet: {"code": "123456"}
```

### Soll-Zustand (Backend)
```
POST /api/auth/2fa/totp/verify-setup
```
Backend erwartet ZWEI Felder: `code` UND `setup_token`.

### Ablauf
1. `POST /api/auth/2fa/totp/setup` → Response enthaelt `setup_token`, `qr_uri`, `manual_key`
2. User scannt QR-Code und gibt Code ein
3. `POST /api/auth/2fa/totp/verify-setup` mit `code` + `setup_token` aus Schritt 1

### Fix in iOS
```swift
// 1. Endpoint-Definition erweitern
case .totpVerifySetup(let code, let setupToken)

// 2. Body anpassen
case .totpVerifySetup(let code, let setupToken):
    return ["code": code, "setup_token": setupToken]
```

### Request Body
```json
{
  "code": "123456",
  "setup_token": "eyJhbGciOiJSUzI1NiIs..."
}
```

### Response 200
```json
{
  "status": "ok",
  "message": "Authenticator-App erfolgreich eingerichtet",
  "backup_codes": ["a1b2c3d4", "e5f6g7h8", "..."]
}
```

**Wichtig:** `backup_codes` dem User anzeigen und zum Speichern auffordern! Werden nur einmal zurueckgegeben.

---

## 4. HOCH: Forum-Thread — `content` → `body`

### Ist-Zustand (iOS)
```swift
// Endpoint+Messaging.swift
case .createForumThread(let categoryId, let title, let content):
    return ["title": title, "content": content]
```

### Soll-Zustand (Backend)
```
POST /api/forum/categories/{category_id}/threads
```
Backend liest `body`, nicht `content`.

### Fix in iOS
```swift
case .createForumThread(let categoryId, let title, let body):
    return ["title": title, "body": body]
```

### Response 201
```json
{
  "id": 1,
  "title": "Mein Thread",
  "body": "Inhalt...",
  "author_id": 5,
  "author_name": "Max",
  "created_at": "2026-03-16T10:00:00+00:00"
}
```

---

## 5. HOCH: Poll-Erstellung — Falsche Feldnamen + Struktur

### Ist-Zustand (iOS)
```swift
// Endpoint+Messaging.swift
case .createPoll(let convId, let question, let options):
    return ["question": question, "options": options]
```

### Soll-Zustand (Backend)
```
POST /api/messages/{conv_id}/poll
```
Komplett andere Struktur — Backend unterstuetzt Multi-Fragen-Umfragen.

### Fix in iOS
```swift
case .createPoll(let convId, let title, let questions):
    return [
        "title": title,
        "questions": questions,
        "is_anonymous": false
    ]
```

### Vollstaendige Request-Struktur
```json
{
  "title": "Umfrage-Titel",
  "is_anonymous": false,
  "expires_at": "2026-03-17T18:00:00+00:00",
  "questions": [
    {
      "question_text": "Welcher Termin passt euch?",
      "question_type": "single_choice",
      "options": ["Montag 10:00", "Mittwoch 14:00", "Freitag 16:00"]
    }
  ]
}
```

### Erlaubte `question_type` Werte
| Typ | Beschreibung | Zusatzfelder |
|-----|-------------|--------------|
| `single_choice` | Einzelauswahl | `options`: String-Array |
| `multi_choice` | Mehrfachauswahl | `options`: String-Array |
| `free_text` | Freitext | keine |
| `scale` | Skala | `scale_min`, `scale_max` (Integer) |

### Response 201
```json
{
  "status": "ok",
  "poll_id": 7,
  "message_id": 456
}
```

### Poll-Abstimmung (ebenfalls pruefen!)
```
POST /api/messages/polls/{poll_id}/respond
```
```json
{
  "responses": [
    {
      "question_id": 1,
      "selected_option_id": 3
    }
  ]
}
```
Fuer `multi_choice`: `"selected_option_ids": [2, 3]` statt `selected_option_id`.
Fuer `free_text`: `"free_text": "Antwort"`.
Fuer `scale`: `"scale_value": 4`.

---

## 6. HOCH: Registrierung — Fehlende Consent-Felder

### Ist-Zustand (iOS)
```swift
// Endpoint+Auth.swift
case .register(let email, let password, let name, let schoolName):
    return ["email": email, "password": password, "name": name, "school_name": schoolName]
```

### Soll-Zustand (Backend)
```
POST /api/auth/register
```
Backend verlangt `gdpr_consent` UND `agb_consent` als `true`.

### Fix in iOS
```swift
case .register(let email, let password, let name, let schoolName):
    return [
        "email": email,
        "password": password,
        "name": name,
        "school_name": schoolName,
        "gdpr_consent": true,
        "agb_consent": true
    ]
```

**Wichtig:** Vor dem API-Call MUSS der User in der App explizit:
1. Datenschutzerklaerung akzeptieren (Checkbox/Toggle fuer `gdpr_consent`)
2. AGB akzeptieren (Checkbox/Toggle fuer `agb_consent`)

Beide Werte nur als `true` senden wenn der User aktiv zugestimmt hat (DSGVO-Pflicht).

### Gilt auch fuer Kunden-Registrierung
```
POST /api/auth/register-customer
```
Gleiche Pflichtfelder: `gdpr_consent` + `agb_consent`.

---

## 7. MITTEL: Fehlende/Undokumentierte Socket.IO Events

Die iOS-App nutzt REST-Calls fuer Aktionen. Falls die App kuenftig WebSocket-Support bekommt, hier die vollstaendige Event-Liste:

### Client → Server (iOS sendet)

| Event | Payload | Zweck |
|-------|---------|-------|
| `join_conv` | `{conversation_id: Int}` | Raum beitreten (bei Konversation oeffnen) |
| `leave_conv` | `{conversation_id: Int}` | Raum verlassen (bei Konversation schliessen) |
| `send_message` | `{conversation_id: Int, body: String, reply_to_id?: Int}` | Nachricht senden |
| `typing` | `{conversation_id: Int}` | Tipp-Indikator |
| `add_reaction` | `{message_id: Int, emoji: String}` | Reaktion hinzufuegen |
| `remove_reaction` | `{message_id: Int, emoji: String}` | Reaktion entfernen |
| `poll_response` | `{poll_id: Int, responses: [...]}` | Umfrage beantworten |
| `location_update` | `{session_id: Int, latitude: Float, longitude: Float, accuracy?: Float}` | GPS-Position senden |

### Server → Client (iOS empfaengt)

| Event | Payload | Zweck |
|-------|---------|-------|
| `new_message` | Serialisiertes Message-Objekt | Neue Nachricht |
| `unread_update` | `{}` | Ungelesene-Zaehler aktualisieren |
| `user_typing` | `{user_id: Int, user_name: String}` | Jemand tippt |
| `reaction_update` | `{message_id: Int, reactions: [{emoji, count, users}]}` | Reaktion geaendert |
| `message_edited` | `{message_id: Int, body: String, edited_at: String}` | Nachricht bearbeitet |
| `message_deleted` | `{message_id: Int, deleted_by_name: String}` | Nachricht geloescht |
| `poll_update` | `{poll_id: Int, total_respondents: Int}` | Umfrage aktualisiert |
| `conv_updated` | `{conv_id, display_name, conv_type, status, last_message: {sender_name, body, created_at}}` | Konversation aktualisiert |
| `live_location_started` | `{session_id, conversation_id, started_by, started_by_name, duration_minutes, expires_at}` | Live-Standort gestartet |
| `live_location_ended` | `{session_id, conversation_id}` | Live-Standort beendet |
| `location_update` | `{session_id, user_id, user_name, latitude, longitude, accuracy}` | Position aktualisiert |
| `error` | `{message: String}` | Fehlermeldung |

---

## 8. NIEDRIG: Fehlende Spec-Eintraege — In Spec nachtragen

### 8.1 Admin Setup Wizard

Bereits in iOS und Backend vorhanden, fehlt in der Spec:

| Endpoint | Method | Beschreibung |
|----------|--------|-------------|
| `/api/admin/setup/status` | GET | Setup-Status abfragen |
| `/api/admin/setup/complete` | POST | Setup abschliessen (Body: `{name, email, phone, address, city, zip_code, description, website}`) |
| `/api/admin/setup/skip` | POST | Setup ueberspringen |

### 8.2 Customer Accounting

| Endpoint | Method | Beschreibung |
|----------|--------|-------------|
| `/api/customer/accounting/status` | GET | Buchhaltungs-Status |
| `/api/customer/accounting/sync-jobs` | GET | Sync-Jobs auflisten |
| `/api/customer/accounting/sync-jobs/{id}/retry` | POST | Job erneut versuchen |
| `/api/customer/accounting/retry-all-failed` | POST | Alle fehlgeschlagenen erneut |
| `/api/customer/accounting/sync-now` | POST | Sofort synchronisieren |
| `/api/customer/accounting/test-connection` | POST | Verbindung testen |

### 8.3 Notification Preferences — Doppelte Endpoints

Es existieren ZWEI identische Endpoints:
- `GET/PUT /api/auth/notification-preferences` — fuer ALLE Rollen
- `GET/PUT /api/customer/notification-preferences` — nur fuer Kunden

**Empfehlung:** iOS sollte `/api/auth/notification-preferences` verwenden (funktioniert fuer alle Rollen).

### Felder (GET/PUT):
```json
{
  "email_session_reminder": true,
  "email_booking_confirmation": true,
  "email_course_update": true,
  "email_dog_birthday": true,
  "email_owner_birthday": true,
  "email_new_message": true,
  "push_new_message": true,
  "push_session_reminder": true
}
```

---

## 9. Spec-Korrekturen (CROSS-PLATFORM-FEATURE-SPEC.md)

Nach Abschluss der iOS-Fixes muessen folgende Stellen in der Spec korrigiert werden:

| Zeile/Abschnitt | Alt | Neu |
|-----------------|-----|-----|
| Forum-Thread Body | `{title, content}` | `{title, body}` |
| Poll Body | nicht dokumentiert | Vollstaendige Struktur (siehe Abschnitt 5) |
| Socket Events | 8 Events | Alle 19 Events (siehe Abschnitt 7) |
| Admin Setup Wizard | fehlt | Neuer Abschnitt |
| Customer Accounting | fehlt | Neuer Abschnitt |
| Notification Preferences | nur `/api/customer/` | Beide Pfade + Empfehlung |

---

## Checkliste fuer iOS-Entwickler

- [ ] Fix 1: `markDelivered` Pfad: `/mark-delivered` → `/delivered`
- [ ] Fix 2: `messageInfo` Pfad: `/{id}/info` → `/info/{id}`
- [ ] Fix 3: `totpVerifySetup` um `setup_token` Parameter erweitern
- [ ] Fix 4: Forum-Thread Feld: `content` → `body`
- [ ] Fix 5: Poll komplett ueberarbeiten: `{question, options}` → `{title, questions: [{question_text, question_type, options}]}`
- [ ] Fix 6: Register + Register-Customer: `gdpr_consent` + `agb_consent` hinzufuegen + UI-Checkboxen
- [ ] Fix 7: Notification Preferences auf `/api/auth/notification-preferences` umstellen (falls aktuell `/api/customer/` verwendet)
- [ ] Testen: Jeden gefixten Endpoint manuell gegen das Backend testen
- [ ] Spec: CROSS-PLATFORM-FEATURE-SPEC.md im Shared-Docs-Repo aktualisieren und Submodule anheben
