# Backend-Arbeit: Passkey-Verwaltung & Privacy Visibility

Datum: 2026-03-21

Die iOS-App hat die UI fuer zwei Features implementiert, die folgende Backend-Endpoints benoetigen.

---

## 1. Passkey-Verwaltung

Die App kann bereits Passkeys registrieren und zur Anmeldung nutzen. Fuer die Verwaltung (Auflisten, Loeschen, Umbenennen) werden drei neue Endpoints benoetigt.

### Neue Endpoints

#### `GET /api/auth/passkey/list`

Gibt alle registrierten Passkeys des authentifizierten Nutzers zurueck.

**Auth:** JWT erforderlich

**Response (200):**
```json
{
  "passkeys": [
    {
      "id": 1,
      "name": "iPhone von Lukas",
      "created_at": "2026-03-15T10:30:00Z",
      "last_used_at": "2026-03-20T08:15:00Z"
    }
  ]
}
```

**Hinweise:**
- `name` sollte beim Registrieren automatisch gesetzt werden (z.B. User-Agent oder "Passkey 1", "Passkey 2")
- `last_used_at` wird bei jedem erfolgreichen Login mit diesem Passkey aktualisiert
- `created_at` ist der Zeitpunkt der Registrierung

---

#### `DELETE /api/auth/passkey/{id}`

Entfernt einen registrierten Passkey.

**Auth:** JWT erforderlich

**Response (200):**
```json
{
  "status": "success",
  "message": "Passkey entfernt"
}
```

**Fehler:**
- `404` wenn Passkey nicht existiert oder nicht dem Nutzer gehoert
- `409` wenn es der letzte 2FA-Faktor ist und 2FA aktiv (optional: Warnung statt Blockade)

---

#### `PATCH /api/auth/passkey/{id}`

Benennt einen Passkey um.

**Auth:** JWT erforderlich

**Request Body:**
```json
{
  "name": "Neuer Name"
}
```

**Response (200):**
```json
{
  "status": "success",
  "message": "Passkey aktualisiert"
}
```

### Bestehende Endpoints anpassen

- **`POST /api/auth/passkey/register/complete`**: Sollte einen automatisch generierten `name` fuer den Passkey speichern
- **`POST /api/auth/passkey/login/complete`**: Sollte `last_used_at` beim erfolgreichen Login aktualisieren

### Datenbank

Die bestehende Passkey-Tabelle benoetigt ggf. folgende Spalten (falls nicht vorhanden):
- `name` (VARCHAR, NOT NULL, default: "Passkey")
- `last_used_at` (TIMESTAMP, nullable)

---

## 2. Privacy Visibility Settings

Die Endpoints sind in der CROSS-PLATFORM-FEATURE-SPEC.md bereits spezifiziert (Zeilen 786-800). Falls sie im Backend noch nicht implementiert sind:

### Endpoints

#### `GET /api/auth/profile/visibility`

Gibt die Sichtbarkeitseinstellungen des authentifizierten Nutzers zurueck. Erstellt automatisch Defaults bei erstem Aufruf.

**Auth:** JWT erforderlich

**Response (200):**
```json
{
  "discoverable": true,
  "last_name_full": true,
  "profile_photo_visibility": "all",
  "dogs_visibility": "trainers",
  "dog_owners_visibility": "trainers",
  "phone_visibility": "nobody",
  "last_seen_visibility": "all"
}
```

---

#### `PATCH /api/auth/profile/visibility`

Aktualisiert Sichtbarkeitseinstellungen. Akzeptiert sowohl camelCase als auch snake_case.

**Auth:** JWT erforderlich

**Request Body:** (alle Felder optional, nur geaenderte senden)
```json
{
  "discoverable": false,
  "phone_visibility": "trainers"
}
```

**Response (200):** Komplettes Settings-Objekt (wie GET)

**Validierung:**
- Visibility-Werte muessen `"all"`, `"trainers"` oder `"nobody"` sein
- Boolean-Felder: `true`/`false`

### Datenbank

Neue Tabelle `user_visibility_settings`:
| Spalte | Typ | Default |
|--------|-----|---------|
| user_id | INTEGER FK | - |
| discoverable | BOOLEAN | true |
| last_name_full | BOOLEAN | true |
| profile_photo_visibility | VARCHAR | "all" |
| dogs_visibility | VARCHAR | "trainers" |
| dog_owners_visibility | VARCHAR | "trainers" |
| phone_visibility | VARCHAR | "nobody" |
| last_seen_visibility | VARCHAR | "all" |

---

## Prioritaet

1. **Privacy Visibility** — einfacher zu implementieren, kein Passkey-Handling noetig
2. **Passkey List/Delete** — Kernfunktion fuer die Verwaltung
3. **Passkey Rename** — Nice-to-have, niedrige Prioritaet
