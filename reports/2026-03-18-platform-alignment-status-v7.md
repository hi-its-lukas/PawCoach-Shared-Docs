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
- iOS hat dafuer bereits vorbereitete Contract-Modelle und Endpoints
  (`AdminLocation`, `AdminLocationsResponse`, `location_id`, `location_object`)

### Messaging Realtime

- `new_message`, `message_receipts_updated` und `presence_update` sind Teil des aktiven
  Contracts
- `client_message_id` ist fuer REST-Senden und iOS-Reconciliation dokumentiert

## Noch bekannte Grenzen

- Das Standort-System ist jetzt in Backend, klassischem Web-Admin und iOS-Admin
  end-to-end angeschlossen:
  - `SchoolLocation` wird in den Schuleinstellungen verwaltet
  - Kurs-, Session- und Makeup-Flows nutzen `location_id` als primaeren Pfad
  - `location` bleibt nur als bewusster Freitext-Fallback fuer Sonderfaelle erhalten
- `openapi.json` muss nach Backend-Aenderungen immer neu generiert werden, sonst bleibt
  Markdown aktueller als der maschinenlesbare Contract.

## Recheck fuer iOS

Der iOS-Entwickler sollte den aktuellen Stand deshalb wie folgt pruefen:

1. Contract-Ebene:
   `AdminLocation`, `AdminLocationsResponse`, `location_id` und `location_object`
   muessen fuer Admin-Courses, Admin-Sessions und School-Settings vorhanden bleiben.
2. Endpoint-Ebene:
   `GET/POST/PUT/DELETE /api/admin/locations[/{id}]` muessen im Networking weiterhin
   abgebildet sein.
3. Flow-Ebene:
   School-Settings-CRUD sowie Kurs-, Session- und Makeup-Auswahl muessen dieselben
   Standorte konsistent verwenden.
3. Flow-Ebene:
   Admin-Views und ViewModels muessen noch von string-basierten Ortsfeldern auf einen
   echten Standort-Flow umgestellt werden.

## Doku-Policy ab jetzt

- Aktive Arbeit -> `plans/`
- Stabile Wahrheit -> Shared Specs, Guides, Reports, OpenAPI
- Erledigte Plaene werden geloescht
