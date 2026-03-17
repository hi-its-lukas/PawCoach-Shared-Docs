# Duale Registrierungs-Flows: Design

**Datum:** 2026-03-12
**Status:** Approved

**Goal:** Zwei getrennte Registrierungs-Flows: SaaS-Registrierung (Hundeschule gruenden) und Kunden-Registrierung (bestehender Schule beitreten).

**Architecture:** Unified Registration Flow mit gemeinsamem Einstiegs-Screen (`RegisterChoiceView`), der zu zwei spezialisierten Flows verzweigt. Kunden finden ihre Schule per Einladungslink/QR-Code, manuellem Code oder Namenssuche. Admin kann pro Schule konfigurieren, ob Selbstregistrierung sofort oder nach Genehmigung freigeschaltet wird.

**Tech Stack:** SwiftUI, neue Backend-Endpoints (Flask), Universal Links, CoreImage (QR-Code-Generierung)

---

## 1. Neue Screens & Navigation

### RegisterChoiceView (neuer Einstieg)
- Ersetzt den direkten Link zu RegisterView auf dem Login-Screen
- Zwei Kacheln mit Icons:
  - **"Hundeschule gruenden"** (SF Symbol: `building.2.fill`) → navigiert zu RegisterView (bestehend)
  - **"Als Kunde beitreten"** (SF Symbol: `person.badge.plus`) → navigiert zu CustomerRegisterView (neu)
- Zurueck-Button zum Login

### RegisterView (bestehend, minimale Anpassung)
- Label "Hundeschul-Name" bleibt, Untertitel "Du erstellst eine neue Hundeschule"
- Sonst unveraendert

### CustomerRegisterView (neu, 2 Steps)
- **Step 1 — Schule finden:**
  - Textfeld fuer Schul-Code (z.B. `FIDO-2026`)
  - ODER Suchfeld mit Autocomplete (Schulnamen-Suche)
  - Bei Deep-Link/QR-Code: Schule ist vorausgefuellt, Step wird uebersprungen
  - Zeigt gefundene Schule als Bestaetigungskarte (Name, Logo, Stadt)
- **Step 2 — Registrierung:**
  - Name, E-Mail, Passwort, Passwort bestaetigen (kein Schulname!)
  - Submit → E-Mail-Bestaetigungs-Screen
  - Falls Schule "Admin-Genehmigung" erfordert: Hinweis "Deine Anmeldung wird geprueft"

### SetPasswordView (neu)
- Fuer Einladungs-Flow: User kommt ueber Link mit Token
- Zeigt: Neues Passwort + Bestaetigung
- Ruft `POST /api/auth/set-password` auf

---

## 2. Backend-Endpoints

### Neue Endpoints

| Methode | Endpoint | Beschreibung |
|---------|----------|--------------|
| `POST` | `/api/auth/register-customer` | Kunden-Selbstregistrierung mit Schul-Zuordnung |
| `GET` | `/api/schools/search?q=...` | Schulen nach Name suchen (oeffentlich, rate-limited) |
| `GET` | `/api/schools/join/:code` | Schul-Code validieren, gibt Schulinfo zurueck |
| `GET` | `/api/admin/pending-registrations` | Liste der wartenden Kunden-Anmeldungen |
| `POST` | `/api/admin/pending-registrations/:id/approve` | Beitrittsanfrage genehmigen |
| `POST` | `/api/admin/pending-registrations/:id/reject` | Beitrittsanfrage ablehnen |

### POST /api/auth/register-customer
```json
// Request
{
  "name": "Max Mustermann",
  "email": "max@example.de",
  "password": "sicheres-passwort",
  "school_code": "FIDO-2026"
}

// Response (201)
{
  "status": "pending_approval"   // oder "confirmation_sent"
}
```

### GET /api/schools/search?q=Hundeschule
```json
// Response (200)
[
  { "id": 42, "name": "Hundeschule Fido", "city": "Berlin", "code": "FIDO-2026" }
]
```
- Max. 10 Ergebnisse, nur aktive Schulen
- Kein Auth erforderlich, Rate-Limiting

### GET /api/schools/join/FIDO-2026
```json
// Response (200)
{ "id": 42, "name": "Hundeschule Fido", "city": "Berlin", "requires_approval": true }
```

### Bestehende Endpoints (Erweiterung)
- `POST /api/admin/settings` — neues Feld `customer_self_registration: "auto_approve" | "require_approval"`
- Schul-Code wird automatisch generiert oder vom Admin anpassbar

---

## 3. Deep-Links & QR-Codes

### Universal Links
- `https://app.pawcoach.de/join/:code` → oeffnet CustomerRegisterView mit vorausgefuelltem Schul-Code
- `https://app.pawcoach.de/set-password/:token` → oeffnet SetPasswordView
- `https://app.pawcoach.de/confirm/:token` → bestaetigt E-Mail via confirmEmail(token:)

### Implementierung
- `PawCoachApp.swift` bekommt `.onOpenURL { url in ... }` Handler
- URL-Parsing in `DeepLinkHandler` (neu) mit Cases: `.joinSchool(code)`, `.setPassword(token)`, `.confirmEmail(token)`
- Bei `.joinSchool`: Wenn nicht eingeloggt → CustomerRegisterView oeffnen mit Code, wenn eingeloggt → Schule beitreten

### QR-Code
- Admin kann in Schuleinstellungen einen QR-Code anzeigen/teilen
- QR-Code enthaelt die Join-URL: `https://app.pawcoach.de/join/FIDO-2026`
- Client-seitig aus Schul-Code generiert (CoreImage CIQRCodeGenerator)

---

## 4. Admin-Konfiguration & Genehmigungsflow

### Schuleinstellungen (Admin)
- Neues Feld in SchoolSettingsView: "Kunden-Selbstregistrierung"
  - Segmented Picker: "Sofort freischalten" / "Genehmigung erforderlich"
- Anzeige des Schul-Codes mit Kopieren-Button und QR-Code-Anzeige
- Option den Schul-Code zu regenerieren

### Genehmigungs-Flow (wenn require_approval)
1. Kunde registriert sich → Status `pending_approval`
2. Admin sieht in AdminRequestsView neue Beitrittsanfragen (Badge-Counter)
3. Admin kann genehmigen oder ablehnen
4. Kunde bekommt Push-Benachrichtigung bei Statusaenderung
5. Bei Genehmigung: Kunde kann sich einloggen und sieht sein Dashboard

---

## 5. Fehlerbehandlung & Edge Cases

### Registrierung
- E-Mail bereits vergeben → "Diese E-Mail ist bereits registriert. Melde dich an oder setze dein Passwort zurueck."
- Ungueltiger Schul-Code → "Dieser Code ist ungueltig. Bitte pruefe den Code oder suche nach der Schule."
- Schule deaktiviert → "Diese Hundeschule nimmt aktuell keine neuen Kunden auf."

### Genehmigung
- Kunde loggt sich ein waehrend Status `pending_approval` → Wartescreen: "Deine Anmeldung wird geprueft."
- Admin lehnt ab → Konto wird deaktiviert, Kunde bekommt E-Mail

### Deep-Links
- Abgelaufener Set-Password-Token → Fehlermeldung + "Neuen Link anfordern"-Button
- Join-Link mit unbekanntem Code → Fallback auf manuelle Suche/Code-Eingabe
- App nicht installiert → Webseite zeigt App-Store-Link (bestehende AASA-Config)

### Rate-Limiting
- Schulsuche: max 10 Anfragen/Minute pro IP
- Registrierung: max 5 Versuche/Stunde pro IP
