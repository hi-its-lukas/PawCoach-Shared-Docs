# iOS Standort-System — Gegencheck nach Integration

**Datum:** 2026-03-18
**Durchgeführt von:** iOS-Agent (Claude Code)
**Anlass:** Abschluss der End-to-End-Integration des Standort-Systems in Backend, Web-Admin und iOS-Admin
**Zielgruppe:** Head of Development

---

## Zusammenfassung

Der iOS-Code wurde systematisch gegen die Shared-Docs geprüft:
- `CROSS-PLATFORM-FEATURE-SPEC.md` (v7)
- `CROSS-PLATFORM-SYNC-STRATEGY.md`
- `IOS-DEVELOPER-GUIDE.md`
- `BACKEND-DEVELOPER-GUIDE.md`
- `openapi.json`

**Ergebnis: Alle drei Kernfragen sind positiv beantwortet.**

---

## 1. Contract-Alignment (Models & Endpoints)

| Prüfpunkt | Status | Datei / Zeile |
|-----------|--------|---------------|
| `AdminLocation` Model | ✅ | `AdminModels.swift:100-123` |
| `AdminLocationsResponse` | ✅ | `AdminModels.swift:7` |
| `AdminCourse.locationId` + `.locationObject` | ✅ | `AdminModels.swift:37-38` |
| `AdminSession.locationId` + `.locationObject` | ✅ | `AdminModels.swift:76-77` |
| `AdminCourseForm.locationId` | ✅ | `AdminModels.swift:199` |
| `AdminSessionForm.locationId` | ✅ | `AdminModels.swift:286` |
| `MakeupSessionForm.locationId` | ✅ | `AdminModels.swift:306` |
| `AdminLocationForm` (CRUD-Form) | ✅ | `AdminModels.swift:264-281` |
| `GET /api/admin/locations` | ✅ | `Endpoint+Admin.swift:96-97` |
| `POST /api/admin/locations` | ✅ | `Endpoint+Admin.swift:100-101` |
| `PUT /api/admin/locations/{id}` | ✅ | `Endpoint+Admin.swift:104-105` |
| `DELETE /api/admin/locations/{id}` | ✅ | `Endpoint+Admin.swift:108-109` |

**OpenAPI-Alignment:** Alle 4 Location-Endpoints stimmen exakt mit `openapi.json` überein.

---

## 2. Flow-Konsistenz (Settings, Kurse, Sessions, Makeup)

| View | `location_id` primär? | `location` nur Fallback? | Datei |
|------|:---------------------:|:------------------------:|-------|
| SchoolSettingsView (CRUD) | ✅ | ✅ | `SchoolSettingsView.swift:278-506` |
| CourseEditView (Picker) | ✅ | ✅ nur wenn kein ID gewählt | `CourseEditView.swift:214-248` |
| AdminSessionEditView (Picker) | ✅ | ✅ nur wenn kein ID gewählt | `AdminSessionEditView.swift:87-122` |
| MakeupSessionSheet (Picker) | ✅ | ✅ nur wenn kein ID gewählt | `AdminSessionEditView.swift:199-300` |
| AdminViewModel (Logik) | ✅ | — | `AdminViewModel+Settings.swift:35-92` |

**Pattern:** Alle Editoren verwenden einen Picker für `selectedLocationId` als Primärpfad.
Das `location`-Textfeld erscheint nur als Fallback, wenn kein gespeicherter Standort gewählt ist.
Beim Speichern wird `location` nur gesendet wenn `selectedLocationId == nil`.

---

## 3. Alte Workarounds / Sonderpfade

| Prüfpunkt | Status | Bemerkung |
|-----------|--------|-----------|
| Polling statt Socket.IO | ✅ Keines | Durchgehend event-driven via WebSocketManager |
| Versteckte Alternate-Endpoints | ✅ Keine | Alle Endpoints matchen OpenAPI |
| `location` String als Primärpfad | ✅ Nirgends | Immer `locationId` primär |
| Clientseitige Sonderlogik | ✅ Keine | Kein client-only Standort-Handling |

---

## 4. Offene Punkte (kein Blocker)

### a) `max_locations` UI-Gating fehlt

- **Stelle:** `SchoolSettingsView.swift:332-336` — Button "Standort hinzufügen" ist immer aktiv
- **Infrastruktur vorhanden:** `CapabilityStore.limit(for: "max_locations")` existiert
- **Backend-Absicherung:** Serverseitig wird korrekt mit 403 geblockt
- **Empfehlung:** Proaktives UI-Gating ergänzen (Einzeiler), um bessere UX zu bieten

### b) Rollenstring-Checks in nicht-standortbezogenen Admin-Views

Folgende Stellen nutzen Rollenstrings statt Capabilities — betreffen aber nicht den Standort-Flow:

| Datei | Zeile | Pattern |
|-------|-------|---------|
| `AdminViewModel.swift` | 71-75 | `KeychainHelper.read("activeRole") == "admin"` |
| `MainTabView.swift` | 116 | `caps.currentRole == "admin"` für admin_dashboard |
| `CourseEditView.swift` | 50 | `.role == "trainer"` (Datenfilterung) |
| `AdminSessionCreateView.swift` | 48 | `.role == "trainer"` (Datenfilterung) |
| `AdminAvailabilityView.swift` | 115 | `.role == "trainer" \|\| .role == "admin"` |
| `ForumViewModel.swift` | 23 | `role == "admin" \|\| role == "trainer"` |

**Einordnung:** Die Trainer/Customer-Filterung in Listen ist **Datenfilterung** (welche User haben
welche Rolle), nicht Feature-Gating — das ist vertretbar. Das `isAdmin`-Guard und die
`admin_dashboard`-Prüfung sollten langfristig über einen Capability-Key laufen.

---

## Fazit

| Kernfrage | Ergebnis |
|-----------|----------|
| Frontend, Backend und App verhalten sich identisch gegen denselben Contract | ✅ Ja |
| Standort-Flow funktioniert konsistent in Settings, Kursen und Sessions | ✅ Ja |
| Keine alten Workarounds oder role-only Sonderpfade im Standort-Bereich aktiv | ✅ Ja |

Das Standort-System ist end-to-end korrekt integriert. Die offenen Punkte
(`max_locations`-UI-Gating, Capability-Migration für nicht-standortbezogene Checks) sind
keine Blocker und können als eigenständige Folgeaufgaben eingeplant werden.
