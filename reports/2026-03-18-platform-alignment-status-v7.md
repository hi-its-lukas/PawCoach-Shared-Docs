# Platform Alignment Status v7

**Datum:** 2026-03-18
**Scope:** Backend, Web und iOS nach den Pricing-, Capability-, Realtime- und
Standort-Aenderungen

## Kurzfazit

Die gemeinsame Architektur ist jetzt deutlich konsistenter dokumentiert als zuvor:

- Pricing und Entitlements basieren auf einem kanonischen Katalog
- der Capability-Contract ist die verbindliche Freigabequelle
- Messaging-Realtime ist als REST + Socket.IO Contract dokumentiert
- das Standort-System ist als echtes Datenmodell eingefuehrt

Der stabile Referenzstand liegt nicht mehr in den einzelnen Umsetzungsplaenen, sondern in
den Shared Specs, den Developer Guides und `openapi.json`.

## Verifizierter Stand

### Pricing und Entitlements

- `pricing_catalog.py` ist die fachliche Quelle fuer Plaene, Addons und Limits
- Marketing-Namen `Start`, `Growth`, `Scale` sind etabliert
- `messaging` ist kein kostenpflichtiges Addon mehr
- `gutscheinsystem` ist der kanonische Gutschein-Slug
- `max_locations` ist Teil des Capability-Contracts

### Capability-Contract

- `GET /api/auth/capabilities` ist die verbindliche Quelle fuer:
  - aktiven Kontext
  - Subscription-Zustand
  - Addons
  - Features
  - Limits

### Standort-System

- `SchoolLocation` ist das zentrale Modell
- Admin-CRUD fuer Standorte ist serverseitig verfuegbar
- Kurse und Sessions koennen per `location_id` an Standorte gebunden werden
- Admin-Responses liefern `location_object` fuer Client-Darstellung

### Messaging Realtime

- `new_message`, `message_receipts_updated` und `presence_update` sind Teil des aktiven
  Contracts
- `client_message_id` ist fuer REST-Senden und iOS-Reconciliation dokumentiert

## Noch bekannte Grenzen

- Eine vollstaendige dedizierte iOS-UI fuer Standort-CRUD ist noch kein stabiler
  Endanwender-Flow; Backend-Contract und App-Modelle sind aber vorhanden.
- `openapi.json` muss nach Backend-Aenderungen immer neu generiert werden, sonst bleibt
  Markdown aktueller als der maschinenlesbare Contract.

## Doku-Policy ab jetzt

- Aktive Arbeit -> `plans/`
- Stabile Wahrheit -> Shared Specs, Guides, Reports, OpenAPI
- Erledigte Plaene werden geloescht
