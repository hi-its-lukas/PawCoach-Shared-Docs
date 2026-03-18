# Platform Alignment Status v7

**Datum:** 2026-03-18 (finaler Stand)
**Scope:** Backend, Web und iOS nach den Pricing-, Capability-, Realtime- und
Standort-Aenderungen

## Kurzfazit

Die gemeinsame Architektur ist jetzt vollstaendig konsistent:

- Pricing und Entitlements basieren auf einem kanonischen Katalog
- der Capability-Contract ist die verbindliche Freigabequelle
- Messaging-Realtime ist als REST + Socket.IO Contract dokumentiert
- das Standort-System ist als echtes Datenmodell end-to-end integriert
- alle sichtbaren UI-Gates nutzen den Capability-Contract

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
- Wichtige Feature-Keys fuer die Clients sind jetzt explizit:
  - `admin_dashboard`
  - `forum_moderation`
  - `multi_location_management`
  - `messaging`
  - `community_forum`
- iOS nutzt diese Keys an allen sichtbaren UI-Stellen:
  - `admin_dashboard` fuer Verwaltungstab und Admin-Guard
  - `forum_moderation` fuer Pin/Lock/Delete in Forum-Views
  - `max_locations` fuer Standort-Hinzufuegen-Button in Schuleinstellungen

### Standort-System

- `SchoolLocation` ist das zentrale Modell
- Admin-CRUD fuer Standorte ist serverseitig verfuegbar
- Kurse und Sessions koennen per `location_id` an Standorte gebunden werden
- Admin-Responses liefern `location_object` fuer Client-Darstellung
- iOS und Web nutzen den strukturierten Standortpfad end-to-end:
  - `AdminLocation`, `AdminLocationsResponse`, `location_id`, `location_object`
  - School-Settings-CRUD mit `max_locations`-Limit-Pruefung
  - Kurs-, Session- und Makeup-Flows
  - Einzelstunden-Bestaetigung mit optionalem `location_id`

### Messaging Realtime

- `new_message`, `message_receipts_updated` und `presence_update` sind Teil des aktiven
  Contracts
- `client_message_id` ist fuer REST-Senden und iOS-Reconciliation dokumentiert

## Noch bekannte Grenzen

- `location` bleibt weiterhin als bewusster Freitext-Fallback fuer Sonderfaelle erhalten.
- `openapi.json` muss nach Backend-Aenderungen immer neu generiert werden, sonst bleibt
  Markdown aktueller als der maschinenlesbare Contract.

## iOS-Recheck: Abgeschlossen

Der finale iOS-Gegencheck hat alle Pruefpunkte verifiziert:

1. Contract-Ebene: Alle Models, CodingKeys und Endpoints stimmen mit OpenAPI ueberein.
2. Endpoint-Ebene: Alle 4 Location-Endpoints identisch mit OpenAPI.
3. Flow-Ebene: Kurs-, Session-, Makeup- und Settings-Flows nutzen konsistent `location_id`.
4. Capability-Gating: `admin_dashboard`, `forum_moderation` und `max_locations` greifen
   an allen sichtbaren UI-Stellen.
5. Keine role-only Feature-Gates mehr im produktiven Flow (nur Datenfilterung verbleibt).
6. Einzelstunden-Bestaetigung sendet optional `location_id` und bleibt ohne Auswahl
   kompatibel zum Backend-Fallback.

Detaillierter Report: `reports/2026-03-18-ios-location-recheck.md`

## Doku-Policy ab jetzt

- Aktive Arbeit -> `plans/`
- Stabile Wahrheit -> Shared Specs, Guides, Reports, OpenAPI
- Erledigte Plaene werden geloescht
