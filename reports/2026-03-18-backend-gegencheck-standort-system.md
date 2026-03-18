# Backend-Gegencheck: Standort-System & Cross-Platform Alignment

**Datum:** 2026-03-18
**Erstellt von:** Backend-Team (Claude Code)
**Scope:** Verifizierung des Standort-Systems gegen Shared-Docs-Contracts nach Abschluss der Location-Integration

---

## 1. Ergebnis: API-Contract-Konformitaet

| Komponente | Status | Details |
|---|---|---|
| `SchoolLocation` Model + FK in Course/Session | OK | `location_id` FK, `location_rel` Relationship, `resolved_location`/`effective_location` Properties |
| `GET/POST/PUT/DELETE /api/admin/locations` | OK | Tenant-Isolation, Slug-Uniqueness, Cascade-Delete (setzt `location_id=NULL`) |
| Course-API (`/api/admin/courses`) | OK | Akzeptiert `location_id`, validiert Tenant, serialisiert `location_id` + `location_object` |
| Session-API (`/api/admin/sessions`) | OK | `location_id` in Create/Update/Makeup, Fallback-Kette Session -> Course -> Location |
| Capability-Contract | OK | `multi_location_management` als Plan-Feature, `max_locations` in Limits |
| Serializer (`helpers.py`) | OK | Gibt immer `location_id`, `location`, `location_object` zurueck |

**Fazit:** Alle JSON-API-Endpoints (iOS + moderne Web-Aufrufe) sind vollstaendig aligned mit dem Shared-Docs-Contract.

---

## 2. Login-Verhalten: Identisch ueber alle Plattformen

| Pruefpunkt | Status |
|---|---|
| `GET /api/auth/capabilities` als kanonische Quelle fuer UI-Gating | OK |
| Feature-Flags steuern Navigation (nicht nur Rolle) | OK |
| Web + iOS nutzen denselben Contract (Version 1) | OK |
| `multi_location_management` + `max_locations` korrekt im Contract | OK |

---

## 3. Verbleibende Legacy-Stellen im klassischen Web-Admin

Drei Stellen in den **Form-basierten Web-Admin-Routes** (nicht API) nutzen noch ausschliesslich den `location`-Freitext-String und setzen **kein `location_id`**:

### 3.1 `_auto_generate_sessions()` — PRIORITAET HOCH

**Datei:** `blueprints/admin/courses.py`, Zeile 41-42

```python
sess = Session(
    ...
    location=course.location,       # nur String
    location_type=course.location_type,
    # location_id=course.location_id  <-- FEHLT
)
```

**Auswirkung:** Wenn ein Admin ueber das Web-Formular einen Kurs mit Zeitplan anlegt, bekommen auto-generierte Sessions kein `location_id`. Diese Sessions erscheinen in der iOS-App ohne strukturiertes Standort-Objekt.

### 3.2 Web-Formulare Course/Session — PRIORITAET MITTEL

**Dateien:**
- `blueprints/admin/courses.py` Zeile 106 (Edit), 337 (Create), 477 (Create)
- `blueprints/admin/sessions.py` Zeile 178 (Edit)

Die Formulare setzen nur `course.location = request.form.get("location", "")`. Kein `location_id`-Dropdown vorhanden.

**Templates betroffen:** `create_course.html`, `edit_course.html`, `create_session.html`, `edit_session.html`

**Auswirkung:** Gemischte Daten wenn derselbe Admin mal Web-Formular, mal iOS nutzt. iOS setzt `location_id`, Web setzt nur String.

### 3.3 Course Duplicate — PRIORITAET NIEDRIG

**Datei:** `blueprints/admin/courses.py` Zeile 282-283

```python
Course(
    ...
    location=original.location,       # nur String
    location_type=original.location_type,
    # location_id=original.location_id  <-- FEHLT
)
```

**Auswirkung:** Duplizierte Kurse verlieren die `location_id`-Zuordnung.

---

## 4. Empfohlene Fixes

| # | Stelle | Fix | Aufwand |
|---|---|---|---|
| 1 | `_auto_generate_sessions()` | `location_id=course.location_id` ergaenzen | 1 Zeile |
| 2 | Course Duplicate | `location_id=original.location_id` ergaenzen | 1 Zeile |
| 3 | Course/Session Web-Formulare | Location-Dropdown aus `SchoolLocation`-Tabelle statt Freitext | Templates + Routes |

Fixes 1 und 2 sind trivial und risikofrei. Fix 3 erfordert Template-Anpassungen mit Location-Picker.

---

## 5. Gesamtbewertung

- **API-Ebene (iOS + JSON-Clients):** Vollstaendig aligned, keine Gaps
- **Klassisches Web-Admin:** Funktional, aber 3 Legacy-Stellen ohne `location_id`
- **Keine alten role-only Sonderpfade:** Alle Feature-Gates laufen ueber Capability-Contract
- **Keine alten Workarounds in API-Endpoints:** Sauber migriert
