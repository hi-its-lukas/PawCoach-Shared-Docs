# Cross-Platform Sync-Strategie

**Stand:** 2026-03-18
**Ziel:** Backend, Web-Frontend und iOS entwickeln gegen dieselben fachlichen Regeln,
denselben API-Contract und dieselben Entitlements.

---

## 1. Grundsatz

PawCoach hat drei relevante Laufzeitoberflaechen:

- Backend/API als System of Record
- Web-Frontend als browserbasierter Client
- iOS-App als nativer Client

Die Produktlogik darf nicht auseinanderlaufen. Deshalb gibt es genau eine gemeinsame
Dokumentationsquelle: `PawCoach-Shared-Docs`.

---

## 2. Single Source of Truth

### 2.1 Kanonische Artefakte

| Artefakt | Kanonischer Ort | Konsumenten |
|----------|-----------------|-------------|
| Feature-Spec | `PawCoach-Shared-Docs/CROSS-PLATFORM-FEATURE-SPEC.md` | Backend, Web, iOS |
| Sync-Strategie | `PawCoach-Shared-Docs/CROSS-PLATFORM-SYNC-STRATEGY.md` | Backend, Web, iOS |
| Backend Guide | `PawCoach-Shared-Docs/BACKEND-DEVELOPER-GUIDE.md` | Backend |
| Web Guide | `PawCoach-Shared-Docs/WEB-DEVELOPER-GUIDE.md` | Web |
| iOS Guide | `PawCoach-Shared-Docs/IOS-DEVELOPER-GUIDE.md` | iOS |
| OpenAPI | `PawCoach-Shared-Docs/openapi.json` | Web, iOS, QA |
| Reports | `PawCoach-Shared-Docs/reports/` | Backend, Web, iOS |
| Aktive Plaene | `PawCoach-Shared-Docs/plans/` | Backend, Web, iOS |

### 2.2 Projekt-Repositories

Die Produkt-Repositories enthalten nur das Submodule:

- `Dog-School-Manager/docs/shared/`
- `PawCoach-iOS/docs/shared/`

Repo-lokale Doku bleibt erlaubt, aber nur fuer plattformspezifische Arbeitsanleitungen,
Betriebsthemen oder lokale Einstiegspunkte in die kanonischen Shared Docs.

---

## 3. Contract-first Regeln

### 3.1 Backend ist die fachliche Autoritaet

- Persistenz, Rechtepruefung, Pricing und Entitlements werden im Backend entschieden.
- Web und iOS implementieren gegen dokumentierte Contracts.
- Kein Client fuehrt eine neue API-Annahme ein, die nicht vorher im Backend festgelegt wurde.

### 3.2 Capability-Contract ist Pflicht

Rollen allein reichen nicht. Die Freigabe eines Features ergibt sich aus:

- aktivem Kontext
- Rolle des Users
- aktivem Plan der Schule
- gebuchten oder inkludierten Addons
- Limits wie `max_trainers`, `max_courses`, `max_locations`

Der verbindliche Contract dafuer ist `GET /api/auth/capabilities`.

### 3.3 OpenAPI ist Referenz fuer REST-Pfade

`openapi.json` wird aus dem Backend generiert und in dieses Shared-Repo geschrieben.
Neue oder geaenderte REST-Pfade gelten erst dann als stabil, wenn die OpenAPI-Spec
aktualisiert wurde.

### 3.4 Pricing-Katalog ist Pflicht

Landingpage, Registrierung, Billing und Capability-Contract muessen auf demselben
Produktkatalog beruhen. Manuell gepflegte Parallelquellen fuer Plaene, Preise oder
Addons sind nicht erlaubt.

---

## 4. Entwicklungsprozess

### 4.1 Neues oder geaendertes Feature

1. Problem oder Luecke in `plans/` dokumentieren, wenn noch keine stabile Spezifikation existiert.
2. Zielbild, API-Contract und Entitlements festlegen.
3. Backend implementieren und testen.
4. OpenAPI und Shared-Spec aktualisieren.
5. Web und iOS gegen denselben Contract umsetzen.
6. Paritaet pruefen und Delta schliessen.
7. Umgesetzte Plaene aus `plans/` entfernen.

### 4.2 Breaking Changes

Breaking Changes muessen vermieden werden. Wenn sie noetig sind:

1. alte und neue Semantik dokumentieren
2. Migration fuer Web und iOS planen
3. Deprecation-Zeitraum festlegen
4. Abschaltung erst nach erfolgreicher Client-Migration

### 4.3 Keine Workaround-Regel

Nicht erlaubt als dauerhafte Loesung:

- Polling statt verfuegbarer Echtzeit-Events
- UI einblenden und auf `403` hoffen
- Rollenstrings hart codieren, wenn Plan, Addons oder Limits relevant sind
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

- REST ist Standard fuer Laden und Mutationen.
- Socket.IO ist Standard fuer in-app Realtime-Events.
- Push ist fuer Offline-Benachrichtigung und Deep Linking, nicht fuer den Live-State.
- Messaging nutzt insbesondere `new_message`, `message_receipts_updated` und `presence_update`.

### 5.3 Standorte

- Mehr-Standort-Betrieb ist ein fachliches Kernkonzept und kein Freitext-Workaround.
- Standortdaten werden ueber `SchoolLocation` plus `location_id` auf Kursen und Sessions modelliert.
- Standortverfuegbarkeit richtet sich nach dem Capability-Contract und `max_locations`.

### 5.4 Testing

- Backend: API-Contract-Tests und Rechte-Tests
- Web: UI-Sichtbarkeit und happy-path Flows
- iOS: Capability-Gating, Navigation und Realtime-State
- Cross-platform: End-to-end Tests fuer kritische Kernablaeufe

---

## 6. Release-Checkliste

- Shared-Dokumente im Shared-Repo aktualisiert
- OpenAPI neu generiert
- Capability-Contract dokumentiert
- Pricing-Katalog und Public-Pricing geprueft
- Entitlements fuer Plan, Addons und Limits getestet
- Web-Sichtbarkeit gegen Capabilities getestet
- iOS-Sichtbarkeit gegen Capabilities getestet
- Realtime-Flows ohne Polling oder Sonderwege getestet
- Umgesetzte Plaene aus `plans/` entfernt

---

## 7. Stabile Referenzen

Die aktuelle stabile Referenz fuer den Architekturstand ist:

- `CROSS-PLATFORM-FEATURE-SPEC.md`
- `BACKEND-DEVELOPER-GUIDE.md`
- `WEB-DEVELOPER-GUIDE.md`
- `IOS-DEVELOPER-GUIDE.md`
- `reports/2026-03-18-platform-alignment-status-v7.md`

Der Plan `plans/2026-03-17-unified-modernization-implementation-plan.md` bleibt nur
solange bestehen, wie daraus noch offene Arbeit abgeleitet wird.
