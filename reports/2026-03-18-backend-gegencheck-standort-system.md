# Backend-Gegencheck v2: Standort-System & Cross-Platform Alignment

**Datum:** 2026-03-18 (aktualisiert nach Recheck)
**Erstellt von:** Backend-Team (Claude Code)
**Scope:** Verifizierung des Standort-Systems gegen Shared-Docs-Contracts — zweiter Durchlauf nach Umsetzung aller Location-Fixes

---

## 1. Ergebnis: Alle 3 Legacy-Luecken aus v1 sind BEHOBEN

| Luecke (aus v1-Report) | Status | Fix |
|---|---|---|
| `_auto_generate_sessions()` ohne `location_id` | BEHOBEN | `courses.py:42` kopiert jetzt `location_id=course.location_id` |
| Web-Formulare nur Freitext | BEHOBEN | Alle 5 Templates haben `<select name="location_id">` Dropdown |
| Course Duplicate ohne `location_id` | BEHOBEN | `courses.py:313` kopiert `location_id=original.location_id` |

---

## 2. Neu umgesetzte Komponenten (seit v1)

| Feature | Dateien | Details |
|---|---|---|
| Standort-CRUD in Schuleinstellungen | `settings.py:425-536`, `school_settings.html` | Vollstaendiges UI mit Create/Edit/Delete |
| `_resolve_location_fields()` Helper | `courses.py:64-75`, `sessions.py:35-49` | Zentrale Validierung: location_id gegen School pruefen, Fallback auf Freitext |
| Plan-Limit-Pruefung | `settings.py:440-445`, `locations.py:64-67` | `max_locations` wird bei Create geprueft |
| Loesch-Schutz | `settings.py:520-524` | Verhindert Delete wenn Standort in Kursen/Sessions referenziert |
| Location-Dropdown in allen Templates | `create_course.html`, `edit_course.html`, `create_session.html`, `edit_session.html`, `makeup_session.html` | Strukturiertes `<select>` mit `display_label` + Freitext-Fallback |

---

## 3. API-Contract-Konformitaet: Vollstaendig aligned

| Komponente | Status | Details |
|---|---|---|
| `SchoolLocation` Model + FK | OK | `location_id` FK, `location_rel` Relationship, `resolved_location`/`effective_location` Properties |
| `GET/POST/PUT/DELETE /api/admin/locations` | OK | Tenant-Isolation, Slug-Uniqueness, Cascade-Delete, Rate-Limiting |
| Course-API | OK | Akzeptiert `location_id`, validiert Tenant, serialisiert `location_id` + `location_object` |
| Session-API | OK | `location_id` in Create/Update/Makeup, Fallback-Kette Session -> Course -> Location |
| Capability-Contract | OK | `multi_location_management` als Plan-Feature, `max_locations` in Limits |
| Serializer (`helpers.py`) | OK | Gibt immer `location_id`, `location`, `location_object` zurueck |
| Schemas (`api_schemas.py`) | OK | Alle CRUD-Schemas haben `location_id = fields.Integer(allow_none=True)` |

---

## 4. Login-Verhalten: Identisch ueber alle Plattformen

| Pruefpunkt | Status |
|---|---|
| `GET /api/auth/capabilities` als kanonische Quelle fuer UI-Gating | OK |
| Feature-Flags steuern Navigation (nicht nur Rolle) | OK |
| Web + iOS nutzen denselben Contract (Version 1) | OK |
| `multi_location_management` + `max_locations` korrekt im Contract | OK |
| Keine role-only Feature-Gates (nur Auth-Checks) | OK |

---

## 5. Plan-Limits korrekt konfiguriert

| Plan | `max_locations` | `multi_location_management` |
|---|---|---|
| Starter | 1 | Nein |
| Professional | 3 | Ja |
| Enterprise | unbegrenzt | Ja |
| Trial | 3 | Ja (Professional-Level) |

---

## 6. Verbleibender Gap: Einzelstunde-Bestaetigung

**Prioritaet:** NIEDRIG

**Datei:** `blueprints/api_admin/requests.py:336-339`
**Schema:** `api_schemas.py:666-678` (`AdminConfirmEinzelstundeSchema`)

`POST /api/admin/einzelstunde-requests/<id>/confirm` akzeptiert nur `location` (String) und `location_type`, aber **kein `location_id`**. Die daraus erzeugte Session wird ohne strukturierten Standort-Bezug angelegt.

**Empfohlener Fix:**
1. `location_id = fields.Integer(allow_none=True)` im Schema ergaenzen
2. Tenant-Validierung im Handler (analog `_resolve_location_fields()`)
3. `new_session.location_id` setzen

**Aufwand:** ~10 Zeilen

---

## 7. Gesamtbewertung

| Bereich | Status |
|---|---|
| API-Ebene (iOS + JSON-Clients) | Vollstaendig aligned |
| Klassisches Web-Admin | Vollstaendig migriert (alle Formulare nutzen `location_id`) |
| Standort-CRUD in Schuleinstellungen | Implementiert mit Plan-Limits und Loesch-Schutz |
| Capability-Contract | Korrekt, keine role-only Sonderpfade |
| Einzelstunde-Bestaetigung | Einziger verbleibender Gap (nur Freitext, kein `location_id`) |

**Fazit:** Das Standort-System ist zu ~98% end-to-end integriert. Der einzige offene Punkt ist die Einzelstunde-Bestaetigung — ein Low-Priority-Fix mit minimalem Aufwand.
