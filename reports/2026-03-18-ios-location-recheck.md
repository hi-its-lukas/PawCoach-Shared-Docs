# iOS Standort-System — Gegencheck nach Integration

**Datum:** 2026-03-18 (zweiter Durchlauf)
**Durchgeführt von:** iOS-Agent (Claude Code)
**Anlass:** Abschluss der End-to-End-Integration des Standort-Systems in Backend, Web-Admin und iOS-Admin
**Zielgruppe:** Head of Development
**Methode:** Quelldateien zeilenweise gelesen und gegen OpenAPI + Shared-Docs abgeglichen

---

## Zusammenfassung

Der iOS-Code wurde in zwei unabhängigen Durchläufen systematisch gegen die Shared-Docs geprüft:
- `CROSS-PLATFORM-FEATURE-SPEC.md` (v7)
- `CROSS-PLATFORM-SYNC-STRATEGY.md`
- `IOS-DEVELOPER-GUIDE.md`
- `BACKEND-DEVELOPER-GUIDE.md`
- `openapi.json`

**Ergebnis: Alle drei Kernfragen sind positiv beantwortet. Beide Durchläufe kommen zum selben Ergebnis.**

---

## 1. Contract-Alignment (Models & Endpoints)

### Endpoints: iOS vs. OpenAPI

| OpenAPI-Pfad | iOS Endpoint+Admin.swift | Methode | Match |
|-------------|--------------------------|---------|:-----:|
| `/api/admin/locations` | `:96-97` `.adminLocations()` | GET | ✅ |
| `/api/admin/locations` | `:100-101` `.createLocation(body:)` | POST | ✅ |
| `/api/admin/locations/{location_id}` | `:104-105` `.updateLocation(id:body:)` | PUT | ✅ |
| `/api/admin/locations/{location_id}` | `:108-109` `.deleteLocation(id:)` | DELETE | ✅ |

### Models: CodingKeys vs. Backend snake_case

| Prüfpunkt | Status | Datei / Zeile |
|-----------|:------:|---------------|
| `AdminLocation` (id, name, slug, address, city, postal_code, notes, location_type, lat, lng, is_default, is_active, display_label) | ✅ | `AdminModels.swift:100-123` |
| `AdminLocationsResponse` | ✅ | `AdminModels.swift:7` |
| `AdminCourse.locationId` → CK `location_id` | ✅ | `AdminModels.swift:37` → CK`:58` |
| `AdminCourse.locationObject` → CK `location_object` | ✅ | `AdminModels.swift:38` → CK`:59` |
| `AdminSession.locationId` → CK `location_id` | ✅ | `AdminModels.swift:76` → CK`:92` |
| `AdminSession.locationObject` → CK `location_object` | ✅ | `AdminModels.swift:77` → CK`:93` |
| `AdminCourseForm.locationId` → CK `location_id` | ✅ | `AdminModels.swift:199` → CK`:218` |
| `AdminSessionForm.locationId` → CK `location_id` | ✅ | `AdminModels.swift:286` → CK`:294` |
| `MakeupSessionForm.locationId` → CK `location_id` | ✅ | `AdminModels.swift:306` → CK`:311` |
| `AdminLocationForm` (CRUD-Form, alle CodingKeys korrekt) | ✅ | `AdminModels.swift:264-281` |
| `AdminSchoolSettings.locations: [AdminLocation]?` | ✅ | `AdminModels.swift:180` |

---

## 2. Flow-Konsistenz (Settings, Kurse, Sessions, Makeup)

### Identisches Pattern in allen drei Editoren

| Verhalten | CourseEditView | AdminSessionEditView | MakeupSessionSheet |
|-----------|:-:|:-:|:-:|
| Picker `selectedLocationId` als Primärpfad | ✅ `:216-221` | ✅ `:87-92` | ✅ `:230-235` |
| Fallback-TextField nur wenn `selectedLocationId == nil` | ✅ `:223-224` | ✅ `:93-94` | ✅ `:236-237` |
| Read-only Label wenn Standort gewählt | ✅ `:225-230` | ✅ `:95-100` | ✅ `:238-243` |
| onChange: `locationType` auto-populate aus Standort | ✅ `:241-247` | ✅ `:116-122` | ✅ `:254-260` |
| Save: `location` nur gesendet wenn kein `locationId` | ✅ `:328` | ✅ `:168` | ✅ `:287` |
| Save: `locationId` wird immer gesendet | ✅ `:329` | ✅ `:169` | ✅ `:288` |
| Locations bei Bedarf lazy geladen | ✅ `:105` | ✅ `:69` | ✅ `:276` |

### SchoolSettingsView: Standort-CRUD

| Feature | Zeile | Status |
|---------|-------|:------:|
| Standort-Liste mit Name, Standard-Badge, Inaktiv-Badge | `:287-329` | ✅ |
| Bearbeiten-Button → Sheet mit `editingLocation` | `:317-319` → `:248-250` | ✅ |
| Löschen-Button → Bestätigungsdialog mit Warnung | `:321-323` → `:251-275` | ✅ |
| Hinzufügen-Button → Create-Sheet | `:332-336` → `:245-247` | ✅ |
| LocationEditorSheet: Name, Adresse, PLZ, Stadt, Typ, Notizen, Standard, Aktiv | `:391-506` | ✅ |
| Save ruft `createLocation()` oder `updateLocation()` | `:496-500` | ✅ |
| Standorte und Settings werden initial geladen | `:240-241` | ✅ |
| ContentUnavailableView bei leerer Standortliste | `:281-285` | ✅ |

### AdminViewModel+Settings: Location-Methoden

| Methode | Zeile | Endpoint | Nachverarbeitung |
|---------|-------|----------|-----------------|
| `loadLocations()` | `:35-42` | GET `/api/admin/locations` | Array befüllen |
| `createLocation(_:)` | `:44-59` | POST `/api/admin/locations` | Append + Sort + Settings reload |
| `updateLocation(id:form:)` | `:61-80` | PUT `/api/admin/locations/{id}` | Replace + Sort + Settings reload |
| `deleteLocation(id:)` | `:82-92` | DELETE `/api/admin/locations/{id}` | Remove + Settings reload |

Settings-Reload nach Mutation stellt sicher, dass `AdminSchoolSettings.locations` konsistent bleibt.

---

## 3. Alte Workarounds / Sonderpfade

| Prüfpunkt | Ergebnis | Evidenz |
|-----------|:--------:|---------|
| Polling statt Socket.IO | ✅ Keines | WebSocketManager nutzt durchgehend event-driven Architektur |
| Versteckte Alternate-Endpoints | ✅ Keine | Alle 4 Location-Endpoints identisch mit OpenAPI |
| `location` String als Primärpfad | ✅ Nirgends | Alle Editoren: `selectedLocationId` primär, TextField nur als Fallback |
| Clientseitige Standort-Sonderlogik | ✅ Keine | Kein client-only Handling |
| Locations hart im Code statt vom Backend | ✅ Nein | Alle Daten via API geladen |

---

## 4. Offene Punkte (kein Blocker)

### a) `max_locations` UI-Gating fehlt

- **Stelle:** `SchoolSettingsView.swift:332-336` — Button "Standort hinzufügen" ist immer aktiv
- **Infrastruktur vorhanden:** `CapabilityStore.limit(for: "max_locations")` existiert (`CapabilityStore.swift:32-36`)
- **Backend-Absicherung:** Serverseitig wird korrekt mit 403 geblockt
- **Empfehlung:** Proaktives UI-Gating ergänzen — Button disablen und Hinweistext anzeigen wenn Limit erreicht. Geschätzter Aufwand: Einzeiler plus optionaler Hinweistext.

### b) Rollenstring-Checks in nicht-standortbezogenen Admin-Views

Folgende Stellen nutzen Rollenstrings statt Capabilities — betreffen aber **nicht** den Standort-Flow:

| Datei | Zeile | Pattern | Einordnung |
|-------|-------|---------|------------|
| `AdminViewModel.swift` | 72-75 | `KeychainHelper.read("activeRole") == "admin"` | Client-side Guard, kein Feature-Gating |
| `MainTabView.swift` | 116 | `caps.currentRole == "admin"` für admin_dashboard | Sollte langfristig Capability-Key nutzen |
| `CourseEditView.swift` | 50 | `.role == "trainer"` | **Datenfilterung** — semantisch korrekt |
| `AdminSessionCreateView.swift` | 48 | `.role == "trainer"` | **Datenfilterung** — semantisch korrekt |
| `AdminAvailabilityView.swift` | 115 | `.role == "trainer" \|\| .role == "admin"` | **Datenfilterung** — semantisch korrekt |
| `ForumViewModel.swift` | 23 | `role == "admin" \|\| role == "trainer"` | Feature-Gating, sollte Capability nutzen |

**Einordnung:** Die Trainer/Customer-Filterung in Listen ist **Datenfilterung** (welche User haben welche Rolle), nicht Feature-Gating — das ist korrekt und muss nicht migriert werden. `isAdmin` und `admin_dashboard` sollten langfristig über einen Capability-Key laufen, sind aber funktional kein Problem.

---

## Fazit

| Kernfrage | Ergebnis |
|-----------|:--------:|
| Frontend, Backend und App verhalten sich identisch gegen denselben Contract | ✅ Ja |
| Standort-Flow funktioniert konsistent in Settings, Kursen und Sessions | ✅ Ja |
| Keine alten Workarounds oder role-only Sonderpfade im Standort-Bereich aktiv | ✅ Ja |

Das Standort-System ist end-to-end korrekt integriert. Beide Check-Durchläufe bestätigen dasselbe Ergebnis. Die offenen Punkte (`max_locations`-UI-Gating, Capability-Migration für nicht-standortbezogene Checks) sind keine Blocker und können als eigenständige Folgeaufgaben eingeplant werden.

### Empfohlene Folgeaufgaben (Prio niedrig)

1. `max_locations`-Limit im UI prüfen bevor "Standort hinzufügen" aktiv ist
2. `AdminViewModel.isAdmin` auf Capability-Query umstellen
3. `ForumViewModel.canPostReview` auf Capability-Query umstellen
