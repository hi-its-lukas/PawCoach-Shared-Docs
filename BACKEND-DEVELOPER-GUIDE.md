# Backend Developer Guide

**Stand:** 2026-03-18
**Repo:** `Dog-School-Manager`

## Zweck

Dieser Leitfaden fasst den aktuell implementierten Backend-Stand zusammen. Er ersetzt
keine Details in `openapi.json`, sondern zeigt die kanonischen Stellen fuer Produktlogik,
Entitlements, Pricing, Standorte und Messaging-Realtime.

## Kanonische Referenzen

- `CROSS-PLATFORM-FEATURE-SPEC.md`
- `CROSS-PLATFORM-SYNC-STRATEGY.md`
- `openapi.json`
- `reports/2026-03-18-platform-alignment-status-v7.md`

## Verbindliche Backend-Bausteine

### 1. Pricing und Entitlements

- Der kanonische Pricing-Katalog liegt in `pricing_catalog.py`.
- Marketing-Namen und technische Plan-IDs sind getrennt:
  - `starter` -> `Start`
  - `professional` -> `Growth`
  - `enterprise` -> `Scale`
- `messaging` ist in allen Plaenen enthalten.
- Der kanonische Gutscheinslug ist `gutscheinsystem`.
- `max_locations` ist Teil des Vertrags und steuert Mehr-Standort-Betrieb.

### 2. Capability-Contract

- Verbindlicher Endpoint: `GET /api/auth/capabilities`
- Response: `context`, `subscription`, `addons`, `features`, `limits`, `contract_version`
- Web und iOS muessen darauf aufbauen; Backend bleibt dennoch letzte Autoritaet fuer
  Rechtepruefungen.

### 3. Public Pricing Contract

- Verbindlicher Public-Endpoint: `GET /api/public/pricing`
- Landingpage, Registrierung und Billing muessen sich aus demselben Katalog speisen.
- Keine zweite statische Planquelle in Templates oder Views.

### 3a. Billing- und Kuendigungslogik

- Monats- und Jahresabos nutzen getrennte Stripe-Price-IDs.
- `billing_period` muss in Checkout, Webhooks, Profilstatus und Admin-Responses
  konsistent gespiegelt werden.
- Das Stripe-Portal ist nur fuer Zahlungsdaten und Zahlungsarten gedacht.
- Kuendigungen laufen ueber den serverseitigen Flow:
  - `POST /api/admin/subscription/cancel`
  - `POST /api/admin/subscription/cancel/undo`
- Beim Vormerken ist `reason` Pflicht, `note` optional.
- Die 1-Monats-Frist wird serverseitig berechnet; bei Jahresabos gegen die naechste
  Verlaengerung.

### 4. Standort-System

- Kanonisches Datenmodell: `SchoolLocation`
- CRUD: `GET/POST/PUT/DELETE /api/admin/locations[/{id}]`
- Kurse und Sessions referenzieren optional `location_id`
- Session-Standort ueberschreibt den Kurs-Standardstandort
- Serialisierte Admin-Responses enthalten `location_id` und `location_object`

### 5. Messaging Realtime

- REST bleibt Write-Path
- Socket.IO bleibt Live-State
- Verbindliche Events:
  - `new_message`
  - `message_receipts_updated`
  - `presence_update`
  - `reaction_update`
  - `poll_update`
- `POST /api/messages/{conv_id}` akzeptiert optional `client_message_id`

### 6. Statische Standort-Nachrichten und Treffpunkte

- `POST /api/messages/{conv_id}/location` ist der kanonische Endpoint fuer statische
  Standort-Nachrichten.
- Der Endpoint akzeptiert entweder:
  - direkte Koordinaten (`lat/lng` oder `latitude/longitude`)
  - oder eine freie `address`, die serverseitig geocodiert wird
- Optional zulaessig:
  - `name` oder `label`
  - `body`
- Persistiert werden dabei mindestens:
  - `location_lat`
  - `location_lng`
  - `location_label`
  - `location_address`
- Das Feature muss bewusst auch wechselnde Trainingsorte unterstuetzen, nicht nur den
  aktuellen GPS-Standort des sendenden Users.

## Wichtige Implementierungsorte

| Thema | Primare Dateien |
|------|------------------|
| Pricing-Katalog | `pricing_catalog.py`, `blueprints/saas.py`, `blueprints/api_public.py` |
| Capability-Logik | `capability_service.py`, `plan_limits.py`, `blueprints/api_auth.py` |
| Standort-System | `models/schools.py`, `models/courses.py`, `blueprints/api_admin/locations.py` |
| Messaging Realtime | `blueprints/api_messaging.py`, `socketio_handlers.py` |
| Web-Bruecke | `app.py`, `templates/base.html`, `templates/customer/customer_base.html`, `templates/admin/admin_base.html` |

## Aenderungsregeln

1. Erst Contract und Datenmodell aendern, dann UI.
2. Neue REST-Pfade anschliessend in `openapi.json` aktualisieren.
3. Neue Produktlogik nie nur in Templates oder Clients abbilden.
4. Fertige Plaene aus `plans/` in stabile Doku ueberfuehren und danach loeschen.
