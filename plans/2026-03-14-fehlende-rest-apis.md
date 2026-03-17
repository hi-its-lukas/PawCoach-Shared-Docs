# Fehlende REST APIs Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** REST APIs fuer 6 Features nachrüsten, die bisher nur ueber Web-Formulare (server-rendered HTML) nutzbar sind.

**Architektur:** Jedes Feature wird als neues Modul unter dem bestehenden API-Blueprint registriert. Alle Endpoints folgen dem etablierten Pattern: `@jwt_or_session_required`, `@limiter.limit()`, Marshmallow-Validierung, Tenant-Isolation via `school_id`-Check.

**Tech Stack:** Flask, Marshmallow (Validation), Flask-Limiter, SQLAlchemy, pytest

---

## Bestehende API-Patterns (Referenz)

**Auth:** `@jwt_or_session_required` Decorator, dann `_require_admin()` oder `_require_trainer()` aus `api_admin/helpers.py`

**Tenant-Isolation:** Jede Query filtert nach `school_profile_id == _get_admin_school_id(user)`

**Validierung:** Marshmallow-Schemas in `api_schemas.py`, `@validate_json(SchemaClass)` Decorator

**Rate-Limiting:** `60/hour` fuer Reads, `30/hour` fuer Writes, `10/hour` fuer Deletes

**Fehler:** `jsonify({"error": "..."}), <status>` — keine internen Details leaken

**Serialisierung:** `_serialize_*()` Helper-Funktionen

**Pagination:** `paginate_query()` aus `blueprints/api_helpers.py`

**CSRF:** API-Blueprints sind CSRF-exempt (`csrf.exempt(bp)`)

**Tests:** `client` Fixture, Login via API, `Authorization: Bearer <token>` Header

**Referenz-Implementation:** `blueprints/api_admin/courses.py` zeigt das kanonische Pattern.

---

## Task 1: Setup Wizard API

**Kontext:** Einfachstes Feature, keine externen Abhaengigkeiten. Guter Einstiegspunkt.

**Bestehende Web-Routes** (in `blueprints/admin/setup.py`):
- `GET/POST /admin/setup-wizard` — Formular anzeigen/absenden
- `POST /admin/setup-wizard/skip` — Setup ueberspringen

**Files:**
- Create: `blueprints/api_admin/setup.py`
- Modify: `blueprints/api_admin/__init__.py` — Import hinzufuegen
- Modify: `api_schemas.py` — `AdminSetupWizardSchema` hinzufuegen
- Test: `tests/test_api_admin_setup.py`

**Endpoints:**

| Method | Path | Rate-Limit | Beschreibung |
|--------|------|------------|--------------|
| GET | `/api/admin/setup/status` | 60/hour | Setup-Status und aktuelle Schuldaten |
| POST | `/api/admin/setup/complete` | 30/hour | Setup-Daten absenden |
| POST | `/api/admin/setup/skip` | 30/hour | Setup ueberspringen |

### Step 1: Schema in `api_schemas.py` definieren

```python
class AdminSetupWizardSchema(Schema):
    class Meta:
        unknown = EXCLUDE

    name = fields.String(required=True, validate=validate.Length(min=1, max=200))
    email = fields.Email(allow_none=True)
    phone = fields.String(allow_none=True, validate=validate.Length(max=30))
    address = fields.String(allow_none=True, validate=validate.Length(max=500))
    city = fields.String(allow_none=True, validate=validate.Length(max=100))
    zip_code = fields.String(allow_none=True, validate=validate.Length(max=10))
    description = fields.String(allow_none=True, validate=validate.Length(max=5000))
    website = fields.String(allow_none=True, validate=validate.Length(max=500))
```

### Step 2: Failing Test schreiben

```python
# tests/test_api_admin_setup.py
"""Tests fuer Setup Wizard API."""
import pytest
from tests.conftest import login_api_user


def test_setup_status_returns_current_data(client, sample_school):
    """GET /api/admin/setup/status muss Schuldaten zurueckgeben."""
    school, admin = sample_school
    token = login_api_user(client, admin)
    resp = client.get("/api/admin/setup/status",
                      headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 200
    data = resp.get_json()
    assert "setup_completed" in data
    assert "school" in data


def test_setup_status_requires_auth(client):
    """Ohne Auth muss 401 kommen."""
    resp = client.get("/api/admin/setup/status")
    assert resp.status_code == 401


def test_setup_complete_updates_school(client, sample_school):
    """POST /api/admin/setup/complete muss Schuldaten aktualisieren."""
    school, admin = sample_school
    token = login_api_user(client, admin)
    resp = client.post("/api/admin/setup/complete",
                       headers={"Authorization": f"Bearer {token}"},
                       json={"name": "Neue Hundeschule", "email": "test@example.com",
                             "phone": "+49123456789", "city": "Berlin"})
    assert resp.status_code == 200
    data = resp.get_json()
    assert data["school"]["name"] == "Neue Hundeschule"
    assert data["setup_completed"] is True


def test_setup_complete_validates_input(client, sample_school):
    """Leerer Name muss abgelehnt werden."""
    school, admin = sample_school
    token = login_api_user(client, admin)
    resp = client.post("/api/admin/setup/complete",
                       headers={"Authorization": f"Bearer {token}"},
                       json={"name": ""})
    assert resp.status_code == 422


def test_setup_skip(client, sample_school):
    """POST /api/admin/setup/skip muss setup_completed setzen."""
    school, admin = sample_school
    token = login_api_user(client, admin)
    resp = client.post("/api/admin/setup/skip",
                       headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 200
    assert resp.get_json()["setup_completed"] is True
```

### Step 3: Test ausfuehren, Fail verifizieren

```bash
pytest tests/test_api_admin_setup.py -v
```

### Step 4: API-Modul implementieren

```python
# blueprints/api_admin/setup.py
"""Setup Wizard API."""
from flask import jsonify, request

from blueprints.api_admin import api_admin_bp as bp
from blueprints.api_admin.helpers import _require_admin, _get_admin_school_id
from extensions import limiter, db
from models import SchoolProfile
from api_schemas import AdminSetupWizardSchema
from decorators import jwt_or_session_required, validate_json


@bp.route("/setup/status", methods=["GET"])
@jwt_or_session_required
@limiter.limit("60 per hour")
def setup_status():
    user, err = _require_admin()
    if err:
        return err
    school = SchoolProfile.query.get(_get_admin_school_id(user))
    if not school:
        return jsonify({"error": "Schule nicht gefunden"}), 404
    return jsonify({
        "setup_completed": school.setup_completed or False,
        "school": {
            "name": school.name,
            "email": school.email,
            "phone": school.phone,
            "address": school.address,
            "city": school.city,
            "zip_code": school.zip_code,
            "description": school.description,
            "website": school.website,
        }
    })


@bp.route("/setup/complete", methods=["POST"])
@jwt_or_session_required
@limiter.limit("30 per hour")
@validate_json(AdminSetupWizardSchema)
def setup_complete():
    user, err = _require_admin()
    if err:
        return err
    school = SchoolProfile.query.get(_get_admin_school_id(user))
    if not school:
        return jsonify({"error": "Schule nicht gefunden"}), 404
    data = request.validated_data
    for field in ("name", "email", "phone", "address", "city", "zip_code", "description", "website"):
        if field in data:
            setattr(school, field, data[field])
    school.setup_completed = True
    db.session.commit()
    return jsonify({
        "setup_completed": True,
        "school": {
            "name": school.name, "email": school.email,
            "phone": school.phone, "city": school.city,
        }
    })


@bp.route("/setup/skip", methods=["POST"])
@jwt_or_session_required
@limiter.limit("30 per hour")
def setup_skip():
    user, err = _require_admin()
    if err:
        return err
    school = SchoolProfile.query.get(_get_admin_school_id(user))
    if not school:
        return jsonify({"error": "Schule nicht gefunden"}), 404
    school.setup_completed = True
    db.session.commit()
    return jsonify({"setup_completed": True})
```

### Step 5: In `__init__.py` registrieren

```python
from blueprints.api_admin import setup  # noqa: F401
```

### Step 6: Tests ausfuehren, Pass verifizieren

```bash
pytest tests/test_api_admin_setup.py -v
```

### Step 7: Commit

```bash
git add blueprints/api_admin/setup.py blueprints/api_admin/__init__.py api_schemas.py tests/test_api_admin_setup.py
git commit -m "feat: Setup Wizard REST API"
```

---

## Task 2: Accounting Dashboard API

**Bestehende Web-Routes** (in `blueprints/admin/invoicing.py`):
- `GET /admin/accounting` — Sync-Jobs auflisten
- `POST /admin/accounting/retry/<job_id>` — Job wiederholen
- `POST /admin/accounting/retry-all` — Alle fehlgeschlagenen wiederholen
- `POST /admin/accounting/sync-now` — Sofort synchronisieren
- `POST /admin/accounting/test-connection` — Verbindung testen

**Files:**
- Create: `blueprints/api_admin/accounting.py`
- Modify: `blueprints/api_admin/__init__.py`
- Test: `tests/test_api_admin_accounting.py`

**Endpoints:**

| Method | Path | Rate-Limit | Beschreibung |
|--------|------|------------|--------------|
| GET | `/api/admin/accounting/jobs` | 60/hour | Jobs auflisten (paginiert, filterbar) |
| POST | `/api/admin/accounting/jobs/<id>/retry` | 10/hour | Einzelnen Job wiederholen |
| POST | `/api/admin/accounting/retry-all` | 10/hour | Alle fehlgeschlagenen wiederholen |
| POST | `/api/admin/accounting/sync-now` | 10/hour | Sofort synchronisieren |
| POST | `/api/admin/accounting/test-connection` | 10/hour | Verbindung testen |

### Step 1: Failing Test schreiben

```python
# tests/test_api_admin_accounting.py
"""Tests fuer Accounting Dashboard API."""
from tests.conftest import login_api_user


def test_list_accounting_jobs(client, sample_school):
    """GET /api/admin/accounting/jobs muss Jobs zurueckgeben."""
    school, admin = sample_school
    token = login_api_user(client, admin)
    resp = client.get("/api/admin/accounting/jobs",
                      headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 200
    data = resp.get_json()
    assert "jobs" in data
    assert "total" in data


def test_list_accounting_jobs_filterable(client, sample_school):
    """Jobs muessen nach Status filterbar sein."""
    school, admin = sample_school
    token = login_api_user(client, admin)
    resp = client.get("/api/admin/accounting/jobs?status=failed",
                      headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 200


def test_retry_job_requires_auth(client):
    """Retry ohne Auth muss 401 geben."""
    resp = client.post("/api/admin/accounting/jobs/1/retry")
    assert resp.status_code == 401


def test_test_connection(client, sample_school):
    """Test-Connection muss Ergebnis zurueckgeben."""
    school, admin = sample_school
    token = login_api_user(client, admin)
    resp = client.post("/api/admin/accounting/test-connection",
                       headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code in (200, 400, 503)
    assert "status" in resp.get_json() or "error" in resp.get_json()


def test_sync_now(client, sample_school):
    """Sync-Now muss Jobs verarbeiten."""
    school, admin = sample_school
    token = login_api_user(client, admin)
    resp = client.post("/api/admin/accounting/sync-now",
                       headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 200
    assert "processed" in resp.get_json()


def test_tenant_isolation(client, sample_school, sample_school_2):
    """Admin darf nur Jobs seiner eigenen Schule sehen."""
    school1, admin1 = sample_school
    school2, admin2 = sample_school_2
    token2 = login_api_user(client, admin2)
    resp = client.get("/api/admin/accounting/jobs",
                      headers={"Authorization": f"Bearer {token2}"})
    data = resp.get_json()
    for job in data.get("jobs", []):
        assert job.get("school_id") == school2.id
```

### Step 2: Test ausfuehren, Fail verifizieren

### Step 3: Implementierung

```python
# blueprints/api_admin/accounting.py
"""Accounting Dashboard API."""
from flask import jsonify, request

from blueprints.api_admin import api_admin_bp as bp
from blueprints.api_admin.helpers import _require_admin, _get_admin_school_id
from blueprints.api_helpers import paginate_query
from extensions import limiter, db
from models import AccountingSyncJob
from decorators import jwt_or_session_required


def _serialize_job(job):
    return {
        "id": job.id,
        "school_id": job.school_profile_id,
        "job_type": job.job_type,
        "status": job.status,
        "error_message": job.error_message,
        "created_at": job.created_at.isoformat() if job.created_at else None,
        "completed_at": job.completed_at.isoformat() if job.completed_at else None,
        "retry_count": job.retry_count,
    }


@bp.route("/accounting/jobs", methods=["GET"])
@jwt_or_session_required
@limiter.limit("60 per hour")
def list_accounting_jobs():
    user, err = _require_admin()
    if err:
        return err
    school_id = _get_admin_school_id(user)
    query = AccountingSyncJob.query.filter_by(school_profile_id=school_id)
    status = request.args.get("status")
    if status:
        query = query.filter_by(status=status)
    job_type = request.args.get("type")
    if job_type:
        query = query.filter_by(job_type=job_type)
    query = query.order_by(AccountingSyncJob.created_at.desc())
    page_data = paginate_query(query)
    return jsonify({
        "jobs": [_serialize_job(j) for j in page_data.items],
        "total": page_data.total,
        "page": page_data.page,
        "pages": page_data.pages,
    })


@bp.route("/accounting/jobs/<int:job_id>/retry", methods=["POST"])
@jwt_or_session_required
@limiter.limit("10 per hour")
def retry_accounting_job(job_id):
    user, err = _require_admin()
    if err:
        return err
    school_id = _get_admin_school_id(user)
    job = AccountingSyncJob.query.filter_by(id=job_id, school_profile_id=school_id).first()
    if not job:
        return jsonify({"error": "Job nicht gefunden"}), 404
    job.status = "pending"
    job.error_message = None
    db.session.commit()
    return jsonify({"status": "queued", "job": _serialize_job(job)})


@bp.route("/accounting/retry-all", methods=["POST"])
@jwt_or_session_required
@limiter.limit("10 per hour")
def retry_all_accounting_jobs():
    user, err = _require_admin()
    if err:
        return err
    school_id = _get_admin_school_id(user)
    count = AccountingSyncJob.query.filter_by(
        school_profile_id=school_id, status="failed"
    ).update({"status": "pending", "error_message": None})
    db.session.commit()
    return jsonify({"status": "queued", "count": count})


@bp.route("/accounting/sync-now", methods=["POST"])
@jwt_or_session_required
@limiter.limit("10 per hour")
def sync_accounting_now():
    user, err = _require_admin()
    if err:
        return err
    school_id = _get_admin_school_id(user)
    from blueprints.accounting import process_pending_jobs
    processed = process_pending_jobs(school_id)
    return jsonify({"status": "completed", "processed": processed})


@bp.route("/accounting/test-connection", methods=["POST"])
@jwt_or_session_required
@limiter.limit("10 per hour")
def test_accounting_connection():
    user, err = _require_admin()
    if err:
        return err
    school_id = _get_admin_school_id(user)
    from blueprints.accounting import test_connection
    try:
        result = test_connection(school_id)
        return jsonify({"status": "ok", "provider": result.get("provider")})
    except Exception:
        return jsonify({"status": "error", "error": "Verbindung fehlgeschlagen"}), 503
```

### Step 4: In `__init__.py` registrieren

### Step 5: Tests ausfuehren, Pass verifizieren

### Step 6: Commit

```bash
git add blueprints/api_admin/accounting.py blueprints/api_admin/__init__.py tests/test_api_admin_accounting.py
git commit -m "feat: Accounting Dashboard REST API"
```

---

## Task 3: Trainer Availability Admin API

**Kontext:** Die Trainer-API `/api/trainer/availability` existiert bereits fuer Trainer-Self-Service. Admins brauchen Endpoints um Slots fuer beliebige Trainer ihrer Schule zu verwalten.

**Bestehende Web-Routes** (in `blueprints/trainer.py`, Zeilen 242-303):
- `GET/POST /trainer/kalender/slot/*` — CRUD fuer Verfuegbarkeitsslots

**Files:**
- Create: `blueprints/api_admin/availability.py`
- Modify: `blueprints/api_admin/__init__.py`
- Modify: `api_schemas.py` — Schemas hinzufuegen
- Test: `tests/test_api_admin_availability.py`

**Endpoints:**

| Method | Path | Rate-Limit | Beschreibung |
|--------|------|------------|--------------|
| GET | `/api/admin/availability` | 60/hour | Alle Slots der Schule (filterbar nach `trainer_id`, Datumsbereich) |
| POST | `/api/admin/availability` | 30/hour | Slot fuer beliebigen Trainer erstellen |
| PUT | `/api/admin/availability/<id>` | 30/hour | Slot aktualisieren |
| DELETE | `/api/admin/availability/<id>` | 10/hour | Slot loeschen |

### Step 1: Schemas definieren

```python
class AdminCreateAvailabilitySchema(Schema):
    class Meta:
        unknown = EXCLUDE

    trainer_id = fields.Integer(required=True)
    slot_start = fields.String(required=True)  # ISO datetime
    slot_end = fields.String(required=True)
    course_id = fields.Integer(allow_none=True, load_default=None)


class AdminUpdateAvailabilitySchema(Schema):
    class Meta:
        unknown = EXCLUDE

    slot_start = fields.String()
    slot_end = fields.String()
    course_id = fields.Integer(allow_none=True)
```

### Step 2: Failing Tests schreiben

```python
# tests/test_api_admin_availability.py
"""Tests fuer Trainer Availability Admin API."""
from tests.conftest import login_api_user


def test_list_availability_slots(client, sample_school_with_trainer):
    """Admin muss alle Slots seiner Schule sehen."""
    school, admin, trainer = sample_school_with_trainer
    token = login_api_user(client, admin)
    resp = client.get("/api/admin/availability",
                      headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 200
    assert "slots" in resp.get_json()


def test_create_slot_for_trainer(client, sample_school_with_trainer):
    """Admin muss Slot fuer Trainer erstellen koennen."""
    school, admin, trainer = sample_school_with_trainer
    token = login_api_user(client, admin)
    resp = client.post("/api/admin/availability",
                       headers={"Authorization": f"Bearer {token}"},
                       json={"trainer_id": trainer.id,
                             "slot_start": "2026-04-01T09:00:00",
                             "slot_end": "2026-04-01T10:00:00"})
    assert resp.status_code == 201
    assert resp.get_json()["slot"]["trainer_id"] == trainer.id


def test_create_slot_wrong_trainer_rejected(client, sample_school_with_trainer, sample_school_2):
    """Trainer einer anderen Schule muss abgelehnt werden."""
    school, admin, _ = sample_school_with_trainer
    _, _, other_trainer = sample_school_2  # Trainer einer anderen Schule
    token = login_api_user(client, admin)
    resp = client.post("/api/admin/availability",
                       headers={"Authorization": f"Bearer {token}"},
                       json={"trainer_id": other_trainer.id,
                             "slot_start": "2026-04-01T09:00:00",
                             "slot_end": "2026-04-01T10:00:00"})
    assert resp.status_code in (403, 404)


def test_delete_slot(client, sample_school_with_trainer):
    """Admin muss Slot loeschen koennen."""
    school, admin, trainer = sample_school_with_trainer
    token = login_api_user(client, admin)
    # Erst erstellen
    create_resp = client.post("/api/admin/availability",
                              headers={"Authorization": f"Bearer {token}"},
                              json={"trainer_id": trainer.id,
                                    "slot_start": "2026-04-02T09:00:00",
                                    "slot_end": "2026-04-02T10:00:00"})
    slot_id = create_resp.get_json()["slot"]["id"]
    # Dann loeschen
    resp = client.delete(f"/api/admin/availability/{slot_id}",
                         headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 204
```

### Step 3: Implementierung (analog zu `api_trainer/schedule.py` Pattern)

### Step 4: Registrierung, Tests, Commit

```bash
git add blueprints/api_admin/availability.py blueprints/api_admin/__init__.py api_schemas.py tests/test_api_admin_availability.py
git commit -m "feat: Trainer Availability Admin REST API"
```

---

## Task 4: Manual Customer Booking API

**Bestehende Web-Route** (in `blueprints/trainer.py`, Zeilen 311-410):
- `POST /trainer/kalender/buch-kunde` — Session + Booking erstellen

**Files:**
- Create: `blueprints/api_admin/bookings.py`
- Modify: `blueprints/api_admin/__init__.py`
- Modify: `api_schemas.py`
- Test: `tests/test_api_admin_bookings.py`

**Endpoints:**

| Method | Path | Rate-Limit | Beschreibung |
|--------|------|------------|--------------|
| POST | `/api/admin/bookings/direct` | 30/hour | Direkte Buchung (Session + Booking) erstellen |
| GET | `/api/admin/customers/<id>/dogs` | 60/hour | Hunde eines Kunden auflisten |

### Step 1: Schema

```python
class AdminCreateBookingSchema(Schema):
    class Meta:
        unknown = EXCLUDE

    customer_id = fields.Integer(required=True)
    dog_id = fields.Integer(required=True)
    course_id = fields.Integer(required=True)
    trainer_id = fields.Integer(allow_none=True, load_default=None)
    slot_start = fields.String(required=True)
    slot_end = fields.String(required=True)
    send_payment = fields.Boolean(load_default=False)
```

### Step 2: Failing Tests

```python
# tests/test_api_admin_bookings.py
"""Tests fuer Manual Customer Booking API."""
from tests.conftest import login_api_user


def test_create_direct_booking(client, sample_school_full):
    """Admin muss direkte Buchung erstellen koennen."""
    school, admin, trainer, customer, dog, course = sample_school_full
    token = login_api_user(client, admin)
    resp = client.post("/api/admin/bookings/direct",
                       headers={"Authorization": f"Bearer {token}"},
                       json={"customer_id": customer.id, "dog_id": dog.id,
                             "course_id": course.id, "trainer_id": trainer.id,
                             "slot_start": "2026-04-01T09:00:00",
                             "slot_end": "2026-04-01T10:00:00"})
    assert resp.status_code == 201
    data = resp.get_json()
    assert "booking" in data
    assert "session" in data


def test_create_booking_wrong_customer(client, sample_school_full, sample_school_2):
    """Kunde einer anderen Schule muss abgelehnt werden."""
    school, admin, _, _, _, course = sample_school_full
    _, _, _, other_customer, other_dog, _ = sample_school_2
    token = login_api_user(client, admin)
    resp = client.post("/api/admin/bookings/direct",
                       headers={"Authorization": f"Bearer {token}"},
                       json={"customer_id": other_customer.id, "dog_id": other_dog.id,
                             "course_id": course.id,
                             "slot_start": "2026-04-01T09:00:00",
                             "slot_end": "2026-04-01T10:00:00"})
    assert resp.status_code in (403, 404)


def test_get_customer_dogs(client, sample_school_full):
    """Admin muss Hunde eines Kunden abrufen koennen."""
    school, admin, _, customer, dog, _ = sample_school_full
    token = login_api_user(client, admin)
    resp = client.get(f"/api/admin/customers/{customer.id}/dogs",
                      headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 200
    data = resp.get_json()
    assert "dogs" in data
    assert len(data["dogs"]) >= 1


def test_create_booking_validates_input(client, sample_school_full):
    """Fehlende Pflichtfelder muessen abgelehnt werden."""
    school, admin, _, _, _, _ = sample_school_full
    token = login_api_user(client, admin)
    resp = client.post("/api/admin/bookings/direct",
                       headers={"Authorization": f"Bearer {token}"},
                       json={"customer_id": 1})  # Fehlende Felder
    assert resp.status_code == 422
```

### Step 3: Implementierung — Geschaeftslogik aus `blueprints/trainer.py:kalender_book_customer()` extrahieren

### Step 4: Registrierung, Tests, Commit

```bash
git add blueprints/api_admin/bookings.py blueprints/api_admin/__init__.py api_schemas.py tests/test_api_admin_bookings.py
git commit -m "feat: Manual Customer Booking REST API"
```

---

## Task 5: Stripe Connect Onboarding API

**Bestehende Web-Routes** (in `blueprints/saas.py`, Zeilen 734-855):
- `GET /admin/stripe-connect/onboard` — Onboarding starten (Redirect zu Stripe)
- `GET /admin/stripe-connect/return` — Rueckkehr von Stripe
- `GET /admin/stripe-connect/status` — Status (gibt bereits JSON zurueck)
- `GET /admin/stripe-connect/dashboard` — Redirect zu Stripe Dashboard

**Files:**
- Create: `blueprints/api_admin/stripe_connect.py`
- Modify: `blueprints/api_admin/__init__.py`
- Test: `tests/test_api_admin_stripe_connect.py`

**Endpoints:**

| Method | Path | Rate-Limit | Beschreibung |
|--------|------|------------|--------------|
| GET | `/api/admin/stripe-connect/status` | 60/hour | Connect-Account-Status |
| POST | `/api/admin/stripe-connect/onboard` | 10/hour | Onboarding starten, URL zurueckgeben |
| POST | `/api/admin/stripe-connect/refresh-status` | 10/hour | Status von Stripe aktualisieren |
| GET | `/api/admin/stripe-connect/dashboard-url` | 30/hour | Dashboard-Login-URL |

### Step 1: Failing Tests

```python
# tests/test_api_admin_stripe_connect.py
"""Tests fuer Stripe Connect Onboarding API."""
from unittest.mock import patch
from tests.conftest import login_api_user


def test_stripe_connect_status(client, sample_school):
    """Status-Endpoint muss Connect-Daten zurueckgeben."""
    school, admin = sample_school
    token = login_api_user(client, admin)
    resp = client.get("/api/admin/stripe-connect/status",
                      headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 200
    data = resp.get_json()
    assert "connected" in data


def test_stripe_connect_onboard_returns_url(client, sample_school):
    """Onboard-Endpoint muss URL zurueckgeben statt Redirect."""
    school, admin = sample_school
    token = login_api_user(client, admin)
    with patch("stripe.AccountLink.create") as mock_link:
        mock_link.return_value = type("obj", (), {"url": "https://connect.stripe.com/test"})()
        with patch("stripe.Account.create") as mock_account:
            mock_account.return_value = type("obj", (), {"id": "acct_test123"})()
            resp = client.post("/api/admin/stripe-connect/onboard",
                               headers={"Authorization": f"Bearer {token}"},
                               json={"return_url": "https://example.com/return",
                                     "refresh_url": "https://example.com/refresh"})
    assert resp.status_code == 200
    data = resp.get_json()
    assert "onboarding_url" in data


def test_stripe_connect_dashboard_url(client, sample_school_with_stripe):
    """Dashboard-URL muss als JSON zurueckgegeben werden."""
    school, admin = sample_school_with_stripe
    token = login_api_user(client, admin)
    with patch("stripe.Account.create_login_link") as mock_link:
        mock_link.return_value = type("obj", (), {"url": "https://dashboard.stripe.com/test"})()
        resp = client.get("/api/admin/stripe-connect/dashboard-url",
                          headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 200
    assert "url" in resp.get_json()
```

### Step 2-5: Implementierung (Logik aus `saas.py` extrahieren, URLs statt Redirects zurueckgeben)

### Step 6: Commit

```bash
git add blueprints/api_admin/stripe_connect.py blueprints/api_admin/__init__.py tests/test_api_admin_stripe_connect.py
git commit -m "feat: Stripe Connect Onboarding REST API"
```

---

## Task 6: Price Management Platform Admin API

**Kontext:** Platform-Admin-Level (nicht School-Admin). Erfordert `user.is_platform_admin()`.

**Bestehende Web-Routes** (in `blueprints/saas.py`, Zeilen 1898-2074):
- `GET /admin/price-management` — Planpreise und Addon-Preise auflisten
- `POST /admin/price-management/plan/<plan_id>` — Planpreis aendern (neuer Stripe Price)
- `POST /admin/price-management/addon/<int:price_id>` — Addon-Preis aendern

**Files:**
- Create: `blueprints/api_platform_admin/prices.py`
- Modify: `blueprints/api_platform_admin/__init__.py`
- Modify: `api_schemas.py`
- Test: `tests/test_api_platform_admin_prices.py`

**Endpoints:**

| Method | Path | Rate-Limit | Beschreibung |
|--------|------|------------|--------------|
| GET | `/api/platform-admin/prices` | 60/hour | Alle Plan- und Addon-Preise |
| PUT | `/api/platform-admin/prices/plans/<plan_id>` | 10/hour | Planpreis aendern |
| PUT | `/api/platform-admin/prices/addons/<int:price_id>` | 10/hour | Addon-Preis aendern |

### Step 1: Schemas

```python
class PlatformUpdatePlanPriceSchema(Schema):
    class Meta:
        unknown = EXCLUDE

    price_cents = fields.Integer(required=True, validate=validate.Range(min=0))
    migrate_existing = fields.Boolean(load_default=False)


class PlatformUpdateAddonPriceSchema(Schema):
    class Meta:
        unknown = EXCLUDE

    price_cents = fields.Integer(required=True, validate=validate.Range(min=0))
```

### Step 2: Failing Tests

```python
# tests/test_api_platform_admin_prices.py
"""Tests fuer Price Management Platform Admin API."""
from tests.conftest import login_api_user


def test_list_prices(client, platform_admin):
    """Platform-Admin muss alle Preise sehen."""
    token = login_api_user(client, platform_admin)
    resp = client.get("/api/platform-admin/prices",
                      headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 200
    data = resp.get_json()
    assert "plans" in data
    assert "addons" in data


def test_list_prices_denied_for_school_admin(client, sample_school):
    """School-Admin darf Preise nicht verwalten."""
    school, admin = sample_school
    token = login_api_user(client, admin)
    resp = client.get("/api/platform-admin/prices",
                      headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 403


def test_update_plan_price(client, platform_admin):
    """Platform-Admin muss Planpreis aendern koennen."""
    token = login_api_user(client, platform_admin)
    with patch("stripe.Price.create") as mock_price:
        mock_price.return_value = type("obj", (), {"id": "price_new123"})()
        resp = client.put("/api/platform-admin/prices/plans/starter",
                          headers={"Authorization": f"Bearer {token}"},
                          json={"price_cents": 2999})
    assert resp.status_code == 200


def test_update_plan_price_validates_min(client, platform_admin):
    """Negativer Preis muss abgelehnt werden."""
    token = login_api_user(client, platform_admin)
    resp = client.put("/api/platform-admin/prices/plans/starter",
                      headers={"Authorization": f"Bearer {token}"},
                      json={"price_cents": -100})
    assert resp.status_code == 422
```

### Step 3-5: Implementierung, Registrierung, Tests

### Step 6: Commit

```bash
git add blueprints/api_platform_admin/prices.py blueprints/api_platform_admin/__init__.py api_schemas.py tests/test_api_platform_admin_prices.py
git commit -m "feat: Price Management Platform Admin REST API"
```

---

## Implementierungs-Reihenfolge

1. **Task 1: Setup Wizard API** — Am einfachsten, keine externen Abhaengigkeiten
2. **Task 2: Accounting Dashboard API** — Straightforward CRUD
3. **Task 3: Trainer Availability Admin API** — Erweitert bestehendes Pattern
4. **Task 4: Manual Customer Booking API** — Komplexeste Geschaeftslogik
5. **Task 5: Stripe Connect API** — Erfordert Stripe-Mocking in Tests
6. **Task 6: Price Management API** — Platform-Admin-Level, Stripe-Mocking

## Querschnittsthemen

- **Auth:** Alle Endpoints verwenden `@jwt_or_session_required` + `@limiter.limit()`
- **CSRF:** API-Blueprints sind CSRF-exempt (auf Blueprint-Ebene)
- **Fehler:** Keine PII oder interne Details in Error-Responses
- **DB-Writes:** In try/except mit Rollback wrappen
- **Tenant-Isolation:** In jedem Test verifizieren
- **Keine Secrets:** Stripe-Keys, Account-IDs etc. nicht in Responses exponieren
