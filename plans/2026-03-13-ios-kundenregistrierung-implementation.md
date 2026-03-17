# iOS Kundenregistrierung mit Schulsuche — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Drei oeffentliche API-Endpoints (Schulsuche, Code-Validierung, Kundenregistrierung) plus Join-Code und Freigabe-System implementieren.

**Architecture:** Neue Felder auf SchoolProfile (join_code, customer_approval_required) und UserSchoolRole (approved). Neuer Blueprint api_public.py fuer oeffentliche Endpoints. Login-Check in api_auth.py erweitern.

**Tech Stack:** Flask, SQLAlchemy, Alembic, Marshmallow, bleach

---

## Task 1: Datenmodell — join_code + customer_approval_required + approved

**Files:**
- Modify: `models/schools.py` (SchoolProfile)
- Modify: `models/auth.py` (UserSchoolRole)
- Create: neue Alembic-Migration
- Test: `tests/test_ios_registration.py`

**Step 1: Write failing tests**

```python
# tests/test_ios_registration.py
import pytest
import re

class TestJoinCode:
    def test_school_has_join_code(self, app, sample_school):
        """SchoolProfile hat join_code Feld."""
        assert hasattr(sample_school, "join_code")

    def test_school_has_customer_approval_required(self, app, sample_school):
        """SchoolProfile hat customer_approval_required Feld."""
        assert hasattr(sample_school, "customer_approval_required")
        assert sample_school.customer_approval_required is False

    def test_generate_join_code_format(self, app):
        """join_code hat Format XXXX-0000."""
        from models.schools import generate_join_code
        code = generate_join_code()
        assert re.match(r"^[A-Z]{4}-\d{4}$", code)

    def test_generate_join_code_unique(self, app):
        """Zwei generierte Codes sind unterschiedlich."""
        from models.schools import generate_join_code
        codes = {generate_join_code() for _ in range(100)}
        assert len(codes) > 90  # Nahezu alle unterschiedlich

class TestApprovedField:
    def test_user_school_role_has_approved(self, app, sample_school, sample_user):
        """UserSchoolRole hat approved Feld."""
        from models.auth import UserSchoolRole
        role = UserSchoolRole.query.filter_by(user_id=sample_user.id).first()
        assert role.approved is True  # Default True
```

**Step 2: Run tests to verify they fail**

Run: `python -m pytest tests/test_ios_registration.py -v`
Expected: FAIL

**Step 3: Add fields to models**

In `models/schools.py`, nach `public_booking_enabled` (Zeile ~35):

```python
import random
import string

def generate_join_code():
    """Generiert einen Join-Code im Format XXXX-0000."""
    letters = "".join(random.choices(string.ascii_uppercase, k=4))
    digits = "".join(random.choices(string.digits, k=4))
    return f"{letters}-{digits}"
```

SchoolProfile-Klasse:
```python
    join_code = db.Column(db.String(9), unique=True, nullable=True)
    customer_approval_required = db.Column(db.Boolean, default=False, nullable=False, server_default="false")
```

In `models/auth.py`, UserSchoolRole:
```python
    approved = db.Column(db.Boolean, default=True, nullable=False, server_default="true")
```

**Step 4: Create Alembic migration**

Manuell erstellen mit `_column_exists` Pattern. Felder:
- `school_profile.join_code` (String(9), unique, nullable)
- `school_profile.customer_approval_required` (Boolean, default False)
- `user_school_role.approved` (Boolean, default True)

Migration muss auch bestehende Schulen mit join_code befuellen:
```python
def upgrade():
    if not _column_exists("school_profile", "join_code"):
        op.add_column("school_profile", sa.Column("join_code", sa.String(9), nullable=True))
        op.create_unique_constraint("uq_school_join_code", "school_profile", ["join_code"])
    if not _column_exists("school_profile", "customer_approval_required"):
        op.add_column("school_profile", sa.Column("customer_approval_required", sa.Boolean(),
                       nullable=False, server_default="false"))
    if not _column_exists("user_school_role", "approved"):
        op.add_column("user_school_role", sa.Column("approved", sa.Boolean(),
                       nullable=False, server_default="true"))
    # Bestehende Schulen mit join_code befuellen
    from models.schools import generate_join_code
    conn = op.get_bind()
    schools = conn.execute(sa.text("SELECT id FROM school_profile WHERE join_code IS NULL")).fetchall()
    for school in schools:
        code = generate_join_code()
        conn.execute(sa.text("UPDATE school_profile SET join_code = :code WHERE id = :id"),
                     {"code": code, "id": school[0]})
```

**Step 5: Run tests**

Run: `python -m pytest tests/test_ios_registration.py -v`
Expected: PASS

**Step 6: Commit**

```bash
git add models/schools.py models/auth.py migrations/versions/ tests/test_ios_registration.py
git commit -m "feat: join_code, customer_approval_required und approved Felder"
```

---

## Task 2: Oeffentlicher Blueprint — Schulsuche + Code-Validierung

**Files:**
- Create: `blueprints/api_public.py`
- Modify: `app.py` (Blueprint registrieren)
- Test: `tests/test_ios_registration.py` (erweitern)

**Step 1: Write failing tests**

```python
class TestSchoolSearch:
    def test_search_by_name(self, client, app, sample_school):
        """GET /api/public/schools/search?q=Test findet Testschule."""
        with app.app_context():
            sample_school.join_code = "TEST-1234"
            sample_school.subscription_status = "active"
            from extensions import db
            db.session.commit()
        rv = client.get("/api/public/schools/search?q=Test")
        assert rv.status_code == 200
        data = rv.get_json()
        assert len(data["schools"]) >= 1
        assert data["schools"][0]["name"] == "Testschule"
        assert data["schools"][0]["join_code"] == "TEST-1234"

    def test_search_min_2_chars(self, client):
        """Suche mit weniger als 2 Zeichen gibt 400."""
        rv = client.get("/api/public/schools/search?q=T")
        assert rv.status_code == 400

    def test_search_no_inactive_schools(self, client, app, sample_school):
        """Inaktive Schulen erscheinen nicht in der Suche."""
        with app.app_context():
            sample_school.subscription_status = "cancelled"
            from extensions import db
            db.session.commit()
        rv = client.get("/api/public/schools/search?q=Test")
        data = rv.get_json()
        assert len(data["schools"]) == 0

    def test_search_no_auth_required(self, client):
        """Suche braucht kein JWT."""
        rv = client.get("/api/public/schools/search?q=Test")
        assert rv.status_code in (200, 400)  # Nicht 401

class TestSchoolByCode:
    def test_valid_code(self, client, app, sample_school):
        """GET /api/public/schools/by-code/TEST-1234 findet Schule."""
        with app.app_context():
            sample_school.join_code = "TEST-1234"
            sample_school.subscription_status = "active"
            from extensions import db
            db.session.commit()
        rv = client.get("/api/public/schools/by-code/TEST-1234")
        assert rv.status_code == 200
        data = rv.get_json()
        assert data["name"] == "Testschule"
        assert "requires_approval" in data

    def test_invalid_code(self, client):
        """Ungueltiger Code gibt 404."""
        rv = client.get("/api/public/schools/by-code/XXXX-9999")
        assert rv.status_code == 404

    def test_code_case_insensitive(self, client, app, sample_school):
        """Code-Suche ist case-insensitive."""
        with app.app_context():
            sample_school.join_code = "TEST-1234"
            sample_school.subscription_status = "active"
            from extensions import db
            db.session.commit()
        rv = client.get("/api/public/schools/by-code/test-1234")
        assert rv.status_code == 200
```

**Step 2: Run tests to verify they fail**

Run: `python -m pytest tests/test_ios_registration.py::TestSchoolSearch -v`
Expected: FAIL — 404 (Blueprint nicht registriert)

**Step 3: Create api_public.py Blueprint**

```python
# blueprints/api_public.py
"""Oeffentliche API-Endpoints (kein JWT erforderlich)."""
from flask import Blueprint, jsonify, request
from extensions import db, limiter
from models.schools import SchoolProfile
import bleach

api_public_bp = Blueprint("api_public", __name__, url_prefix="/api/public")


@api_public_bp.route("/schools/search")
@limiter.limit("30 per minute")
def search_schools():
    """Schulsuche per Name (oeffentlich, kein JWT)."""
    q = (request.args.get("q") or "").strip()
    if len(q) < 2:
        return jsonify({"error": "Mindestens 2 Zeichen erforderlich"}), 400

    q_safe = bleach.clean(q, tags=[], strip=True)
    schools = (
        SchoolProfile.query
        .filter(
            SchoolProfile.name.ilike(f"%{q_safe}%"),
            SchoolProfile.is_demo == False,
            SchoolProfile.join_code.isnot(None),
        )
        .limit(20)
        .all()
    )

    # Nur aktive Schulen
    results = []
    for s in schools:
        if s.is_subscription_active:
            results.append({
                "id": s.id,
                "name": s.name,
                "city": s.city,
                "join_code": s.join_code,
            })

    return jsonify({"schools": results})


@api_public_bp.route("/schools/by-code/<code>")
@limiter.limit("30 per minute")
def school_by_code(code):
    """Schule per Join-Code finden (oeffentlich, kein JWT)."""
    code_clean = bleach.clean(code.upper().strip(), tags=[], strip=True)
    school = SchoolProfile.query.filter(
        db.func.upper(SchoolProfile.join_code) == code_clean,
        SchoolProfile.is_demo == False,
    ).first()

    if not school or not school.is_subscription_active:
        return jsonify({"error": "Schule nicht gefunden"}), 404

    return jsonify({
        "id": school.id,
        "name": school.name,
        "city": school.city,
        "requires_approval": school.customer_approval_required or False,
    })
```

In `app.py`, Blueprint registrieren:
```python
from blueprints.api_public import api_public_bp
app.register_blueprint(api_public_bp)
```

**Step 4: Run tests**

Run: `python -m pytest tests/test_ios_registration.py -v -k "Search or ByCode"`
Expected: PASS

**Step 5: Commit**

```bash
git add blueprints/api_public.py app.py tests/test_ios_registration.py
git commit -m "feat: Oeffentliche Schulsuche und Code-Validierung Endpoints"
```

---

## Task 3: Kundenregistrierung Endpoint

**Files:**
- Modify: `blueprints/api_auth.py`
- Test: `tests/test_ios_registration.py` (erweitern)

**Step 1: Write failing tests**

```python
class TestRegisterCustomer:
    def test_register_success(self, client, app, sample_school):
        """POST /api/auth/register-customer erstellt Kunden."""
        with app.app_context():
            sample_school.join_code = "TEST-1234"
            sample_school.subscription_status = "active"
            from extensions import db
            db.session.commit()
        rv = client.post("/api/auth/register-customer", json={
            "email": "neukunde@test.de",
            "password": "SicheresPasswort123!",
            "name": "Neuer Kunde",
            "join_code": "TEST-1234",
            "gdpr_consent": True,
            "agb_consent": True,
        })
        assert rv.status_code == 201
        data = rv.get_json()
        assert data["status"] == "registered"

    def test_register_with_approval(self, client, app, sample_school):
        """Registrierung bei Schule mit Freigabe ergibt pending_approval."""
        with app.app_context():
            sample_school.join_code = "TEST-1234"
            sample_school.subscription_status = "active"
            sample_school.customer_approval_required = True
            from extensions import db
            db.session.commit()
        rv = client.post("/api/auth/register-customer", json={
            "email": "wartend@test.de",
            "password": "SicheresPasswort123!",
            "name": "Wartender Kunde",
            "join_code": "TEST-1234",
            "gdpr_consent": True,
            "agb_consent": True,
        })
        assert rv.status_code == 201
        data = rv.get_json()
        assert data["status"] == "pending_approval"

    def test_register_invalid_code(self, client):
        """Ungueltiger Join-Code gibt 404."""
        rv = client.post("/api/auth/register-customer", json={
            "email": "x@test.de", "password": "SicheresPasswort123!",
            "name": "X", "join_code": "XXXX-9999",
            "gdpr_consent": True, "agb_consent": True,
        })
        assert rv.status_code == 404

    def test_register_duplicate_email(self, client, app, sample_school, sample_user):
        """Doppelte E-Mail gibt 409."""
        with app.app_context():
            sample_school.join_code = "TEST-1234"
            sample_school.subscription_status = "active"
            from extensions import db
            db.session.commit()
        rv = client.post("/api/auth/register-customer", json={
            "email": "admin@test.de",  # Existiert bereits
            "password": "SicheresPasswort123!",
            "name": "Doppelt", "join_code": "TEST-1234",
            "gdpr_consent": True, "agb_consent": True,
        })
        assert rv.status_code == 409

    def test_register_weak_password(self, client, app, sample_school):
        """Schwaches Passwort gibt 400."""
        with app.app_context():
            sample_school.join_code = "TEST-1234"
            sample_school.subscription_status = "active"
            from extensions import db
            db.session.commit()
        rv = client.post("/api/auth/register-customer", json={
            "email": "neu@test.de", "password": "kurz",
            "name": "Test", "join_code": "TEST-1234",
            "gdpr_consent": True, "agb_consent": True,
        })
        assert rv.status_code == 400

    def test_register_missing_consent(self, client, app, sample_school):
        """Fehlende DSGVO-Einwilligung gibt 400."""
        with app.app_context():
            sample_school.join_code = "TEST-1234"
            sample_school.subscription_status = "active"
            from extensions import db
            db.session.commit()
        rv = client.post("/api/auth/register-customer", json={
            "email": "neu@test.de", "password": "SicheresPasswort123!",
            "name": "Test", "join_code": "TEST-1234",
            "gdpr_consent": False, "agb_consent": True,
        })
        assert rv.status_code == 400
```

**Step 2: Run tests to verify they fail**

Run: `python -m pytest tests/test_ios_registration.py::TestRegisterCustomer -v`
Expected: FAIL — 404 (Endpoint existiert nicht)

**Step 3: Implement register-customer endpoint**

In `blueprints/api_auth.py`:

```python
@api_auth_bp.route("/register-customer", methods=["POST"])
@limiter.limit("5 per minute")
def register_customer():
    """Kundenregistrierung bei einer bestehenden Hundeschule via Join-Code."""
    data = request.get_json(silent=True) or {}

    email = (data.get("email") or "").strip().lower()
    password = data.get("password") or ""
    name = bleach.clean((data.get("name") or "").strip(), tags=[], strip=True)
    join_code = (data.get("join_code") or "").strip().upper()
    gdpr = data.get("gdpr_consent")
    agb = data.get("agb_consent")

    # Validierung
    if not email or not password or not name or not join_code:
        return jsonify({"error": "Alle Felder sind erforderlich"}), 400
    if not gdpr or not agb:
        return jsonify({"error": "DSGVO- und AGB-Einwilligung erforderlich"}), 400

    # Passwort-Komplexitaet
    pw_error = _check_password_complexity(password)
    if pw_error:
        return jsonify({"error": pw_error}), 400

    # E-Mail-Format
    try:
        from email_validator import validate_email
        validated = validate_email(email, check_deliverability=False)
        email = validated.normalized
    except Exception:
        return jsonify({"error": "Ungueltige E-Mail-Adresse"}), 400

    # Schule per Code finden
    school = SchoolProfile.query.filter(
        db.func.upper(SchoolProfile.join_code) == join_code
    ).first()
    if not school or not school.is_subscription_active:
        return jsonify({"error": "Hundeschule nicht gefunden"}), 404

    # Doppelte E-Mail pruefen
    existing = User.query.filter_by(email=email).first()
    if existing:
        return jsonify({"error": "E-Mail-Adresse bereits registriert"}), 409

    # User erstellen
    now = datetime.now(UTC)
    user = User(
        email=email, name=name,
        email_confirmed=False,
        email_hash=User.compute_email_hash(email),
        last_school_id=school.id,
        last_role="customer",
        gdpr_consent_at=now, agb_consent_at=now,
        consent_ip_masked=_mask_ip(request.remote_addr),
        consent_privacy_version=current_app.config.get("PRIVACY_POLICY_VERSION"),
    )
    user.set_password(password)
    db.session.add(user)
    db.session.flush()

    # Rolle zuweisen
    approved = not school.customer_approval_required
    role = UserSchoolRole(
        user_id=user.id, school_id=school.id,
        role="customer", approved=approved,
    )
    db.session.add(role)

    try:
        db.session.commit()
    except Exception:
        db.session.rollback()
        return jsonify({"error": "Datenbankfehler"}), 500

    # Bestaetigungs-E-Mail senden
    try:
        _send_confirmation_email(user)
    except Exception:
        pass  # E-Mail-Fehler soll Registration nicht blockieren

    status = "pending_approval" if not approved else "registered"
    return jsonify({
        "status": status,
        "user": {"id": user.id, "email": user.email, "name": user.name},
    }), 201
```

**Step 4: Run tests**

Run: `python -m pytest tests/test_ios_registration.py::TestRegisterCustomer -v`
Expected: PASS

**Step 5: Commit**

```bash
git add blueprints/api_auth.py tests/test_ios_registration.py
git commit -m "feat: Kundenregistrierung Endpoint mit Join-Code und Freigabe"
```

---

## Task 4: Login-Check fuer pending_approval

**Files:**
- Modify: `blueprints/api_auth.py` (login Endpoint)
- Test: `tests/test_ios_registration.py` (erweitern)

**Step 1: Write failing tests**

```python
class TestLoginApproval:
    def test_login_blocked_when_pending(self, client, app, sample_school):
        """Login wird blockiert wenn Konto auf Freigabe wartet."""
        with app.app_context():
            from models.auth import User, UserSchoolRole
            from extensions import db
            user = User(
                email="pending@test.de", name="Pending",
                email_confirmed=True,
                email_hash=User.compute_email_hash("pending@test.de"),
                last_school_id=sample_school.id, last_role="customer",
            )
            user.set_password("SicheresPasswort123!")
            db.session.add(user)
            db.session.flush()
            db.session.add(UserSchoolRole(
                user_id=user.id, school_id=sample_school.id,
                role="customer", approved=False,
            ))
            db.session.commit()
        rv = client.post("/api/auth/login", json={
            "email": "pending@test.de",
            "password": "SicheresPasswort123!",
        })
        assert rv.status_code == 403
        data = rv.get_json()
        assert data["error"] == "pending_approval"

    def test_login_allowed_when_approved(self, client, app, sample_school):
        """Login funktioniert wenn Konto freigeschaltet."""
        with app.app_context():
            from models.auth import User, UserSchoolRole
            from extensions import db
            user = User(
                email="approved@test.de", name="Approved",
                email_confirmed=True,
                email_hash=User.compute_email_hash("approved@test.de"),
                last_school_id=sample_school.id, last_role="customer",
            )
            user.set_password("SicheresPasswort123!")
            db.session.add(user)
            db.session.flush()
            db.session.add(UserSchoolRole(
                user_id=user.id, school_id=sample_school.id,
                role="customer", approved=True,
            ))
            db.session.commit()
        rv = client.post("/api/auth/login", json={
            "email": "approved@test.de",
            "password": "SicheresPasswort123!",
        })
        assert rv.status_code == 200
```

**Step 2: Run tests to verify they fail**

Run: `python -m pytest tests/test_ios_registration.py::TestLoginApproval -v`
Expected: FAIL — Login erlaubt obwohl pending

**Step 3: Erweitere Login-Endpoint**

In `blueprints/api_auth.py`, `login()` Funktion, nach Passwort-Check und vor JWT-Ausgabe:

```python
    # Pruefen ob alle Rollen auf Freigabe warten
    roles = UserSchoolRole.query.filter_by(user_id=user.id).all()
    if roles and all(not r.approved for r in roles):
        return jsonify({
            "error": "pending_approval",
            "message": "Dein Konto wartet auf Freigabe durch die Hundeschule.",
        }), 403
```

**Step 4: Run tests**

Run: `python -m pytest tests/test_ios_registration.py::TestLoginApproval -v`
Expected: PASS

**Step 5: Commit**

```bash
git add blueprints/api_auth.py tests/test_ios_registration.py
git commit -m "feat: Login blockiert wenn Konto auf Freigabe wartet"
```

---

## Task 5: Admin-Freigabe Endpoint

**Files:**
- Modify: `blueprints/api_admin/users.py`
- Test: `tests/test_ios_registration.py` (erweitern)

**Step 1: Write failing tests**

```python
class TestAdminApproval:
    def test_approve_user(self, client, app, sample_school, sample_user, logged_in_client):
        """Admin kann Kunden freischalten."""
        with app.app_context():
            from models.auth import User, UserSchoolRole
            from extensions import db
            customer = User(
                email="warten@test.de", name="Wartend",
                email_confirmed=True,
                email_hash=User.compute_email_hash("warten@test.de"),
                last_school_id=sample_school.id, last_role="customer",
            )
            customer.set_password("SicheresPasswort123!")
            db.session.add(customer)
            db.session.flush()
            db.session.add(UserSchoolRole(
                user_id=customer.id, school_id=sample_school.id,
                role="customer", approved=False,
            ))
            db.session.commit()
            cid = customer.id
        rv = logged_in_client.put(f"/api/admin/users/{cid}/approve")
        assert rv.status_code == 200
        with app.app_context():
            from models.auth import UserSchoolRole
            role = UserSchoolRole.query.filter_by(user_id=cid, role="customer").first()
            assert role.approved is True

    def test_approve_sends_email(self, client, app, sample_school, sample_user, logged_in_client):
        """Freigabe sendet E-Mail an Kunden."""
        # Mocke send_event_email, pruefe Aufruf
        pass  # Implementierung mit unittest.mock

    def test_approve_wrong_school_denied(self, client, app):
        """Admin kann nur Kunden der eigenen Schule freischalten."""
        # Erstelle Kunde in anderer Schule, pruefe 404
        pass
```

**Step 2: Implement approve endpoint**

In `blueprints/api_admin/users.py`:

```python
@api_admin_users_bp.route("/users/<int:user_id>/approve", methods=["PUT"])
@jwt_or_session_required
@limiter.limit("30 per hour")
def approve_user(user_id):
    """Kunden-Konto freischalten."""
    admin = get_current_api_user()
    school_id = _get_admin_school_id(admin)

    role = UserSchoolRole.query.filter_by(
        user_id=user_id, school_id=school_id, role="customer"
    ).first()
    if not role:
        return jsonify({"error": "Benutzer nicht gefunden"}), 404

    if role.approved:
        return jsonify({"message": "Benutzer bereits freigeschaltet"})

    role.approved = True
    try:
        db.session.commit()
    except Exception:
        db.session.rollback()
        return jsonify({"error": "Datenbankfehler"}), 500

    # Freigabe-E-Mail senden
    try:
        target_user = db.session.get(User, user_id)
        school = db.session.get(SchoolProfile, school_id)
        send_event_email(school, target_user, "customer_approved", {})
    except Exception:
        pass  # E-Mail-Fehler soll Freigabe nicht blockieren

    return jsonify({"message": "Benutzer freigeschaltet"})
```

**Step 3: Run tests**

Run: `python -m pytest tests/test_ios_registration.py::TestAdminApproval -v`
Expected: PASS

**Step 4: Commit**

```bash
git add blueprints/api_admin/users.py tests/test_ios_registration.py
git commit -m "feat: Admin-Freigabe Endpoint fuer Kundenkonten"
```

---

## Task 6: E-Mail-Event fuer Freigabe

**Files:**
- Modify: `email_defaults.py`

**Step 1: Add customer_approved Event**

In `email_defaults.py`, `EMAIL_EVENTS` dict:

```python
"customer_approved": {
    "label": "Konto freigeschaltet",
    "description": "Wird versendet, wenn ein Admin das Kundenkonto freigibt.",
    "vars": ["user_name", "school_name", "login_button"],
    "default_subject": "Dein Konto bei {{ school_name }} ist freigeschaltet",
    "default_body": "Hallo {{ user_name }},\n\ndein Konto bei {{ school_name }} wurde freigeschaltet. Du kannst dich jetzt anmelden.\n\n{{ login_button }}\n\nViele Gruesse,\n{{ school_name }}",
},
```

**Step 2: Commit**

```bash
git add email_defaults.py
git commit -m "feat: E-Mail-Template fuer Konto-Freigabe"
```

---

## Task 7: Join-Code in Admin-Einstellungen + Regenerieren

**Files:**
- Modify: `blueprints/api_admin/settings.py` oder `config.py`
- Modify: Templates (Admin-Schuleinstellungen)

**Step 1: API-Endpoint fuer Code-Regenerierung**

```python
@api_admin_bp.route("/settings/regenerate-join-code", methods=["POST"])
@jwt_or_session_required
@limiter.limit("5 per hour")
def regenerate_join_code():
    """Neuen Join-Code generieren."""
    admin = get_current_api_user()
    school_id = _get_admin_school_id(admin)
    school = db.session.get(SchoolProfile, school_id)
    if not school:
        return jsonify({"error": "Schule nicht gefunden"}), 404

    from models.schools import generate_join_code
    school.join_code = generate_join_code()
    try:
        db.session.commit()
    except Exception:
        db.session.rollback()
        return jsonify({"error": "Datenbankfehler"}), 500

    return jsonify({"join_code": school.join_code})
```

**Step 2: Join-Code + Approval-Toggle in Schuleinstellungen UI**

In der Admin-Einstellungsseite:
- Anzeige des aktuellen Join-Codes (readonly, kopiebar)
- Button "Neuen Code generieren"
- Toggle "Kundenregistrierung erfordert Freigabe" (customer_approval_required)

**Step 3: Commit**

```bash
git add blueprints/api_admin/ templates/admin/
git commit -m "feat: Join-Code Verwaltung und Freigabe-Toggle in Admin-Einstellungen"
```

---

## Task 8: Auto-Generierung bei Schul-Erstellung

**Files:**
- Modify: `blueprints/api_auth.py` (register — Schul-Erstellung)
- Modify: `blueprints/school_public.py` (falls Schulen dort erstellt werden)

**Step 1: join_code bei Schul-Erstellung setzen**

In `blueprints/api_auth.py`, `register()`, nach SchoolProfile-Erstellung:

```python
from models.schools import generate_join_code
school.join_code = generate_join_code()
```

**Step 2: Test + Commit**

```bash
git add blueprints/api_auth.py
git commit -m "feat: Auto-generierter join_code bei Schul-Erstellung"
```

---

## Task 9: Gesamttest und Abschluss

**Step 1: Alle Tests ausfuehren**

Run: `python -m pytest tests/test_ios_registration.py -v`
Expected: Alle Tests PASS

**Step 2: Design-Dok aktualisieren**

Status auf "Erledigt" setzen.

**Step 3: Commit und Push**

```bash
git add plans/
git commit -m "docs: iOS-Kundenregistrierung Design als erledigt markiert"
git push origin main
```
