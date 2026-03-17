# Cross-Platform Sync-Strategie

**Stand:** 2026-03-17
**Ziel:** Backend, Web-Frontend und iOS entwickeln gegen dieselben fachlichen Regeln,
denselben API-Contract und dieselben Entitlements.

---

## 1. Grundsatz

PawCoach hat drei Ausfuehrungsoberflaechen nach dem Login:

- Backend/API als System of Record
- Web-Frontend als browserbasierter Client
- iOS-App als nativer Client

Die Produktlogik darf nicht auseinanderlaufen. Es gibt deshalb genau eine gemeinsame
Dokumentationsquelle: `PawCoach-Shared-Docs`.

---

## 2. Single Source of Truth

### 2.1 Kanonische Artefakte

| Artefakt | Kanonischer Ort | Konsumenten |
|----------|-----------------|-------------|
| Feature-Spec | `PawCoach-Shared-Docs/CROSS-PLATFORM-FEATURE-SPEC.md` | Backend, Web, iOS |
| Sync-Strategie | `PawCoach-Shared-Docs/CROSS-PLATFORM-SYNC-STRATEGY.md` | Backend, Web, iOS |
| OpenAPI | `PawCoach-Shared-Docs/openapi.json` | Web, iOS, QA |
| Cross-Platform-Plaene | `PawCoach-Shared-Docs/plans/` | Backend, Web, iOS |

### 2.2 Projekt-Repos

Die Produkt-Repositories enthalten nur das Submodule:

- `Dog-School-Manager/docs/shared/`
- `PawCoach-iOS/docs/shared/`

Repo-lokale Dokumente bleiben erlaubt, aber nur fuer plattformspezifische Themen
wie iOS-interne Audits, Betriebsdokumente oder rein webseitige UI-Notizen.

---

## 3. Contract-first Regeln

### 3.1 Backend ist die fachliche Autoritaet

- Persistenz, Rechtepruefung und Entitlements werden im Backend entschieden.
- Web und iOS implementieren gegen dokumentierte Contracts.
- Kein Client fuehrt eine neue API-Annahme ein, die nicht vorher im Backend festgelegt wurde.

### 3.2 Capability-Contract ist Pflicht

Rollen allein reichen nicht. Die Freigabe eines Features ergibt sich aus:

- aktivem Kontext
- Rolle des Users
- aktivem Plan der Schule
- gebuchten Addons
- optionalen Rolleneinschraenkungen innerhalb einer Schule

Darum wird ein kanonischer Endpoint eingefuehrt, z.B. `GET /api/features` oder
`GET /api/auth/capabilities`. Dieser Contract ist fuer Web und iOS bindend.

### 3.3 OpenAPI ist Referenz fuer API-Pfade

`openapi.json` wird aus dem Backend generiert und in dieses Shared-Repo geschrieben.
Neue oder geaenderte API-Pfade gelten erst dann als stabil, wenn die OpenAPI-Spec
aktualisiert wurde.

---

## 4. Entwicklungsprozess

### 4.1 Neues oder geaendertes Feature

1. Problem oder Luecke in `plans/` dokumentieren.
2. Zielbild und API-Contract festlegen.
3. Entitlements und Sichtbarkeitsregeln festlegen.
4. Backend implementieren und testen.
5. OpenAPI und Shared-Spec aktualisieren.
6. Web und iOS gegen denselben Contract umsetzen.
7. Paritaet pruefen und Delta schliessen.

### 4.2 Breaking Changes

Breaking Changes muessen vermieden werden. Wenn sie noetig sind:

1. alte und neue Semantik dokumentieren
2. Migration fuer Web und iOS planen
3. Deprecation-Zeitraum festlegen
4. Abschaltung erst nach erfolgreicher Client-Migration

### 4.3 Keine Workaround-Regel

Nicht erlaubt als dauerhafte Loesung:

- polling statt verfuegbarer Echtzeit-Events
- UI einblenden und auf `403` hoffen
- Rollenstrings hart codieren, wenn Plan/Addons relevant sind
- versteckte Alternate-Endpoints nur fuer einen Client
- clientseitige fachliche Sonderregeln, die im Backend nicht kanonisch modelliert sind

---

## 5. Technische Leitlinien

### 5.1 Entitlements

- Backend liefert einen normalisierten Capability-Contract.
- Web und iOS bauen Navigation, Menues, Deep Links und Aktionen daraus.
- Nicht verfuegbare Features werden ausgeblendet oder als gesperrt markiert.
- Backend validiert weiterhin serverseitig jeden Zugriff.

### 5.2 Realtime

- REST ist Standard fuer Laden, Erstellen und Mutationen.
- Socket.IO ist Standard fuer in-app Realtime-Events.
- Push ist fuer Offline-Benachrichtigung und Deep Linking, nicht fuer den Live-State.

### 5.3 Testing

- Backend: API-Contract-Tests und Rechte-Tests
- Web: UI-Sichtbarkeit und happy-path Flows
- iOS: Capability-Gating, Navigation und Realtime-State
- Cross-platform: End-to-end Tests fuer kritische Kernablaeufe

---

## 6. Release-Checkliste

- Shared-Dokumente im Shared-Repo aktualisiert
- OpenAPI neu generiert
- Capability-Contract dokumentiert
- Entitlements fuer Plan und Addons getestet
- Web-Sichtbarkeit gegen Capabilities getestet
- iOS-Sichtbarkeit gegen Capabilities getestet
- Realtime-Flows ohne Polling oder Sonderwege getestet
- Release Notes fuer API- und UI-Aenderungen geschrieben

---

## 7. Prioritaeten

| # | Massnahme | Wirkung | Prioritaet |
|---|-----------|---------|------------|
| 1 | Kanonischen Capability-Contract einfuehren | Sehr hoch | Sofort |
| 2 | OpenAPI im Shared-Repo verankern | Hoch | Sofort |
| 3 | Chat-Realtime auf Events statt Workarounds umstellen | Hoch | Sofort |
| 4 | Web und iOS capability-driven machen | Sehr hoch | Naechster Sprint |
| 5 | Contract-Tests und Sichtbarkeits-Tests ausbauen | Hoch | Naechster Sprint |
| 6 | Automatisierten Delta-Check einfuehren | Mittel | Danach |

---

## 8. Referenzplan

Die konkrete, gemeinsame Umsetzung steht in:

- `plans/2026-03-17-unified-modernization-implementation-plan.md`
| Standort als Karten-Vorschau | 2026-03-16 (iOS) |

---

## 9. iOS Modernization Status (2026-03-17)

### Abgeschlossen

| Workstream | Was | Status |
|-----------|-----|--------|
| E1 | CapabilityStore + Models | Done (Build 29) |
| E1 | Capabilities in AppState (Login/Switch/Logout) | Done (Build 29) |
| E2 | MainTabView capability-driven mit Fallback | Done (Build 29) |
| F2 | ChatStore (ersetzt MessagingViewModel) | Done (Build 29) |
| F3 | WebSocketManager Delegate-Pattern | Done (Build 29) |
| F3 | Neue Events: message_receipts_updated, presence_update | Done (Build 29) |
| F1 | clientMessageId im Message-Model | Done (Build 29, Backend noch nicht) |
| F3 | MessageDetailView + MessagesListView auf ChatStore | Done (Build 29) |
| F3 | Polling komplett entfernt | Done (Build 29) |

### Offen (Backend-Voraussetzungen)

| Was | Endpoint | Status |
|-----|----------|--------|
| Capability-Endpoint | GET /api/auth/capabilities | Backend muss implementieren |
| clientMessageId akzeptieren | POST /api/messages/{id} | Backend muss Feld akzeptieren |
| message_receipts_updated Event | Socket.IO | Backend muss bei delivered/read emittieren |
| presence_update Event | Socket.IO | Backend muss bei connect/disconnect emittieren |
