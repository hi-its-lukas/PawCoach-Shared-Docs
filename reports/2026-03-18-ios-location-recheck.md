# iOS Standort-System — Finaler Gegencheck

**Datum:** 2026-03-18 (dritter Durchlauf — final)
**Durchgeführt von:** iOS-Agent (Claude Code)
**Anlass:** Finaler Gegencheck nach Schliessung aller offenen Punkte
**Zielgruppe:** Head of Development
**Methode:** Quelldateien zeilenweise gelesen und gegen OpenAPI + Shared-Docs abgeglichen

---

## Zusammenfassung

Der iOS-Code wurde in drei Durchlaeufen gegen die Shared-Docs geprueft. Der finale
Durchlauf bestaetigt: Alle im vorherigen Report als Folgeaufgaben markierten Punkte
(`admin_dashboard`, `forum_moderation`, `max_locations`) sind umgesetzt und verifiziert.
Zusatzlich sendet die Einzelstunden-Bestaetigung jetzt optional `location_id`.

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

| Pruefpunkt | Status | Datei / Zeile |
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

## 2. Capability-driven UI-Gating (neu verifiziert)

### `admin_dashboard`

| Stelle | Code | Status |
|--------|------|:------:|
| `AdminViewModel.swift:73-78` | `caps.isFeatureEnabled("admin_dashboard")` mit Keychain-Fallback | ✅ |
| `MainTabView.swift:115-116` | `caps.isFeatureEnabled("admin_dashboard") && !user.platformAdmin` | ✅ |
| `MoreMenuView.swift:36` | `capabilityStore.isFeatureEnabled("admin_dashboard")` | ✅ |

### `forum_moderation`

| Stelle | Code | Status |
|--------|------|:------:|
| `ForumViewModel.swift:18-21` | `caps.isFeatureEnabled("forum_moderation")` steuert `canModerate` | ✅ |
| `ThreadDetailView.swift:9-10` | Delegiert an `viewModel.canModerate` | ✅ |
| `ThreadDetailView.swift:51,60,97` | Pin, Lock, Delete nur bei `canModerate` sichtbar | ✅ |

### `max_locations`

| Stelle | Code | Status |
|--------|------|:------:|
| `SchoolSettingsView.swift:7` | `@State private var capabilityStore = CapabilityStore.shared` | ✅ |
| `SchoolSettingsView.swift:348-349` | `capabilityStore.limit(for: "max_locations")` | ✅ |
| `SchoolSettingsView.swift:352-354` | `canCreateLocation` prueft `count < maxLocations` | ✅ |
| `SchoolSettingsView.swift:333-337` | Hinweistext bei erreichtem Limit | ✅ |
| `SchoolSettingsView.swift:344` | `.disabled(!canCreateLocation)` auf Button | ✅ |

---

## 3. Flow-Konsistenz (Settings, Kurse, Sessions, Makeup)

### Identisches Pattern in allen drei Editoren

| Verhalten | CourseEditView | AdminSessionEditView | MakeupSessionSheet |
|-----------|:-:|:-:|:-:|
| Picker `selectedLocationId` als Primaerpfad | ✅ `:216-221` | ✅ `:87-92` | ✅ `:230-235` |
| Fallback-TextField nur wenn `selectedLocationId == nil` | ✅ `:223-224` | ✅ `:93-94` | ✅ `:236-237` |
| Read-only Label wenn Standort gewaehlt | ✅ `:225-230` | ✅ `:95-100` | ✅ `:238-243` |
| onChange: `locationType` auto-populate aus Standort | ✅ `:241-247` | ✅ `:116-122` | ✅ `:254-260` |
| Save: `location` nur gesendet wenn kein `locationId` | ✅ `:328` | ✅ `:168` | ✅ `:287` |
| Save: `locationId` wird immer gesendet | ✅ `:329` | ✅ `:169` | ✅ `:288` |
| Locations bei Bedarf lazy geladen | ✅ `:105` | ✅ `:69` | ✅ `:276` |

### SchoolSettingsView: Standort-CRUD

| Feature | Zeile | Status |
|---------|-------|:------:|
| Standort-Liste mit Name, Standard-Badge, Inaktiv-Badge | `:287-329` | ✅ |
| Bearbeiten-Button → Sheet mit `editingLocation` | `:317-319` → `:248-250` | ✅ |
| Loeschen-Button → Bestaetigungsdialog mit Warnung | `:321-323` → `:251-275` | ✅ |
| Hinzufuegen-Button → Create-Sheet (limit-aware) | `:332-344` | ✅ |
| LocationEditorSheet: Name, Adresse, PLZ, Stadt, Typ, Notizen, Standard, Aktiv | `:391-506` | ✅ |
| Save ruft `createLocation()` oder `updateLocation()` | `:496-500` | ✅ |
| Standorte und Settings werden initial geladen | `:240-241` | ✅ |
| ContentUnavailableView bei leerer Standortliste | `:281-285` | ✅ |

---

## 4. Alte Workarounds / Sonderpfade

| Pruefpunkt | Ergebnis | Evidenz |
|-----------|:--------:|---------|
| Polling statt Socket.IO | ✅ Keines | WebSocketManager event-driven |
| Versteckte Alternate-Endpoints | ✅ Keine | Alle Location-Endpoints identisch mit OpenAPI |
| `location` String als Primaerpfad | ✅ Nirgends | Alle Editoren: `selectedLocationId` primaer |
| Clientseitige Standort-Sonderlogik | ✅ Keine | Kein client-only Handling |
| Role-only Feature-Gating | ✅ Behoben | `admin_dashboard`, `forum_moderation` jetzt capability-driven |
| Role-only Limit-Gating | ✅ Behoben | `max_locations` jetzt capability-driven |

Verbleibende Rollenstring-Checks sind ausschliesslich **Datenfilterung** (welche User
sind Trainer/Kunden in Picker-Listen) — das ist semantisch korrekt und kein Feature-Gating:

| Datei | Zeile | Zweck |
|-------|-------|-------|
| `CourseEditView.swift` | `:50` | Trainer-Picker: `.role == "trainer"` |
| `AdminSessionCreateView.swift` | `:48` | Trainer-Picker: `.role == "trainer"` |
| `AdminAvailabilityView.swift` | `:115` | Trainer-Filter: `.role == "trainer" \|\| .role == "admin"` |
| `AdminDirectBookingView.swift` | `:15` | Kunden-Picker: `.role == "customer"` |
| `AdminDogEditView.swift` | `:289` | Kunden-Picker: `.role == "customer"` |

---

## 5. Nachgezogener Confirm-Fix

### `confirmEinzelstunde` sendet jetzt optional `location_id`

| Aspekt | Detail |
|--------|--------|
| **Datei** | `AdminViewModel+Bookings.swift` |
| **Code** | `confirmEinzelstunde(id:locationId:)` baut jetzt einen JSON-Body mit optionalem `location_id` |
| **UI** | `AdminRequestsView.swift` / `EinzelstundeRow` zeigt jetzt einen optionalen Standort-Picker vor der Bestaetigung |
| **Verhalten** | Ohne Auswahl greift weiter sauber der Backend-Fallback auf den Kurs-Standort |
| **Status** | ✅ Behoben |

---

## 6. Nebenbefund (bereinigt)

Der tote Fallback-Pfad in `ForumViewModel.canModerate` wurde entfernt. Ohne geladene
Capabilities bleibt das Verhalten jetzt explizit fail-closed.

---

## Fazit

| Kernfrage | Ergebnis |
|-----------|:--------:|
| `admin_dashboard`, `forum_moderation`, `max_locations` greifen korrekt | ✅ |
| Standort-Flow konsistent in Settings, Kursen, Sessions und Makeup | ✅ |
| `location: String` nur Fallback | ✅ |
| Keine Drift zwischen App-Code, Shared Docs und OpenAPI | ✅ |
| Keine alten Workarounds oder role-only Sonderpfade mehr aktiv | ✅ |

Das Standort-System und das Capability-Gating sind end-to-end korrekt integriert.
Auch die zuvor letzte Restluecke bei der Einzelstunden-Bestaetigung ist jetzt
geschlossen.
