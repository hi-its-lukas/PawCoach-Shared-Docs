# Gegencheck-Befund: Einzelstunde-Bestaetigung fehlt location_id

**Datum:** 2026-03-18
**Erstellt von:** QA / Code-Review (Pruefinstanz)
**Adressat:** Head of Development
**Prinzip:** Vier-Augen — wer prueft, programmiert nicht; wer programmiert, prueft nicht.

---

## Status: GESCHLOSSEN — Umsetzung erfolgt und verifiziert

---

## 1. Zusammenfassung

Der urspruengliche Gegencheck hat korrekt festgestellt, dass der gepushte Stand den
Endpoint `POST /api/admin/einzelstunde-requests/<id>/confirm` noch nicht auf
`location_id` migriert hatte. Dieser Befund ist inzwischen umgesetzt und verifiziert.

Aktueller Stand:

- `AdminConfirmEinzelstundeSchema` akzeptiert `location_id`
- der Handler validiert `location_id` tenant-sicher gegen die aktuelle Schule
- die erzeugte Session setzt `location_id`
- API-Tests decken gueltige und fremde `location_id` ab

Der historische Befund bleibt zur Nachvollziehbarkeit erhalten. Er beschreibt den
Stand **vor** dem nachgezogenen Fix.

Damals galt:

- alle anderen Session-erzeugenden Flows (Create, Update, Makeup) `location_id` korrekt
  akzeptieren, validieren und persistieren
- der Backend-Gegencheck-Report v2 (`backend-gegencheck-standort-system.md`, Abschnitt 6)
  diesen Punkt als "BEHOBEN" markiert
- die CROSS-PLATFORM-FEATURE-SPEC (Abschnitt 5, Zeile 301) den Endpoint als Teil des
  Standort-Flows dokumentiert

---

## 2. Betroffene Dateien und Stellen

### 2.1 Schema — `location_id` fehlt

**Datei:** `Dog-School-Manager/api_schemas.py`, Zeilen 666-679

```python
class AdminConfirmEinzelstundeSchema(Schema):
    """Schema fuer POST /api/admin/einzelstunde-requests/<id>/confirm."""

    class Meta:
        unknown = EXCLUDE

    start_time = fields.DateTime(required=True)
    trainer_id = fields.Integer(allow_none=True)
    duration_minutes = fields.Integer(
        validate=validate.Range(min=1, max=1440),
    )
    location = fields.String(allow_none=True)
    location_type = fields.String(allow_none=True)
    # FEHLT: location_id = fields.Integer(allow_none=True)
```

### 2.2 Handler — keine Validierung, kein Setzen von location_id

**Datei:** `Dog-School-Manager/blueprints/api_admin/requests.py`, Zeilen 298-372

Der Handler liest `location_id` nicht aus den Request-Daten. Die erzeugte Session
wird nur mit den Freitext-Feldern `location` und `location_type` befuellt:

```python
new_session = Session(
    course_id=req.course_id,
    trainer_id=trainer_id,
    start_time=start_time,
    max_participants=1,
    duration_minutes=duration_minutes,
)
if data.get("location"):
    new_session.location = data["location"]
if data.get("location_type"):
    new_session.location_type = data["location_type"]
# FEHLT: location_id wird nie gelesen, validiert oder gesetzt
```

---

## 3. Erwartetes Verhalten (analog zu Session-Create und Makeup)

Referenz-Implementierung: `blueprints/api_admin/sessions.py`, Zeilen 154-170

1. Schema akzeptiert `location_id = fields.Integer(allow_none=True)`
2. Handler validiert `location_id` gegen die Schule (Tenant-Isolation):
   ```python
   if data.get("location_id"):
       location = db.session.get(SchoolLocation, data["location_id"])
       if not location or location.school_profile_id != school_id:
           return jsonify({"error": "Standort nicht gefunden"}), 404
   ```
3. Handler setzt `new_session.location_id = location.id`
4. Optional: `location_type` aus dem Standort-Objekt uebernehmen, wenn kein
   expliziter Override im Request

---

## 4. Auswirkung

| Bereich | Auswirkung |
|---------|-----------|
| **iOS** | Einzelstunde-Bestaetigung kann keinen strukturierten Standort setzen; Location-Picker im Confirm-Flow waere wirkungslos |
| **Web** | Admin sieht im Confirm-Dialog keinen Standort-Dropdown (nur Freitext) |
| **Daten** | Ueber Einzelstunde erzeugte Sessions haben `location_id = NULL`, obwohl der zugehoerige Kurs einen Standort hat |
| **Konsistenz** | Einziger Session-erzeugende Flow ohne `location_id`-Unterstuetzung |

---

## 5. Empfohlene Aenderungen

### 5.1 `api_schemas.py` — Feld ergaenzen

In `AdminConfirmEinzelstundeSchema` hinzufuegen:

```python
location_id = fields.Integer(allow_none=True)
```

### 5.2 `blueprints/api_admin/requests.py` — Validierung und Persistierung

Im Handler `confirm_einzelstunde_request()`, vor der Session-Erstellung:

```python
location = None
if data.get("location_id"):
    location = db.session.get(SchoolLocation, data["location_id"])
    if not location or location.school_profile_id != school_id:
        return jsonify({"error": "Standort nicht gefunden"}), 404
```

Beim Erstellen der Session:

```python
new_session.location_id = location.id if location else None
if location and not data.get("location_type"):
    new_session.location_type = location.location_type
```

### 5.3 Tests

- Einzelstunde-Bestaetigung mit gueltigem `location_id` → Session hat `location_id` gesetzt
- Einzelstunde-Bestaetigung mit fremdem `location_id` (andere Schule) → 404
- Einzelstunde-Bestaetigung ohne `location_id` → Session hat `location_id = NULL` (Rueckwaertskompatibilitaet)

### 5.4 Doku-Korrektur

`reports/2026-03-18-backend-gegencheck-standort-system.md`, Abschnitt 6: Status von
"BEHOBEN" auf "OFFEN" korrigieren (oder nach tatsaechlicher Umsetzung erneut verifizieren).

---

## 6. Schweregrad zum Befundzeitpunkt

**MITTEL** — Kein Sicherheitsrisiko (da `location_id` gar nicht gelesen wird, kann auch
kein fremder Standort injiziert werden), aber eine funktionale Luecke im ansonsten
vollstaendig migrierten Standort-System.

---

## 7. Pruefprotokoll

```
Geprueft gegen:
- CROSS-PLATFORM-FEATURE-SPEC.md (Version 2026-03-18-v7)
- IOS-DEVELOPER-GUIDE.md (Stand 2026-03-18)
- backend-gegencheck-standort-system.md (v2, 2026-03-18)
- platform-alignment-status-v7.md (2026-03-18)
- openapi.json (aktueller Stand)
- Quellcode Dog-School-Manager (Branch main, HEAD cdf6a87)

Methode: Automatisierte Code-Analyse aller session-erzeugenden Flows,
         Abgleich Schema vs. Handler vs. Shared-Docs-Contract

Ergebnis: 1 Abweichung gefunden (dieser Befund)
          Alle anderen Pruefpunkte (Kurs-CRUD, Session-CRUD, Makeup,
          Standort-CRUD, Capability-Contract, Web-Templates, Plan-Limits)
          sind vollstaendig aligned.
```
