# iOS-Entwickler Anleitung — Shared Docs & API Contract

**Stand:** 2026-03-17
**Betrifft:** Alle, die am PawCoach-iOS-Repo arbeiten

---

## Grundregel

Die geteilte Produkt- und API-Dokumentation wird nicht mehr im iOS-Repo dupliziert.
Sie lebt im Repository `PawCoach-Shared-Docs` und wird im iOS-Repo als Submodule unter
`docs/shared/` eingebunden.

Damit gilt:

- gemeinsame Specs nur in `docs/shared/` aendern
- iOS-spezifische Dokumente weiterhin direkt in `docs/`
- keine lokalen Kopien gemeinsamer Plaene oder Specs mehr anlegen

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
│   │   └── plans/
│   ├── CODEX-REVIEW-PROMPT.md
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

Wenn du einen Contract, eine gemeinsame Spezifikation oder einen cross-platform Plan
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
3. `docs/shared/plans/` auf bestehende Design- oder Fix-Dokumente pruefen
4. erst dann Swift-Implementierung starten

---

## Architektur-Regeln fuer iOS

### 1. Capability-driven UI

Tabs, Menues, Buttons und Navigationsziele werden nicht nur aus Rollenstrings abgeleitet.
Die App muss sich am gemeinsamen Capability-Contract orientieren:

- aktiver Benutzerkontext
- Rolle
- gebuchter Plan der Schule
- aktive Addons
- effektive Feature-Freigaben

Das heisst konkret:

- kein statisches Role-only Gating fuer Admin- oder Forum-Features
- keine versteckten API-Probieraufrufe, um Rechte indirekt zu erraten
- keine Screens fuer Features anzeigen, die im aktuellen Kontext nicht verfuegbar sind

### 2. Realtime ohne Workarounds

Messaging folgt dieser Trennung:

- REST fuer Laden und Mutationen
- Socket.IO fuer Realtime-Events
- Push fuer Offline-Benachrichtigung und Deep Link

Polling-Loops sind nur ein temporarer Fallback und muessen entfernt werden, sobald der
gemeinsame Realtime-Contract steht.

### 3. Models muessen Contract-getrieben sein

Wenn OpenAPI oder Shared-Plan neue Felder definieren, muessen iOS-Models diese bewusst
abbilden. Kritisch fuer Messaging sind insbesondere:

- `conversation_id`
- `client_message_id`
- `delivery_status`
- `delivered_at`
- `read_at`

---

## OpenAPI nutzen

Die Datei `docs/shared/openapi.json` ist die Referenz fuer:

1. Pfade und HTTP-Methoden
2. erwartete Bodies
3. Auth-Anforderungen
4. bekannte Ressourcen und Tags

Schneller Endpoint-Check:

```bash
python3 -m json.tool docs/shared/openapi.json | grep -A 8 '"/api/messages/info/{message_id}"'
```

---

## Aktuelle Referenzplaene

Wichtige gemeinsame Dokumente:

- `docs/shared/plans/2026-03-16-ios-backend-sync-fixes.md`
- `docs/shared/plans/2026-03-17-unified-modernization-implementation-plan.md`

Der zweite Plan ist die aktuelle Referenz fuer:

- Capability-Contract
- Messaging-Modernisierung
- Web/iOS-Paritaet nach Login
- Entitlement-gesteuerte Sichtbarkeit

---

## Checkliste vor dem Merge

- Shared-Dokumente auf aktuellem Stand
- OpenAPI gegen verwendete Endpoints geprueft
- UI-Gating gegen Capabilities statt nur gegen Rollen geprueft
- Realtime-State ohne lokale Sonderlogik getestet
- Push, Deep Link und Reconnect auf Messaging getestet

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
