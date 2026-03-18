# iOS-Entwickler Anleitung — Shared Docs & API Contract

**Stand:** 2026-03-18
**Betrifft:** Alle, die am `PawCoach-iOS`-Repo arbeiten

---

## Grundregel

Geteilte Produkt-, API- und Contract-Dokumentation wird nicht mehr lokal im iOS-Repo
dupliziert. Die kanonische Quelle ist `PawCoach-Shared-Docs`, im iOS-Repo eingebunden als
Submodule unter `docs/shared/`.

Damit gilt:

- gemeinsame Specs nur in `docs/shared/` aendern
- iOS-spezifische Dokumente weiterhin direkt in `docs/`
- erledigte cross-platform Plaene nicht lokal weiterkopieren

---

## Relevante Dateien

```text
PawCoach-iOS/
├── docs/
│   ├── shared/
│   │   ├── CROSS-PLATFORM-FEATURE-SPEC.md
│   │   ├── CROSS-PLATFORM-SYNC-STRATEGY.md
│   │   ├── IOS-DEVELOPER-GUIDE.md
│   │   ├── openapi.json
│   │   ├── reports/
│   │   └── plans/
│   ├── app-guide.md
│   ├── backend-api.md
│   └── ...
```

---

## Setup

Nach einem frischen Clone oder nach Pulls mit Submodule-Aenderungen:

```bash
git submodule update --init --recursive
```

Wenn du gezielt den neuesten Shared-Docs-Stand ziehen willst:

```bash
git submodule update --remote docs/shared
```

---

## Arbeitsweise

### Shared Docs aendern

Wenn du einen Contract, eine gemeinsame Spezifikation oder einen cross-platform Report
anpassen musst:

```bash
cd docs/shared
git checkout main
# Dateien bearbeiten
git add -A
git commit -m "docs: update shared contract"
git push origin main
cd ../..
git add docs/shared
git commit -m "docs: bump shared docs"
```

### iOS-Implementierung starten

Vor jedem groesseren Feature:

1. `docs/shared/CROSS-PLATFORM-FEATURE-SPEC.md` lesen
2. `docs/shared/openapi.json` gegen den geplanten Endpoint pruefen
3. `docs/shared/reports/` auf aktuellen Architekturstand pruefen
4. `docs/shared/plans/` nur dann oeffnen, wenn noch aktive Arbeit offen ist

---

## Architektur-Regeln fuer iOS

### 1. Capability-driven UI

Tabs, Menues, Buttons und Navigationsziele werden aus dem Capability-Contract
abgeleitet, nicht nur aus Rollenstrings.

Konkret:

- `CapabilityStore` ist die zentrale Freigabequelle
- fehlende oder inaktive Features werden nicht einfach sichtbar gelassen
- lokale Rollenstrings sind nur Fallback, nicht die Produktlogik
- wichtige Capability-Keys fuer die App sind insbesondere:
  - `admin_dashboard`
  - `forum_moderation`
  - `trainer_calendar`
  - `messaging`
  - `community_forum`
  - Limits wie `max_locations`

### 2. Realtime ohne Workarounds

Messaging folgt dieser Trennung:

- REST fuer Laden und Mutationen
- Socket.IO fuer Realtime-Events
- Push fuer Offline-Benachrichtigung und Deep Link

Wichtige Live-Events:

- `new_message`
- `message_receipts_updated`
- `presence_update`
- `reaction_update`
- `poll_update`

Statische Standort-Nachrichten muessen davon getrennt bleiben:

- `/api/messages/{id}/location` ist fuer feste Treffpunkte und wechselnde Orte gedacht
- iOS soll dafuer zwei sichtbare UX-Pfade haben:
  - aktuellen Standort senden
  - Treffpunkt waehlen / Adresse suchen
- Der Client darf also nicht nur den aktuellen GPS-Standort des sendenden Users anbieten

### 3. Models muessen Contract-getrieben sein

Wenn OpenAPI oder Shared-Docs neue Felder definieren, muessen iOS-Models diese bewusst
abbilden. Kritisch sind derzeit insbesondere:

- `conversation_id`
- `client_message_id`
- `delivery_status`
- `delivered_at`
- `read_at`
- `location_id`
- `location_object`
- `location_address`

### 4. Standorte sind jetzt ein echtes Fachmodell

- Admin-APIs fuer Standorte existieren serverseitig
- Kurse und Sessions koennen einen Standort per `location_id` referenzieren
- App-Models, Endpoints und Admin-Flows fuer Standorte sind angebunden
- das bedeutet konkret:
  - `AdminLocation` und `AdminLocationsResponse` existieren bereits
  - `AdminCourse` und `AdminSession` enthalten bereits `location_id` und `location_object`
  - die Admin-Networking-Layer enthalten bereits `adminLocations`, `createLocation`,
    `updateLocation` und `deleteLocation`
- der End-to-End-Flow ist jetzt angeschlossen:
  - `AdminViewModel` laedt, erstellt, aktualisiert und loescht Standorte
  - School-Settings enthalten eine dedizierte Standort-CRUD-Oberflaeche
  - Kurs-, Session- und Makeup-Editoren nutzen `location_id` als primaeren Pfad und
    halten `location` nur noch als Freitext-Fallback vor
  - die Einzelstunden-Bestaetigung unterstuetzt ebenfalls `location_id`

### 5. Letzte Recheck-Punkte sind ebenfalls umgesetzt

- `max_locations` wird im Standort-UI proaktiv geprueft
- `AdminViewModel` und `MainTabView` nutzen `admin_dashboard`
- Forum-Moderation nutzt `forum_moderation`
- Chat-Views unterstuetzen jetzt neben dem aktuellen Standort auch freie Treffpunkte
  per Adress- oder POI-Suche

### 6. Was der iOS-Entwickler rechecken soll

Vor dem naechsten Abgleich bitte gezielt pruefen:

1. Bleiben `location_id` und `location_object` in allen relevanten Admin-Flows die
   primaeren Felder?
2. Funktionieren Standort-CRUD in School-Settings sowie die Auswahl in Kurs-, Session-
   und Makeup-Editoren weiterhin gegen denselben Contract?
3. Greifen `admin_dashboard`, `forum_moderation` und `max_locations` an den
   sichtbaren UI-Gates korrekt?
4. Wird `location: String` nur noch als bewusster Freitext-Fallback genutzt?

---

## OpenAPI nutzen

Die Datei `docs/shared/openapi.json` ist die Referenz fuer:

1. Pfade und HTTP-Methoden
2. erwartete Bodies
3. Auth-Anforderungen
4. bekannte Ressourcen und Tags

Schneller Endpoint-Check:

```bash
python3 -m json.tool docs/shared/openapi.json | grep -A 8 '"/api/auth/capabilities"'
```

---

## Stabile Referenzen

Die aktuellen stabilen Referenzen sind:

- `docs/shared/CROSS-PLATFORM-FEATURE-SPEC.md`
- `docs/shared/CROSS-PLATFORM-SYNC-STRATEGY.md`
- `docs/shared/reports/2026-03-18-platform-alignment-status-v7.md`

Der Modernisierungsplan bleibt nur fuer noch offene Arbeit relevant:

- `docs/shared/plans/2026-03-17-unified-modernization-implementation-plan.md`

---

## Checkliste vor dem Merge

- Shared-Dokumente auf aktuellem Stand
- OpenAPI gegen verwendete Endpoints geprueft
- UI-Gating gegen Capabilities statt nur gegen Rollen geprueft
- Realtime-State ohne lokale Sonderlogik getestet
- Push, Deep Link und Reconnect auf Messaging getestet
- bei abgeschlossenen Planpunkten: Verweise auf stabile Doku umgestellt

---

## Bei Problemen

**Submodule leer oder veraltet**

```bash
git submodule update --init --recursive
git submodule update --remote docs/shared
```

**Submodule-Konflikt**

```bash
cd docs/shared
git checkout main
git pull --ff-only origin main
cd ../..
git add docs/shared
git commit -m "docs: resolve shared docs submodule update"
```
