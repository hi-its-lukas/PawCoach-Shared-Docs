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

### 4. Standorte sind jetzt ein echtes Fachmodell

- Admin-APIs fuer Standorte existieren serverseitig
- Kurse und Sessions koennen einen Standort per `location_id` referenzieren
- App-Models und Endpoints fuer Standorte sind vorbereitet
- eine dedizierte iOS-Verwaltungsoberflaeche fuer Standort-CRUD ist kein stabiler
  Endanwender-Flow und muss bei weiterer Arbeit explizit mitgedacht werden

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
