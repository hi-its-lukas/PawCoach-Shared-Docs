# Web Developer Guide

**Stand:** 2026-03-18
**Repo:** `Dog-School-Manager`

## Zweck

Dieser Leitfaden beschreibt den aktuellen Soll- und Ist-Stand fuer das browserbasierte
Frontend nach dem Login sowie fuer die oeffentlichen Web-Flows vor dem Login.

## Kanonische Referenzen

- `CROSS-PLATFORM-FEATURE-SPEC.md`
- `CROSS-PLATFORM-SYNC-STRATEGY.md`
- `openapi.json`
- `reports/2026-03-18-platform-alignment-status-v7.md`

## Vor dem Login

- Pricing, Registrierung und Upsell muessen auf demselben Produktkatalog beruhen.
- Oeffentliche Referenz fuer Preise und Paketdarstellung ist `GET /api/public/pricing`.
- Marketing-Texte duerfen oberflaechlich abweichen, nicht aber Plan- oder Addon-Inhalte.

## Nach dem Login

### Capability-driven UI

- Tabs, Navigation, Sidebars, Menues und Deep Links werden aus dem Capability-Contract
  abgeleitet.
- `403` bleibt serverseitige Absicherung, ersetzt aber kein sauberes UI-Gating.
- Rolle allein ist nicht ausreichend; relevant sind auch Plan, Addons und Limits.

### Standort-System

- Der Web-Client arbeitet gegen `GET/POST/PUT/DELETE /api/admin/locations[/{id}]`.
- Kurs- und Session-Formulare koennen `location_id` setzen.
- Anzeige und Editierung sollen bevorzugt mit `location_object` arbeiten, nicht mit
  alten Freitext-Feldern allein.

### Messaging

- REST fuer Laden und Mutationen
- Socket.IO fuer Live-Status
- Keine stillen Polling-Workarounds, wenn ein Live-Event verfuegbar ist

## Wichtige Backend-Bruecken fuer das Web

| Thema | Einstieg |
|------|----------|
| Capability-Gating in Templates | `app.py` Jinja-Helpers + Basistemplates |
| Pricing | `pricing_catalog.py`, `blueprints/saas.py`, `GET /api/public/pricing` |
| Standorte | `blueprints/api_admin/locations.py`, Admin-Course-/Session-APIs |
| Messaging Realtime | Socket.IO Event-Contract aus der Shared Spec |

## Aenderungsregeln

1. Keine neue Produktlogik nur in Jinja oder Frontend-JavaScript einfuehren.
2. Keine manuell gepflegten Preistabellen neben dem Katalog.
3. Keine role-only Navigation fuer plan- oder addon-sensitive Bereiche.
4. Nach jeder groesseren Aenderung Shared Docs und OpenAPI mitpruefen.
