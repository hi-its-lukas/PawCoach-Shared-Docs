# PawCoach — Cross-Platform Feature-Spezifikation (Unified)

**Version:** 2026-03-17-v6
**Plattformen:** Backend (Flask/Python) | Web-Frontend (Jinja2+JS) | iOS (Swift/SwiftUI)
**Zweck:** Zentrales Dokument fuer ALLE Plattformen. Die kanonische Quelle ist `PawCoach-Shared-Docs`.

> **SYNC-REGEL:** Dieses File wird nur im Shared-Docs-Repository gepflegt.
> - Canonical Repo: `PawCoach-Shared-Docs/CROSS-PLATFORM-FEATURE-SPEC.md`
> - Backend-Checkout: `Dog-School-Manager/docs/shared/CROSS-PLATFORM-FEATURE-SPEC.md`
> - iOS-Checkout: `PawCoach-iOS/docs/shared/CROSS-PLATFORM-FEATURE-SPEC.md`
> - Bei jeder Aenderung: Versionsdatum aktualisieren, im Shared-Repo committen und danach die Submodule aktualisieren.

---

## Inhaltsverzeichnis

1. [Authentifizierung & Benutzerverwaltung](#1-authentifizierung--benutzerverwaltung)
2. [Rollen & Navigation](#2-rollen--navigation)
3. [Trainer-Bereich](#3-trainer-bereich)
4. [Kunden-Bereich](#4-kunden-bereich)
5. [Admin-Bereich (Hundeschule)](#5-admin-bereich-hundeschule)
6. [Plattform-Admin](#6-plattform-admin)
7. [Messaging & Kommunikation](#7-messaging--kommunikation)
8. [Forum](#8-forum)
9. [Einstellungen & Profil](#9-einstellungen--profil)
10. [Offline & Sync (iOS-only)](#10-offline--sync-ios-only)
11. [Push-Benachrichtigungen](#11-push-benachrichtigungen)
12. [SaaS & Billing (Web-only)](#12-saas--billing-web-only)
13. [Plattform-Status-Matrix](#13-plattform-status-matrix)

---

## 1. Authentifizierung & Benutzerverwaltung

### Login-Methoden

| Methode | Backend | Web | iOS | Endpoints |
|---------|:-------:|:---:|:---:|-----------|
| Email + Passwort | Ja | Ja | Ja | Web: `POST /login`, API: `POST /api/auth/login` |
| Passkey (WebAuthn) | Ja | Ja | Ja | `POST /api/auth/passkey/login/begin` + `.../complete` |
| 2FA (TOTP) | Ja | Ja | Ja | Web: `POST /2fa/verify`, API: `POST /api/auth/2fa/verify` |

### Registrierung

| Typ | Endpoint | Backend | Web | iOS |
|-----|----------|:-------:|:---:|:---:|
| Hundeschul-Admin | Web: `POST /register`, API: `POST /api/auth/register` | Ja | Ja | Ja |
| Kunde (Selbst) | Web: `POST /register/customer`, API: `POST /api/auth/register-customer` | Ja | Ja | Ja |
| Schulsuche | `GET /api/public/schools/search?q={query}` | Ja | Ja | Ja |
| Beitrittscode | `GET /api/public/schools/by-code/{code}` | Ja | Ja | Ja |

**Pflichtfelder bei Registrierung (beide Endpoints):**
- `email`, `password`, `name` — Standardfelder
- `school_name` — nur bei Admin-Registrierung
- `school_id` — nur bei Kunden-Registrierung
- `gdpr_consent: true` — **PFLICHT** (Datenschutzeinwilligung, DSGVO)
- `agb_consent: true` — **PFLICHT** (AGB-Zustimmung)

iOS MUSS beide Consent-Felder mit UI-Checkboxen abfragen und als `true` senden.

### Token-Management (API/iOS)

- **Access Token:** JWT, 15 Min. Gueltigkeit
- **Refresh Token:** 30 Tage, in iOS Keychain gespeichert
- **Auto-Refresh:** Bei 401 wird `POST /api/auth/refresh` aufgerufen, parallele Refreshes werden serialisiert
- **Logout:** `POST /api/auth/logout` (Refresh Token mitsenden)

### Web-Session-Management

- Flask-Login mit serverseitigen Sessions
- `httpOnly`, `SameSite=Lax` Cookies

### Auth-Endpoints

| Endpoint | Methode | Zweck |
|----------|---------|-------|
| `/api/auth/me` | GET | Aktuellen User abfragen |
| `/api/auth/forgot-password` | POST | Passwort-Reset anfordern |
| `/api/auth/reset-password` | POST | Passwort-Reset durchfuehren |
| `/api/auth/set-password` | POST | Erstmaliges Passwort setzen |
| `/api/auth/confirm/{token}` | POST | Email bestaetigen |
| `/api/auth/resend-confirmation` | POST | Bestaetigungsmail erneut senden |
| `/api/auth/switch-context` | PUT | Rolle/Schule wechseln |
| `/api/auth/avatar` | POST/DELETE | Avatar hochladen/loeschen |

### 2FA-Verwaltung

| Endpoint | Methode | Zweck |
|----------|---------|-------|
| `/api/auth/2fa/status` | GET | 2FA-Status abfragen |
| `/api/auth/2fa/totp/setup` | POST | TOTP einrichten |
| `/api/auth/2fa/totp/verify-setup` | POST | TOTP-Setup verifizieren |
| `/api/auth/2fa/totp/disable` | POST | TOTP deaktivieren |
| `/api/auth/2fa/backup-codes` | POST | Backup-Codes regenerieren |
| `/api/auth/2fa/disable-all` | POST | Alle 2FA deaktivieren |

### Passkey-Verwaltung

| Endpoint | Methode | Zweck |
|----------|---------|-------|
| `/api/auth/passkey/register/begin` | POST | Passkey registrieren (Start) |
| `/api/auth/passkey/register/complete` | POST | Passkey registrieren (Abschluss) |
| `/api/auth/passkey/login/begin` | POST | Passkey-Login (Start) |
| `/api/auth/passkey/login/complete` | POST | Passkey-Login (Abschluss) |

---

## 2. Rollen & Navigation

| Rolle | Beschreibung | Web-Navigation | iOS-Tabs |
|-------|-------------|----------------|----------|
| **Kunde** | Hundebesitzer | Dashboard, Buchungen, Nachrichten, Profil | Dashboard, Buchungen, Nachrichten, Mehr |
| **Trainer** | Trainiert Kurse | Tagesansicht, Woche, Nachrichten, Profil | Tagesansicht, Woche, Nachrichten, Mehr |
| **Admin** | Hundeschul-Verwaltung | Dashboard, Kurse, Sessions, User, Einstellungen | Dashboard, Verwaltung, Nachrichten, Mehr |
| **Plattform-Admin** | PawCoach-Gesamtverwaltung | Dashboard, Schulen, Einstellungen | Dashboard, Schulen, Einstellungen |

- Benutzer koennen mehrere Rollen haben (z.B. Trainer + Admin)
- Rollenwechsel: Web via Session, API via `PUT /api/auth/switch-context`
- Sichtbarkeit von Tabs, Menues, Buttons und Deep Links wird aus `Rolle + Schulkontext + aktivem Plan + gebuchten Addons + Features` bestimmt, nicht nur aus `role`.
- Backend stellt dafuer einen kanonischen Capability-Contract bereit: `GET /api/auth/capabilities` (implementiert, Contract-Version 1).
  - Query-Parameter: `?school_id=X&role=Y` (optional, Default: aktive Schule/Rolle des Users)
  - Response: `{ context, subscription, addons, features, limits, contract_version }`
  - Auth: JWT oder Session (`@jwt_or_session_required`)
- Web und iOS muessen nicht erlaubte Features proaktiv ausblenden. `403` bleibt Absicherung, ist aber nicht das primaere UI-Gating.

---

## 3. Trainer-Bereich

| Feature | Endpoint | Backend | Web | iOS |
|---------|----------|:-------:|:---:|:---:|
| Tagesansicht | `GET /api/trainer/today` | Ja | Ja | Ja |
| Wochenansicht | `GET /api/trainer/week?week_start={date}` | Ja | Ja | Ja |
| Session-Detail | `GET /api/trainer/session/{id}` | Ja | Ja | Ja |
| Anwesenheit markieren | `POST /api/trainer/attendance/{bookingId}` | Ja | Ja | Ja (offline-faehig) |
| Kalender | `GET /api/trainer/calendar?month={YYYY-MM}` | Ja | Ja | Ja |
| Verfuegbarkeit | `GET/POST/PUT/DELETE /api/trainer/availability[/{id}]` | Ja | Ja | Ja |
| Blockierungen | `GET/POST/DELETE /api/trainer/blockings[/{id}]` | Ja | Ja | Ja |

---

## 4. Kunden-Bereich

### Dashboard

Endpoint: `GET /api/customer/dashboard` — Meine Hunde, Naechste Termine, Aktive Buchungen

### Buchungssystem

| Feature | Endpoint |
|---------|----------|
| Verfuegbare Sessions | `GET /api/customer/sessions?category={cat}&booking_type={type}` |
| Session buchen | `POST /api/customer/sessions/{id}/book` (body: `{dog_id, is_guest}`) |
| Buchung stornieren | `POST /api/customer/bookings/{id}/cancel` |
| Stornierung beantragen | `POST /api/customer/bookings/{id}/cancel-request` |
| Alle Buchungen | `GET /api/customer/bookings` |
| Blockkurs einschreiben | `POST /api/customer/blockcourse/{courseId}/enroll` |
| Flexibler Kurs | `POST /api/customer/flexible/{courseId}/enroll` |

### Einzelstunden

| Feature | Endpoint |
|---------|----------|
| Verfuegbare Slots | `GET /api/customer/einzelstunde/{courseId}/slots` |
| Einzelstunde buchen | `POST /api/customer/einzelstunde/{sessionId}/book` (body: `{dog_id, slot_id}`) |
| Einzelstunde anfragen | `POST /api/customer/einzelstunde/{courseId}/request` (body: `{dog_id, preferred_times, notes}`) |
| Anfrage stornieren | `POST /api/customer/einzelstunde/anfrage/{id}/cancel` |

### Hunde-Verwaltung

| Feature | Endpoint |
|---------|----------|
| Hunde auflisten | `GET /api/customer/dogs` |
| Hund erstellen | `POST /api/customer/dogs` |
| Hund bearbeiten | `PUT /api/customer/dogs/{id}` |
| Hund loeschen | `DELETE /api/customer/dogs/{id}` |
| Foto hochladen | `POST /api/customer/dogs/{id}/photo` |
| Notizen-Zugriff steuern | `POST /api/customer/dogs/{id}/notes-access` |
| Kommentare | `GET/POST/DELETE /api/dogs/{dogId}/comments[/{commentId}]` |
| Dateien | `GET /api/dogs/{dogId}/files`, `POST .../upload`, `DELETE .../{fileId}` |
| Datei teilen | `POST /api/dogs/{dogId}/files/{fileId}/toggle-share` |

### Credits & Bezahlung

| Feature | Endpoint |
|---------|----------|
| Produkte anzeigen | `GET /api/customer/products` |
| Kreditkarten anzeigen | `GET /api/customer/credits` |
| Kreditkarte kaufen | `POST /api/customer/einzelstunde/{courseId}/buy-card` |
| Checkout starten | `POST /api/customer/checkout` (body: `{product_id, dog_id}`) |
| Checkout-Status | `GET /api/customer/checkout/status/{checkoutId}` |
| Gutschein validieren | `POST /api/customer/validate-voucher` |
| Gutscheine anzeigen | `GET /api/customer/vouchers` |

### Rechnungen

| Feature | Endpoint |
|---------|----------|
| Rechnungen auflisten | `GET /api/customer/invoices` |
| Rechnung anzeigen | `GET /api/customer/invoices/{id}` |
| Rechnung als PDF | `GET /api/customer/invoices/{id}/pdf` |

### Profil & Einstellungen

| Feature | Endpoint |
|---------|----------|
| Profil anzeigen/bearbeiten | `GET/PUT /api/customer/profile` |
| Passwort aendern | `POST /api/customer/change-password` |
| Kalender-Abo | `GET /api/customer/calendar-token` |
| Datenexport | `GET /api/customer/data-export` |
| Konto loeschen | `DELETE /api/customer/account` |
| Benachrichtigungen | `GET/PUT /api/customer/notification-preferences` |
| Session-Standort | `GET /api/session-location/{courseId}` |

---

## 5. Admin-Bereich (Hundeschule)

### Dashboard

Endpoint: `GET /api/admin/dashboard` — KPIs: totalCustomers, totalTrainers, totalCourses, totalSessions, todaySessions, weekSessions, activeBookings, revenue.

### Kurse

| Feature | Endpoint |
|---------|----------|
| CRUD | `GET/POST/PUT/DELETE /api/admin/courses[/{id}]` |
| Kurs duplizieren | `POST /api/admin/courses/{id}/duplicate` |

**Kurstypen:** block, flexible, einzelstunde, open_group

### Sessions

| Feature | Endpoint |
|---------|----------|
| CRUD | `GET/POST/PUT/DELETE /api/admin/sessions[/{id}]` |
| Session absagen | `POST /api/admin/sessions/{id}/cancel` |
| Nachholtermin | `POST /api/admin/sessions/{id}/makeup` |
| Direktbuchung | `POST /api/admin/bookings/direct` |

### Benutzer

| Feature | Endpoint |
|---------|----------|
| CRUD | `GET/POST/PUT/DELETE /api/admin/users[/{id}]` |
| 2FA zuruecksetzen | `POST /api/admin/users/{id}/reset-2fa` |
| Passwort-Reset senden | `POST /api/admin/users/{id}/send-password-reset` |
| Einladung erneut senden | `POST /api/admin/users/{id}/resend-invite` |

### Hunde

| Feature | Endpoint |
|---------|----------|
| CRUD | `GET/POST/PUT/DELETE /api/admin/dogs[/{id}]` |
| Foto hochladen | `POST /api/admin/dogs/{id}/photo` |
| Hund zuordnen | `POST /api/admin/users/{userId}/link-dog` |
| Zuordnung aufheben | `POST /api/admin/users/{userId}/unlink-dog/{dogId}` |
| Hunde eines Kunden | `GET /api/admin/customers/{customerId}/dogs` |

### Schuleinstellungen

| Feature | Endpoint |
|---------|----------|
| Einstellungen lesen/speichern | `GET/PUT /api/admin/settings` |
| Hero-Bild | `POST/DELETE /api/admin/settings/hero-image` |
| Absender-Email | `GET/POST/DELETE /api/admin/settings/sender-email` |
| Beitrittscode QR | `GET /api/admin/settings/join-code-qr` |
| Beitrittscode erneuern | `POST /api/admin/settings/regenerate-join-code` |

### Anfragen-Verwaltung

| Feature | Endpoint |
|---------|----------|
| Buchungsanfragen | `GET /api/admin/booking-requests`, `PUT .../{id}` |
| Als gesehen markieren | `POST /api/admin/booking-requests/batch-seen` |
| Einzelstunde-Anfragen | `GET /api/admin/einzelstunde-requests` |
| Einzelstunde bestaetigen/ablehnen | `POST /api/admin/einzelstunde-requests/{id}/{confirm\|reject}` |
| Stornierungsanfragen | `GET /api/admin/cancellation-requests` |
| Stornierung genehmigen/ablehnen | `POST /api/admin/cancellation-requests/{id}/{approve\|reject}` |

### Credits & Gutscheine

| Feature | Endpoint |
|---------|----------|
| Credits | `GET/POST/PUT /api/admin/credits[/{id}]` |
| Gutscheine | `GET/POST/PUT/DELETE /api/admin/vouchers[/{id}]` |
| Gutschein versenden | `POST /api/admin/vouchers/{id}/send` |
| Gutschein-PDF | `GET /api/admin/vouchers/{id}/pdf` |

### Rechnungen

| Feature | Endpoint |
|---------|----------|
| Rechnungen | `GET/POST /api/admin/invoices[/{id}]` |
| Rechnung-PDF | `GET /api/admin/invoices/{id}/pdf` |

### E-Mail-Vorlagen

| Feature | Endpoint |
|---------|----------|
| Vorlagen | `GET /api/admin/email-templates[/{eventType}]` |
| Vorlage bearbeiten | `PUT /api/admin/email-templates/{eventType}` |
| Vorlage zuruecksetzen | `POST /api/admin/email-templates/{eventType}/reset` |

### Kurs-Kategorien & Formate

| Feature | Endpoint |
|---------|----------|
| Kategorien | `GET/POST/PUT/DELETE /api/admin/course-categories[/{id}]` |
| Formate | `GET/POST/PUT/DELETE /api/admin/course-formats[/{id}]` |

### Verfuegbarkeit (Schulweit)

| Feature | Endpoint |
|---------|----------|
| CRUD | `GET/POST/PUT/DELETE /api/admin/availability[/{id}]` |

### Stripe Connect & Zahlungen

| Feature | Endpoint |
|---------|----------|
| Status | `GET /api/admin/stripe-connect/status` |
| Onboarding starten | `POST /api/admin/stripe-connect/onboard` |
| Status aktualisieren | `POST /api/admin/stripe-connect/refresh-status` |
| Dashboard-URL | `GET /api/admin/stripe-connect/dashboard-url` |
| Stripe-Portal | `POST /api/admin/stripe-portal` |
| Zahlungsmethoden | `GET/POST/DELETE /api/admin/payment-methods[/{id}]` |
| Standard setzen | `PUT /api/admin/payment-methods/{id}/default` |

### Abonnement & Addons

| Feature | Endpoint |
|---------|----------|
| Abo-Info | `GET /api/admin/subscription` |
| Plan aendern | `PUT /api/admin/subscription/plan` |
| Abo kuendigen | `POST /api/admin/subscription/cancel` |
| Addons | `GET /api/admin/addons` |
| Addon umschalten/kaufen/deaktivieren | `POST /api/admin/addons/{id}/toggle`, `.../{slug}/checkout`, `.../{slug}/deactivate` |

### Umfragen & Bewertungen

| Feature | Endpoint |
|---------|----------|
| Umfragen | `GET/POST/PUT/DELETE /api/admin/surveys[/{id}]` |
| Umfrage-Dashboard | `GET /api/admin/survey-dashboard` |
| Vorlagen | `GET/POST/PUT/DELETE /api/admin/survey-templates[/{id}]` |
| Bewertungen | `GET /api/admin/reviews` |
| Bewertung genehmigen/ablehnen | `POST /api/admin/reviews/{id}/{approve\|reject}` |

### Weitere Admin-Features

| Feature | Endpoint |
|---------|----------|
| Support-Zugriff | `GET /api/admin/support-access`, `POST .../{id}/{action}` |
| Offene-Gruppen-Genehmigungen | `GET/POST /api/admin/open-group-approvals[/{id}/{action}]` |
| Ausstehende Registrierungen | `GET /api/admin/pending-registrations`, `POST .../{id}/{action}` |
| Buchhaltung | `GET /api/admin/accounting/status`, `.../jobs`, `POST .../retry`, `.../sync-now`, `.../test-connection` |

### Admin Setup Wizard

| Feature | Endpoint | Backend | Web | iOS |
|---------|----------|:-------:|:---:|:---:|
| Setup-Status | `GET /api/admin/setup/status` | Ja | Ja | Ja |
| Setup abschliessen | `POST /api/admin/setup/complete` (body: `{name, email, phone, address, city, zip_code, description, website}`) | Ja | Ja | Ja |
| Setup ueberspringen | `POST /api/admin/setup/skip` | Ja | Ja | Ja |

### Benachrichtigungs-Einstellungen

| Feature | Endpoint | Backend | Web | iOS |
|---------|----------|:-------:|:---:|:---:|
| Einstellungen laden | `GET /api/auth/notification-preferences` | Ja | Ja | Ja |
| Einstellungen speichern | `PUT /api/auth/notification-preferences` | Ja | Ja | Ja |

**Felder:** `email_session_reminder`, `email_booking_confirmation`, `email_course_update`, `email_dog_birthday`, `email_owner_birthday`, `email_new_message`, `push_new_message`, `push_session_reminder` (alle Boolean).

**Hinweis:** Es existiert auch `GET/PUT /api/customer/notification-preferences` (nur Kunden-Rolle). Empfehlung: `/api/auth/notification-preferences` verwenden (funktioniert fuer alle Rollen).

---

## 6. Plattform-Admin

| Feature | Endpoint |
|---------|----------|
| Dashboard | `GET /api/platform/dashboard` |
| Schulen verwalten | `GET/POST/PUT/DELETE /api/platform/schools[/{id}]` |
| Schul-Plan setzen | `POST /api/platform/schools/{id}/set-plan` |
| Zugriff anfordern | `POST /api/platform/schools/{id}/request-access` |
| In Schule eintreten | `POST /api/platform/schools/{id}/enter` |
| Schule verlassen | `POST /api/platform/leave-school` |
| Benutzer verwalten | `GET/POST /api/platform/users`, `.../{id}/revoke`, `.../{id}/reset-2fa` |
| Einstellungen | `GET/PUT /api/platform/settings` |
| E-Mail-Vorlagen | `GET /api/platform/email-templates`, `PUT .../{eventType}` |
| Finanzen | `GET /api/platform/finances` |
| Rechnungen | `GET /api/platform/invoices`, `GET .../{id}/pdf` |
| Wartungsmodus | `GET/PUT /api/platform/maintenance` |
| Wartung planen/absagen | `POST/DELETE /api/platform/maintenance/schedule` |
| Preise | `GET /api/platform/prices`, `PUT .../plans/{planId}`, `PUT .../addons/{priceId}` |
| System-Info | `GET /api/platform/system` |
| Logs | `GET /api/platform/logs?page={p}&per_page={pp}&action={a}` |

---

## 7. Messaging & Kommunikation

### Grundfunktionen

| Feature | Endpoint | Backend | Web | iOS |
|---------|----------|:-------:|:---:|:---:|
| Konversationen laden | `GET /api/messages` | Ja | Ja | Ja |
| Nachrichten laden | `GET /api/messages/{id}` | Ja | Ja | Ja |
| Nachricht senden | `POST /api/messages/{id}` (body: `{body, reply_to_id?}`) | Ja | Ja (Socket.IO) | Ja |
| Ungelesene Anzahl | `GET /api/messages/unread-count` | Ja | Ja | Ja |
| Kontakte laden | `GET /api/messages/contacts` | Ja | Ja | Ja |
| Neue Konversation | `POST /api/messages/new` (body: `{participant_id, message}`) | Ja | Ja | Ja |
| Als gelesen markieren | `POST /api/messages/{id}/read` | Ja | Ja | Ja |

### Nachrichten-Aktionen

| Feature | Endpoint | Backend | Web | iOS |
|---------|----------|:-------:|:---:|:---:|
| Nachricht bearbeiten | `PUT /api/messages/{messageId}/edit` | Ja | Ja | Ja |
| Nachricht loeschen | `DELETE /api/messages/{messageId}` | Ja | Ja | Ja |
| Reaktion hinzufuegen | `POST /api/messages/{messageId}/reactions` (body: `{emoji}`) | Ja | Ja | Ja |
| Reaktion entfernen | `DELETE /api/messages/{messageId}/reactions/{emoji}` | Ja | Ja | Ja |

### Konversations-Verwaltung

| Feature | Endpoint | Backend | Web | iOS |
|---------|----------|:-------:|:---:|:---:|
| Chat schliessen | `POST /api/messages/{id}/close` | Ja | Ja | Ja |
| Chat oeffnen | `POST /api/messages/{id}/reopen` | Ja | Ja | Ja |
| Chat archivieren | `POST /api/messages/{id}/archive` | Ja | Ja | Ja |
| Chat entarchivieren | `POST /api/messages/{id}/unarchive` | Ja | Ja | Ja |
| Chat leeren | `DELETE /api/messages/{id}/clear` | Ja | Ja | Ja |

### Gruppen & Broadcast

| Feature | Endpoint | Backend | Web | iOS |
|---------|----------|:-------:|:---:|:---:|
| Gruppe erstellen | `POST /api/messages/group` (body: `{participant_ids, name}`) | Ja | Ja | Ja |
| Mitglieder laden | `GET /api/messages/{id}/members` | Ja | Ja | Ja |
| Mitglied hinzufuegen | `POST /api/messages/{id}/members` (body: `{user_id}`) | Ja | Ja | Ja |
| Mitglied entfernen | `DELETE /api/messages/{id}/members/{userId}` | Ja | Ja | Ja |
| Broadcast senden | `POST /api/messages/broadcast` | Ja | Ja | Ja |

### Medien & Standort

| Feature | Endpoint | Backend | Web | iOS |
|---------|----------|:-------:|:---:|:---:|
| Medien hochladen | `POST /api/messages/{id}/upload` (multipart) | Ja | Ja | Ja |
| Standort senden | `POST /api/messages/{id}/location` (body: `{lat, lng}`) | Ja | Ja | Ja |
| Live-Standort starten | `POST /api/messages/{id}/live-location` | Ja | Ja | Ja |
| Positionen abfragen | `GET /api/messages/live-location/{sessionId}/positions` | Ja | Ja | Ja |
| Live-Standort stoppen | `DELETE /api/messages/live-location/{sessionId}` | Ja | Ja | Ja (Menu-Button) |
| Standort-Teilen umschalten | `POST /api/messages/{id}/toggle-location-sharing` | Ja | Ja | Ja |

**Standort-Nachrichten (iOS):**
- Standort-Nachrichten (`message_type: "location"`) werden als **Karten-Vorschau** mit Marker dargestellt
- Tap auf die Karte oeffnet **Apple Maps** mit dem geteilten Standort
- Message-Model enthaelt: `latitude: Double?`, `longitude: Double?`, `location_name: String?`
- Live-Standort kann ueber Menu-Button "Live-Standort beenden" gestoppt werden

**Hinweis:** Der `/api/messages/{id}/location` Endpoint darf NICHT blockieren wenn eine Live-Location-Session aktiv ist. Statischer Standort und Live-Location sind unabhaengige Features.

### Umfragen im Chat

| Feature | Endpoint | Backend | Web | iOS |
|---------|----------|:-------:|:---:|:---:|
| Umfrage erstellen | `POST /api/messages/{id}/poll` (body: `{title, questions, is_anonymous?, expires_at?}`) | Ja | Ja | Ja |
| Abstimmen | `POST /api/messages/polls/{pollId}/respond` (body: `{responses: [{question_id, selected_option_id?}]}`) | Ja | Ja | Ja |
| Ergebnisse | `GET /api/messages/polls/{pollId}/results` | Ja | Ja | Ja |
| Umfrage schliessen | `POST /api/messages/polls/{pollId}/close` | Ja | Ja | Ja |

**Poll-Erstellung — `questions`-Struktur:**
```json
{
  "title": "Umfrage-Titel",
  "is_anonymous": false,
  "expires_at": "2026-03-17T18:00:00+00:00",
  "questions": [
    {
      "question_text": "Frage?",
      "question_type": "single_choice|multi_choice|free_text|scale",
      "options": ["Option A", "Option B"],
      "scale_min": 1,
      "scale_max": 5
    }
  ]
}
```

**Abstimmung — `responses`-Struktur:**
- `single_choice`: `{question_id, selected_option_id}`
- `multi_choice`: `{question_id, selected_option_ids: [...]}`
- `free_text`: `{question_id, free_text: "..."}`
- `scale`: `{question_id, scale_value: 4}`

### Blockieren, Melden & Stummschalten

| Feature | Endpoint | Backend | Web | iOS |
|---------|----------|:-------:|:---:|:---:|
| User blockieren | `POST /api/messages/block/{userId}` | Ja | Ja | Ja |
| User entblocken | `DELETE /api/messages/block/{userId}` | Ja | Ja | Ja |
| User melden | `POST /api/messages/report/{userId}` (body: `{reason?}`) | Ja | Ja | Ja |
| Konversation stummschalten | `POST /api/messages/{id}/mute` | Ja | Ja | Ja |
| Stummschaltung aufheben | `DELETE /api/messages/{id}/mute` | Ja | Ja | Ja |
| Stummgeschaltete auflisten | `GET /api/messages/muted` | Ja | - | Ja |

### Zustellstatus & Nachrichteninfo

| Feature | Endpoint | Backend | Web | iOS |
|---------|----------|:-------:|:---:|:---:|
| Als zugestellt markieren | `POST /api/messages/{id}/delivered` | Ja | - | Ja |
| Nachrichteninfo | `GET /api/messages/info/{messageId}` | Ja | - | Ja (Swipe-Left) |
| Chat-Suche | `GET /api/messages/{id}/search?q={term}` | Ja | Ja | Lokal |
| Medien-Zaehler | `GET /api/messages/{id}/media` | Ja | - | Ja |
| Gemeinsame Kurse | `GET /api/messages/shared-courses/{userId}` | Ja | Ja | Ja |

**Zustellstatus-Werte:**

| Status | Icon (iOS) | Bedeutung |
|--------|------------|-----------|
| `sending` | Uhr | Wird gesendet (nur lokal) |
| `sent` | Ein Haken | Server hat empfangen |
| `delivered` | Zwei graue Haken | An Empfaenger zugestellt |
| `read` | Zwei gruene Haken | Vom Empfaenger gelesen |
| `failed` | Ausrufezeichen | Senden fehlgeschlagen |

`mark_as_read` (`POST /api/messages/{id}/read`) setzt automatisch `read_at` auf alle ungelesenen fremden Nachrichten.

### Reply/Quote (Nachricht zitieren)

Beim Senden wird `reply_to_id` mitgeschickt:
```json
POST /api/messages/{id}
{
  "body": "Morgen um 10!",
  "reply_to_id": 42
}
```

Response:
```json
{
  "id": 43,
  "body": "Morgen um 10!",
  "reply_to_id": 42,
  "reply_to_body": "Wann ist die naechste Session?",
  "reply_to_sender_name": "Max Mustermann"
}
```

### Kontaktinfo-Seite / Panel

Beim Tap/Klick auf den Chat-Namen:
- Avatar, Name, Rolle
- Zuletzt online
- Medien/Links/Doks Zaehler
- Hunde des Kontakts + weitere Halter
- Gemeinsame Kurse
- Stumm schalten / Chat leeren / Blockieren / Melden

**iOS-spezifisch:**
- Karten-Vorschau bei Standort-Nachrichten (Tap oeffnet Apple Maps)
- Swipe-Left auf eigene Nachrichten → Nachrichteninfo mit Zustellzeiten
- Swipe-Right auf Nachrichten → Reply/Zitieren
- Live-Standort beenden ueber Menu-Button

### Kontaktliste mit Hunde-Sektion

`GET /api/messages/contacts` liefert:
```json
{
  "contacts": [{"id": 1, "name": "Max", "role": "trainer", ...}],
  "dogs": [{"id": 1, "name": "Bello", "breed": "Labrador", "owners": [{"id": 2, "name": "Anna"}]}]
}
```

**Sichtbarkeitslogik:**
1. Admin/Trainer sehen alle User + alle Hunde
2. Kunde sieht: Trainer + Admins + Halter eigener Hunde + Kunden im selben Kurs
3. Kunde sieht zusaetzlich: Kunden mit `discoverable: true`
4. Profildetails werden gemaess `ProfileVisibility` gefiltert (last_name_full etc.)

### Nachrichten-Datenmodell

```json
{
  "id": 1,
  "sender_id": 5,
  "sender_name": "Max Mustermann",
  "body": "Hallo!",
  "message_type": "text",
  "created_at": "2026-03-15T10:22:00Z",
  "edited_at": null,
  "is_deleted": false,
  "delivered_at": "2026-03-15T10:22:01Z",
  "read_at": "2026-03-15T10:23:00Z",
  "delivery_status": "read",
  "reply_to_id": null,
  "reply_to_body": null,
  "reply_to_sender_name": null
}
```

### Konversations-Datenmodell

```json
{
  "id": 1,
  "conv_type": "direct",
  "subject": null,
  "display_name": "Max Mustermann",
  "last_message": "Hallo!",
  "last_message_at": "2026-03-15T10:22:00Z",
  "unread_count": 2,
  "is_closed": false,
  "is_archived": false,
  "is_muted": false,
  "is_blocked": false,
  "last_seen": "2026-03-15T14:32:00Z",
  "created_by_id": 5,
  "members": [{"id": 1, "user_id": 5, "user_name": "Max Mustermann"}]
}
```

### Real-Time (Socket.IO)

**Client → Server:**

| Event | Payload | Zweck |
|-------|---------|-------|
| `join_conv` | `{conversation_id}` | Raum beitreten (bei Konversation oeffnen) |
| `leave_conv` | `{conversation_id}` | Raum verlassen (bei Konversation schliessen) |
| `send_message` | `{conversation_id, body, reply_to_id?}` | Nachricht senden |
| `typing` | `{conversation_id}` | Tipp-Indikator |
| `add_reaction` | `{message_id, emoji}` | Reaktion hinzufuegen |
| `remove_reaction` | `{message_id, emoji}` | Reaktion entfernen |
| `poll_response` | `{poll_id, responses: [{question_id, selected_option_id?, selected_option_ids?, free_text?, scale_value?}]}` | Umfrage beantworten |
| `location_update` | `{session_id, latitude, longitude, accuracy?}` | GPS-Position senden |

**Server → Client:**

| Event | Payload | Zweck |
|-------|---------|-------|
| `new_message` | Serialisiertes Message-Objekt | Neue Nachricht |
| `unread_update` | `{}` | Ungelesene-Zaehler neu laden |
| `user_typing` | `{user_id, user_name}` | Jemand tippt |
| `reaction_update` | `{message_id, reactions: [{emoji, count, users}]}` | Reaktion geaendert |
| `message_edited` | `{message_id, body, edited_at}` | Nachricht bearbeitet |
| `message_deleted` | `{message_id, deleted_by_name}` | Nachricht geloescht |
| `poll_update` | `{poll_id, total_respondents}` | Umfrage aktualisiert |
| `conv_updated` | `{conv_id, display_name, conv_type, status, last_message: {sender_name, body, created_at}}` | Konversation aktualisiert |
| `live_location_started` | `{session_id, conversation_id, started_by, started_by_name, duration_minutes, expires_at}` | Live-Standort gestartet |
| `live_location_ended` | `{session_id, conversation_id}` | Live-Standort beendet |
| `location_update` | `{session_id, user_id, user_name, latitude, longitude, accuracy}` | Position aktualisiert |
| `error` | `{message}` | Fehlermeldung |

### Live-Location — UI-Verhalten (alle Plattformen)

**Banner / Panel bei aktiver Session:**

Wenn eine Konversation eine aktive `active_live_location` enthaelt (aus `GET /api/messages/{id}` Response), MUSS ein persistentes Banner am oberen Rand des Chat-Views angezeigt werden:

- **Inhalt:** Pulsierender gruener Punkt + "Live-Standort aktiv (Name)" + Countdown-Timer + optional Karten-Toggle + "Beenden"-Button
- **Sichtbarkeit:** Fuer ALLE Chat-Teilnehmer sichtbar, solange die Session aktiv ist
- **"Beenden"-Button:** Nur sichtbar fuer den Session-Starter oder Admin (DELETE `/api/messages/live-location/{sessionId}`)
- **Timer:** Countdown bis `expires_at`, Format: "noch MM:SS"
- **Karte:** Collapsible Map (Leaflet/MapKit) mit Echtzeit-Positionen der teilnehmenden User

**Automatisches Aufraeumen abgelaufener Sessions:**

- **Backend:** `active_live_location` wird bei jedem Konversations-Laden geprueft. Wenn `expires_at < now`, wird `is_active = false` gesetzt und `null` zurueckgegeben.
- **Frontend:** Wenn der Countdown-Timer 0 erreicht:
  1. Timer-Text auf "abgelaufen" setzen
  2. GPS-Tracking stoppen
  3. Nach 2 Sekunden: Banner/Panel ausblenden, `active_live_location` State auf `null` setzen
- **Socket:** `live_location_ended` Event wird vom Starter oder automatisch gesendet → Banner sofort ausblenden

**Konversationswechsel:**

- Beim Wechsel zu einer anderen Konversation: GPS-Tracking stoppen, Panel ausblenden
- Beim Laden einer Konversation: `active_live_location` pruefen und Panel entsprechend anzeigen/verbergen

**iOS-Implementierung:**

- Banner analog zu WhatsApp Live-Location-Banner im Chat
- MapKit anstelle von Leaflet verwenden
- GPS-Tracking via `CLLocationManager` mit `allowsBackgroundLocationUpdates` fuer Hintergrund-Updates
- "Beenden"-Button per `DELETE /api/messages/live-location/{sessionId}` mit JWT-Auth

---

## 8. Forum

| Feature | Endpoint | Backend | Web | iOS |
|---------|----------|:-------:|:---:|:---:|
| Kategorien | `GET /api/forum/categories` | Ja | Ja | Ja |
| Threads | `GET /api/forum/categories/{id}/threads` | Ja | Ja | Ja |
| Thread-Detail | `GET /api/forum/threads/{id}` | Ja | Ja | Ja |
| Thread erstellen | `POST /api/forum/categories/{id}/threads` (body: `{title, body}`) | Ja | Ja | Ja |
| Antworten | `POST /api/forum/threads/{id}/reply` (body: `{content}`) | Ja | Ja | Ja |
| Thread anpinnen | `POST /api/forum/threads/{id}/pin` | Ja | Ja | Ja |
| Thread sperren | `POST /api/forum/threads/{id}/lock` | Ja | Ja | Ja |
| Beitrag loeschen | `POST /api/forum/post/{id}/delete` | Ja | Ja | Ja |

---

## 9. Einstellungen & Profil

### Screens / Seiten

| Screen | Web | iOS |
|--------|:---:|:---:|
| Profil anzeigen/bearbeiten | Ja | Ja |
| Passwort aendern | Ja | Ja |
| 2FA verwalten | Ja | Ja |
| Benachrichtigungen | Ja | Ja |
| Kalender-Abo | Ja | Ja |
| App-Sperre (Face ID / Touch ID) | - | Ja |
| Datenschutz (Export, Loeschen) | Ja | Ja |
| Sichtbarkeit (ProfileVisibility) | Ja | Ja |

### Sichtbarkeitseinstellungen (ProfileVisibility)

| Einstellung | Optionen | Standard | DB-Feld |
|-------------|----------|----------|---------|
| Auffindbar fuer andere Mitglieder | true/false | true | `discoverable` |
| Nachname vollstaendig | true/false | true | `last_name_full` |
| Profilbild | all/trainers/nobody | all | `profile_photo_visibility` |
| Meine Hunde | all/trainers/nobody | trainers | `dogs_visibility` |
| Weitere Halter meiner Hunde | all/trainers/nobody | trainers | `dog_owners_visibility` |
| Telefonnummer | all/trainers/nobody | nobody | `phone_visibility` |
| Zuletzt online | all/trainers/nobody | all | `last_seen_visibility` |

**Endpoints:**
- `GET /api/auth/profile/visibility` — Einstellungen laden (auto-creates defaults)
- `PATCH /api/auth/profile/visibility` — Einstellungen speichern (akzeptiert camelCase + snake_case)

---

## 10. Offline & Sync (iOS-only)

### Architektur

- **Optimistisches Senden:** Nachrichten/Anwesenheit werden sofort lokal angezeigt
- **PendingAction Queue:** Schreibaktionen werden bei fehlendem Netzwerk in SwiftData gespeichert
- **SyncEngine:** Ueberwacht Netzwerk via NWPathMonitor, arbeitet Queue bei Rueckkehr ab
- **Konfliktstrategie:** Server-Daten ueberschreiben lokale Daten bei Sync

| Feature | Offline-Verhalten |
|---------|------------------|
| Anwesenheit markieren | Optimistisch lokal, spaeter synchronisiert |
| Nachricht senden | Lokal angezeigt (Status: "sending"), bei Rueckkehr gesendet |
| Daten lesen | Cached-Daten angezeigt, bei Verbindung aktualisiert |

---

## 11. Push-Benachrichtigungen

### Registrierung

```
POST /api/push/register   { "token": "APNs-Device-Token", "platform": "ios" }
POST /api/push/unregister { "token": "..." }
```

### Kategorien

| Event | Push |
|-------|:----:|
| Neue Nachricht | Ja (mit Mute-Check) |
| Buchungsbestaetigung | Ja |
| Session-Absage | Ja |
| Anfrage genehmigt/abgelehnt | Ja |
| Neue Bewertung (Admin) | Ja |
| Neue Buchungsanfrage (Admin) | Ja |

---

## 12. SaaS & Billing (Web-only)

Checkout und Vertragsverwaltung bleiben webbasiert, aber die resultierenden Entitlements
muessen danach fuer alle Clients ueber den gemeinsamen Capability-Contract verfuegbar sein.

### Web-Routes

| Route | Funktion |
|-------|----------|
| `/pricing` | Preisseite |
| `/register` | Hundeschul-Registrierung |
| `/billing` | Abo-Verwaltung |
| `/addons` | Addon-Verwaltung |

### Stripe-Webhooks

| Event | Aktion |
|-------|--------|
| `checkout.session.completed` | Abo/Addon aktivieren |
| `invoice.paid` | Zahlung bestaetigen |
| `invoice.payment_failed` | Zahlungsfehlschlag verarbeiten |
| `customer.subscription.updated` | Abo-Status aktualisieren |
| `customer.subscription.deleted` | Abo-Kuendigung verarbeiten |

### Plan- und Addon-Regel

- Billing-Checkout ist Web-only.
- Feature-Freischaltung ist nie Web-only.
- Sobald ein Plan oder Addon aktiv ist, muessen Backend, Web und iOS denselben wirksamen Zustand sehen.
- Optional buchbare Addons muessen im Capability-Contract separat ausgewiesen werden.

---

## 13. Plattform-Status-Matrix

### Plattform-spezifische Features

| Feature | Backend | Web | iOS | Bemerkung |
|---------|:-------:|:---:|:---:|-----------|
| Offline-Modus | - | - | Ja | Native SwiftData + SyncEngine |
| App-Sperre (Face ID) | - | - | Ja | Native Biometrie |
| Admin-Verwaltungsoberflaeche (CRUD) | Ja | Ja | Ja | |
| SaaS-Billing (Stripe) | Ja | Ja | - | Web-only Checkout |
| Oeffentliche Schulseiten (SEO) | Ja | Ja | - | Web-only Landingpages |
| E-Mail-Vorlagen-Editor | Ja | Ja | - | Web-only WYSIWYG |
| Zustellstatus-Haekchen | Ja | - | Ja | Web zeigt Status nicht visuell |
| Nachrichteninfo (Swipe) | Ja | - | Ja | Web koennte Dialog nutzen |
| Chat-Suche | Ja | Ja (Server) | Ja (Lokal) | Beide Varianten verfuegbar |

### Statistik

| Metrik | Wert |
|--------|------|
| API-Endpoints gesamt | ~570 |
| Datenmodelle | ~100 |
| Web-UI-Seiten | ~60 |
| iOS-Screens | ~80 |
| Benutzerrollen | 4 |
| Feature-Bereiche | 12 |
| Real-Time Events (Socket.IO) | 8 |
