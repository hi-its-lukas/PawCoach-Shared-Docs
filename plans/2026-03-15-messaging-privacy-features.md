# Messaging Privacy & Info Features — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Backend-Endpoints und Modelle für 7 fehlende Messaging-Features implementieren, die die iOS-App bereits nutzt/erwartet: Block/Unblock, Report, Clear Chat, Mute, Profile Visibility, Last Seen, Shared Courses.

**Architecture:** Neue Modelle in `models/messaging.py` (UserBlock, UserReport, ConversationMute), neues Modell `ProfileVisibility` in `models/auth.py`. Alle neuen API-Endpoints in `blueprints/api_messaging.py` (prefix `/api/messages`) bzw. `blueprints/api_auth.py` (für Visibility). Eine Alembic-Migration für alle neuen Tabellen + Felder. `last_seen` wird aus `User.last_activity_at` abgeleitet (bereits vorhanden).

**Tech Stack:** Flask, SQLAlchemy, Alembic, pytest, JWT + Session Auth

---

## Task 1: Neue Modelle — UserBlock, UserReport, ConversationMute, ProfileVisibility

**Files:**
- Modify: `models/messaging.py` (am Ende anfügen)
- Modify: `models/auth.py` (ProfileVisibility nach User-Klasse)
- Modify: `models/__init__.py` (neue Exports)

**Step 1: UserBlock, UserReport, ConversationMute in `models/messaging.py` anfügen**

```python
class UserBlock(db.Model):
    """Ein User blockiert einen anderen User (schulweit)."""

    __tablename__ = "user_block"
    __table_args__ = (db.UniqueConstraint("blocker_id", "blocked_id", "school_id"),)

    id = db.Column(db.Integer, primary_key=True)
    blocker_id = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    blocked_id = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    school_id = db.Column(db.Integer, db.ForeignKey("school_profile.id"), nullable=False)
    created_at = db.Column(db.DateTime, default=lambda: datetime.now(UTC))

    blocker = db.relationship("User", foreign_keys=[blocker_id])
    blocked = db.relationship("User", foreign_keys=[blocked_id])


class UserReport(db.Model):
    """Meldung eines Users durch einen anderen User."""

    __tablename__ = "user_report"

    id = db.Column(db.Integer, primary_key=True)
    reporter_id = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    reported_id = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    school_id = db.Column(db.Integer, db.ForeignKey("school_profile.id"), nullable=False)
    reason = db.Column(db.Text, nullable=True)
    created_at = db.Column(db.DateTime, default=lambda: datetime.now(UTC))

    reporter = db.relationship("User", foreign_keys=[reporter_id])
    reported = db.relationship("User", foreign_keys=[reported_id])


class ConversationMute(db.Model):
    """Stummschaltung einer Konversation fuer einen User."""

    __tablename__ = "conversation_mute"
    __table_args__ = (db.UniqueConstraint("conversation_id", "user_id"),)

    id = db.Column(db.Integer, primary_key=True)
    conversation_id = db.Column(db.Integer, db.ForeignKey("conversation.id"), nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    muted_until = db.Column(db.DateTime, nullable=True)  # NULL = permanent
    created_at = db.Column(db.DateTime, default=lambda: datetime.now(UTC))

    conversation = db.relationship("Conversation", foreign_keys=[conversation_id])
    user = db.relationship("User", foreign_keys=[user_id])
```

**Step 2: ProfileVisibility in `models/auth.py` anfügen (nach User-Klasse)**

```python
class ProfileVisibility(db.Model):
    """Sichtbarkeitseinstellungen fuer ein Benutzerprofil."""

    __tablename__ = "profile_visibility"

    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False, unique=True)
    discoverable = db.Column(db.Boolean, default=True, nullable=False)
    last_name_full = db.Column(db.Boolean, default=True, nullable=False)
    profile_photo_visibility = db.Column(db.String(20), default="all", nullable=False)  # all|trainers|nobody
    dogs_visibility = db.Column(db.String(20), default="trainers", nullable=False)
    dog_owners_visibility = db.Column(db.String(20), default="trainers", nullable=False)
    phone_visibility = db.Column(db.String(20), default="nobody", nullable=False)
    last_seen_visibility = db.Column(db.String(20), default="all", nullable=False)

    user = db.relationship("User", backref=db.backref("profile_visibility", uselist=False))
```

**Step 3: `models/__init__.py` aktualisieren**

Hinzufügen zu imports und `__all__`:
- `from models.messaging import UserBlock, UserReport, ConversationMute`
- `from models.auth import ProfileVisibility`
- In `__all__`: `"UserBlock"`, `"UserReport"`, `"ConversationMute"`, `"ProfileVisibility"`

**Step 4: Commit**

```bash
git add models/messaging.py models/auth.py models/__init__.py
git commit -m "feat: Modelle für Block, Report, Mute, ProfileVisibility"
```

---

## Task 2: Alembic-Migration

**Files:**
- Create: `migrations/versions/xxxx_add_block_report_mute_visibility.py`

**Step 1: Migration generieren**

```bash
cd /home/azure-ubuntu/Dog-School-Manager
flask db migrate -m "add user_block, user_report, conversation_mute, profile_visibility"
```

**Step 2: Migration prüfen und idempotent machen**

Die generierte Migration muss das `_table_exists`/`_column_exists`-Pattern verwenden:

```python
def _table_exists(table):
    bind = op.get_bind()
    insp = sa.inspect(bind)
    return table in insp.get_table_names()

def upgrade():
    if not _table_exists("user_block"):
        op.create_table("user_block", ...)
    if not _table_exists("user_report"):
        op.create_table("user_report", ...)
    if not _table_exists("conversation_mute"):
        op.create_table("conversation_mute", ...)
    if not _table_exists("profile_visibility"):
        op.create_table("profile_visibility", ...)
```

**Step 3: Migration ausführen**

```bash
flask db upgrade
```

**Step 4: Commit**

```bash
git add migrations/
git commit -m "feat: Migration für Block/Report/Mute/Visibility-Tabellen"
```

---

## Task 3: Block/Unblock API-Endpoints

**Files:**
- Modify: `blueprints/api_messaging.py` (am Ende, vor letztem Endpoint)
- Test: `tests/test_messaging_block_report.py`

**iOS erwartet:**
- `POST /api/messages/block/<user_id>` → 204
- `DELETE /api/messages/block/<user_id>` → 204

**Step 1: Tests schreiben**

```python
# tests/test_messaging_block_report.py
import json
import pytest
from models import Conversation, ConversationMember, User, UserSchoolRole, db
from models.messaging import UserBlock, UserReport, ConversationMute


class TestBlockUnblock:
    def _setup_users(self, app, sample_school):
        """Zwei User erstellen: Admin + Kunde."""
        from flask import current_app
        customer = User(
            email="block-target@test.de", name="Block Target",
            email_confirmed=True, email_hash=User.compute_email_hash("block-target@test.de"),
            last_school_id=sample_school.id, last_role="customer",
            consent_privacy_version=current_app.config.get("PRIVACY_POLICY_VERSION"),
        )
        customer.set_password("TestPasswort123!")
        db.session.add(customer)
        db.session.flush()
        db.session.add(UserSchoolRole(user_id=customer.id, school_id=sample_school.id, role="customer"))
        db.session.commit()
        return customer

    def test_block_user(self, app, sample_school, sample_user, logged_in_client):
        target = self._setup_users(app, sample_school)
        resp = logged_in_client.post(f"/api/messages/block/{target.id}")
        assert resp.status_code == 204
        assert UserBlock.query.filter_by(blocker_id=sample_user.id, blocked_id=target.id).first() is not None

    def test_block_user_duplicate(self, app, sample_school, sample_user, logged_in_client):
        target = self._setup_users(app, sample_school)
        logged_in_client.post(f"/api/messages/block/{target.id}")
        resp = logged_in_client.post(f"/api/messages/block/{target.id}")
        assert resp.status_code == 204  # Idempotent

    def test_block_self_forbidden(self, app, sample_school, sample_user, logged_in_client):
        resp = logged_in_client.post(f"/api/messages/block/{sample_user.id}")
        assert resp.status_code == 400

    def test_unblock_user(self, app, sample_school, sample_user, logged_in_client):
        target = self._setup_users(app, sample_school)
        logged_in_client.post(f"/api/messages/block/{target.id}")
        resp = logged_in_client.delete(f"/api/messages/block/{target.id}")
        assert resp.status_code == 204
        assert UserBlock.query.filter_by(blocker_id=sample_user.id, blocked_id=target.id).first() is None

    def test_unblock_nonexistent(self, app, sample_school, sample_user, logged_in_client):
        resp = logged_in_client.delete("/api/messages/block/99999")
        assert resp.status_code == 204  # Idempotent
```

**Step 2: Tests laufen lassen → FAIL erwartet**

```bash
pytest tests/test_messaging_block_report.py -v
```

**Step 3: Endpoints implementieren in `blueprints/api_messaging.py`**

Am Ende der Datei anfügen:

```python
# ---------------------------------------------------------------------------
# Block / Unblock
# ---------------------------------------------------------------------------

@api_messaging_bp.route("/block/<int:user_id>", methods=["POST"])
@jwt_or_session_required
@limiter.limit("30 per hour")
def block_user(user_id):
    """User blockieren."""
    user = get_current_api_user()
    sid = get_api_school_id()
    if user.id == user_id:
        return jsonify({"error": "Sich selbst blockieren nicht möglich"}), 400
    target = User.query.get(user_id)
    if not target:
        return jsonify({"error": "User nicht gefunden"}), 404
    existing = UserBlock.query.filter_by(blocker_id=user.id, blocked_id=user_id, school_id=sid).first()
    if not existing:
        db.session.add(UserBlock(blocker_id=user.id, blocked_id=user_id, school_id=sid))
        db.session.commit()
    return "", 204


@api_messaging_bp.route("/block/<int:user_id>", methods=["DELETE"])
@jwt_or_session_required
@limiter.limit("30 per hour")
def unblock_user(user_id):
    """Block aufheben."""
    user = get_current_api_user()
    sid = get_api_school_id()
    block = UserBlock.query.filter_by(blocker_id=user.id, blocked_id=user_id, school_id=sid).first()
    if block:
        db.session.delete(block)
        db.session.commit()
    return "", 204
```

Import `UserBlock` am Anfang der Datei hinzufügen.

**Step 4: Tests laufen lassen → PASS erwartet**

```bash
pytest tests/test_messaging_block_report.py::TestBlockUnblock -v
```

**Step 5: Block-Check in Nachrichtenversand einbauen**

In `blueprints/api_messaging.py`, Funktion `send_message` (POST `/<int:conversation_id>`): Vor dem Speichern prüfen, ob der Absender von einem Mitglied der Konversation geblockt wurde (bei Direktnachrichten).

```python
# In send_message, nach Konversations-Check:
if conv.conv_type == "direct":
    other_member = ConversationMember.query.filter(
        ConversationMember.conversation_id == conv.id,
        ConversationMember.user_id != user.id
    ).first()
    if other_member:
        is_blocked = UserBlock.query.filter_by(
            blocker_id=other_member.user_id, blocked_id=user.id, school_id=sid
        ).first()
        if is_blocked:
            return jsonify({"error": "Nachricht kann nicht gesendet werden"}), 403
```

**Step 6: Commit**

```bash
git add blueprints/api_messaging.py tests/test_messaging_block_report.py
git commit -m "feat: Block/Unblock API-Endpoints mit Tests"
```

---

## Task 4: Report API-Endpoint

**Files:**
- Modify: `blueprints/api_messaging.py`
- Modify: `tests/test_messaging_block_report.py`

**iOS erwartet:**
- `POST /api/messages/report/<user_id>` mit optionalem `{"reason": "..."}` → 204

**Step 1: Tests hinzufügen**

```python
class TestReport:
    def _setup_target(self, app, sample_school):
        from flask import current_app
        user = User(
            email="report-target@test.de", name="Report Target",
            email_confirmed=True, email_hash=User.compute_email_hash("report-target@test.de"),
            last_school_id=sample_school.id, last_role="customer",
            consent_privacy_version=current_app.config.get("PRIVACY_POLICY_VERSION"),
        )
        user.set_password("TestPasswort123!")
        db.session.add(user)
        db.session.flush()
        db.session.add(UserSchoolRole(user_id=user.id, school_id=sample_school.id, role="customer"))
        db.session.commit()
        return user

    def test_report_user(self, app, sample_school, sample_user, logged_in_client):
        target = self._setup_target(app, sample_school)
        resp = logged_in_client.post(
            f"/api/messages/report/{target.id}",
            data=json.dumps({"reason": "Spam"}),
            content_type="application/json",
        )
        assert resp.status_code == 204
        report = UserReport.query.filter_by(reporter_id=sample_user.id, reported_id=target.id).first()
        assert report is not None
        assert report.reason == "Spam"

    def test_report_user_no_reason(self, app, sample_school, sample_user, logged_in_client):
        target = self._setup_target(app, sample_school)
        resp = logged_in_client.post(f"/api/messages/report/{target.id}")
        assert resp.status_code == 204

    def test_report_self_forbidden(self, app, sample_school, sample_user, logged_in_client):
        resp = logged_in_client.post(f"/api/messages/report/{sample_user.id}")
        assert resp.status_code == 400
```

**Step 2: Endpoint implementieren**

```python
@api_messaging_bp.route("/report/<int:user_id>", methods=["POST"])
@jwt_or_session_required
@limiter.limit("10 per hour")
def report_user(user_id):
    """User melden."""
    user = get_current_api_user()
    sid = get_api_school_id()
    if user.id == user_id:
        return jsonify({"error": "Sich selbst melden nicht möglich"}), 400
    target = User.query.get(user_id)
    if not target:
        return jsonify({"error": "User nicht gefunden"}), 404
    reason = None
    if request.is_json and request.json:
        reason = request.json.get("reason", "")[:1000]  # Max 1000 Zeichen
    db.session.add(UserReport(reporter_id=user.id, reported_id=user_id, school_id=sid, reason=reason))
    db.session.commit()
    current_app.logger.warning("User %d reported user %d (school %d): %s", user.id, user_id, sid, reason or "")
    return "", 204
```

**Step 3: Tests laufen lassen → PASS**

```bash
pytest tests/test_messaging_block_report.py::TestReport -v
```

**Step 4: Commit**

```bash
git add blueprints/api_messaging.py tests/test_messaging_block_report.py
git commit -m "feat: Report-User API-Endpoint mit Tests"
```

---

## Task 5: Clear Chat API-Endpoint

**Files:**
- Modify: `blueprints/api_messaging.py`
- Modify: `tests/test_messaging_block_report.py`

**iOS erwartet:**
- `DELETE /api/messages/<conversation_id>/clear` → 204

Logik: Für den aufrufenden User werden alle Nachrichten als gelöscht markiert (MessageDeletion-Einträge), nicht physisch gelöscht.

**Step 1: Tests hinzufügen**

```python
class TestClearChat:
    def _setup_conv(self, sample_school, sample_user):
        conv = Conversation(school_id=sample_school.id, conv_type="direct",
                           created_by_id=sample_user.id)
        db.session.add(conv)
        db.session.flush()
        db.session.add(ConversationMember(conversation_id=conv.id, user_id=sample_user.id))
        from models.messaging import Message
        for i in range(3):
            db.session.add(Message(conversation_id=conv.id, sender_id=sample_user.id,
                                   body=f"Msg {i}"))
        db.session.commit()
        return conv

    def test_clear_chat(self, app, sample_school, sample_user, logged_in_client):
        conv = self._setup_conv(sample_school, sample_user)
        resp = logged_in_client.delete(f"/api/messages/{conv.id}/clear")
        assert resp.status_code == 204
        from models.messaging import MessageDeletion, Message
        msgs = Message.query.filter_by(conversation_id=conv.id).all()
        for m in msgs:
            assert MessageDeletion.query.filter_by(message_id=m.id, user_id=sample_user.id).first() is not None

    def test_clear_chat_nonmember(self, app, sample_school, sample_user, logged_in_client):
        conv = Conversation(school_id=sample_school.id, conv_type="direct",
                           created_by_id=sample_user.id)
        db.session.add(conv)
        db.session.commit()
        # User ist kein Mitglied
        resp = logged_in_client.delete(f"/api/messages/{conv.id}/clear")
        assert resp.status_code == 403
```

**Step 2: Endpoint implementieren**

```python
@api_messaging_bp.route("/<int:conv_id>/clear", methods=["DELETE"])
@jwt_or_session_required
@limiter.limit("10 per hour")
def clear_chat(conv_id):
    """Alle Nachrichten einer Konversation fuer den aktuellen User loeschen."""
    user = get_current_api_user()
    sid = get_api_school_id()
    conv = Conversation.query.filter_by(id=conv_id, school_id=sid).first()
    if not conv:
        return jsonify({"error": "Nicht gefunden"}), 404
    member = ConversationMember.query.filter_by(conversation_id=conv_id, user_id=user.id).first()
    if not member:
        return jsonify({"error": "Kein Mitglied"}), 403
    msgs = Message.query.filter_by(conversation_id=conv_id).all()
    for msg in msgs:
        existing = MessageDeletion.query.filter_by(message_id=msg.id, user_id=user.id).first()
        if not existing:
            db.session.add(MessageDeletion(message_id=msg.id, user_id=user.id))
    db.session.commit()
    return "", 204
```

**Step 3: Tests laufen lassen → PASS**

```bash
pytest tests/test_messaging_block_report.py::TestClearChat -v
```

**Step 4: Commit**

```bash
git add blueprints/api_messaging.py tests/test_messaging_block_report.py
git commit -m "feat: Clear-Chat API-Endpoint mit Tests"
```

---

## Task 6: Mute/Unmute API-Endpoints

**Files:**
- Modify: `blueprints/api_messaging.py`
- Modify: `tests/test_messaging_block_report.py`

**Endpoints (für Backend-Sync, iOS speichert bisher nur lokal):**
- `POST /api/messages/<conv_id>/mute` → 204
- `DELETE /api/messages/<conv_id>/mute` → 204
- `GET /api/messages/muted` → `{"muted": [conv_id, ...]}`

**Step 1: Tests**

```python
class TestMute:
    def _setup_conv(self, sample_school, sample_user):
        conv = Conversation(school_id=sample_school.id, conv_type="direct",
                           created_by_id=sample_user.id)
        db.session.add(conv)
        db.session.flush()
        db.session.add(ConversationMember(conversation_id=conv.id, user_id=sample_user.id))
        db.session.commit()
        return conv

    def test_mute_conversation(self, app, sample_school, sample_user, logged_in_client):
        conv = self._setup_conv(sample_school, sample_user)
        resp = logged_in_client.post(f"/api/messages/{conv.id}/mute")
        assert resp.status_code == 204

    def test_unmute_conversation(self, app, sample_school, sample_user, logged_in_client):
        conv = self._setup_conv(sample_school, sample_user)
        logged_in_client.post(f"/api/messages/{conv.id}/mute")
        resp = logged_in_client.delete(f"/api/messages/{conv.id}/mute")
        assert resp.status_code == 204

    def test_muted_list(self, app, sample_school, sample_user, logged_in_client):
        conv = self._setup_conv(sample_school, sample_user)
        logged_in_client.post(f"/api/messages/{conv.id}/mute")
        resp = logged_in_client.get("/api/messages/muted")
        assert resp.status_code == 200
        data = resp.get_json()
        assert conv.id in data["muted"]
```

**Step 2: Endpoints implementieren**

```python
@api_messaging_bp.route("/<int:conv_id>/mute", methods=["POST"])
@jwt_or_session_required
@limiter.limit("30 per hour")
def mute_conversation(conv_id):
    """Konversation stummschalten."""
    user = get_current_api_user()
    sid = get_api_school_id()
    conv = Conversation.query.filter_by(id=conv_id, school_id=sid).first()
    if not conv:
        return jsonify({"error": "Nicht gefunden"}), 404
    member = ConversationMember.query.filter_by(conversation_id=conv_id, user_id=user.id).first()
    if not member:
        return jsonify({"error": "Kein Mitglied"}), 403
    existing = ConversationMute.query.filter_by(conversation_id=conv_id, user_id=user.id).first()
    if not existing:
        db.session.add(ConversationMute(conversation_id=conv_id, user_id=user.id))
        db.session.commit()
    return "", 204


@api_messaging_bp.route("/<int:conv_id>/mute", methods=["DELETE"])
@jwt_or_session_required
@limiter.limit("30 per hour")
def unmute_conversation(conv_id):
    """Stummschaltung aufheben."""
    user = get_current_api_user()
    sid = get_api_school_id()
    mute = ConversationMute.query.filter_by(conversation_id=conv_id, user_id=user.id).first()
    if mute:
        db.session.delete(mute)
        db.session.commit()
    return "", 204


@api_messaging_bp.route("/muted")
@jwt_or_session_required
@limiter.limit("60 per hour")
def list_muted():
    """Liste aller stummgeschalteten Konversationen."""
    user = get_current_api_user()
    mutes = ConversationMute.query.filter_by(user_id=user.id).all()
    return jsonify({"muted": [m.conversation_id for m in mutes]})
```

**Step 3: Tests laufen lassen → PASS**

```bash
pytest tests/test_messaging_block_report.py::TestMute -v
```

**Step 4: Commit**

```bash
git add blueprints/api_messaging.py tests/test_messaging_block_report.py
git commit -m "feat: Mute/Unmute API-Endpoints mit Tests"
```

---

## Task 7: Profile Visibility API-Endpoints

**Files:**
- Modify: `blueprints/api_auth.py` (GET + PATCH `/api/auth/profile/visibility`)
- Create: `tests/test_profile_visibility.py`

**iOS erwartet:**
- `GET /api/auth/profile/visibility` → JSON mit allen Visibility-Feldern
- `PATCH /api/auth/profile/visibility` → aktualisiert Felder

**Step 1: api_auth.py prüfen** — wo die Datei endet und welcher Blueprint/Prefix genutzt wird.

**Step 2: Tests schreiben**

```python
# tests/test_profile_visibility.py
import json
import pytest
from models import User, db
from models.auth import ProfileVisibility


class TestProfileVisibility:
    def test_get_default_visibility(self, app, sample_school, sample_user, logged_in_client):
        resp = logged_in_client.get("/api/auth/profile/visibility")
        assert resp.status_code == 200
        data = resp.get_json()
        assert data["discoverable"] is True
        assert data["last_name_full"] is True
        assert data["profile_photo_visibility"] == "all"
        assert data["last_seen_visibility"] == "all"
        assert data["phone_visibility"] == "nobody"

    def test_patch_visibility(self, app, sample_school, sample_user, logged_in_client):
        resp = logged_in_client.patch(
            "/api/auth/profile/visibility",
            data=json.dumps({"discoverable": False, "phone_visibility": "trainers"}),
            content_type="application/json",
        )
        assert resp.status_code == 200
        pv = ProfileVisibility.query.filter_by(user_id=sample_user.id).first()
        assert pv.discoverable is False
        assert pv.phone_visibility == "trainers"

    def test_patch_invalid_value(self, app, sample_school, sample_user, logged_in_client):
        resp = logged_in_client.patch(
            "/api/auth/profile/visibility",
            data=json.dumps({"phone_visibility": "INVALID"}),
            content_type="application/json",
        )
        assert resp.status_code == 400
```

**Step 3: Endpoints in `blueprints/api_auth.py` implementieren**

```python
VISIBILITY_FIELDS_BOOL = {"discoverable", "last_name_full"}
VISIBILITY_FIELDS_CHOICE = {
    "profile_photo_visibility", "dogs_visibility",
    "dog_owners_visibility", "phone_visibility", "last_seen_visibility",
}
VISIBILITY_CHOICES = {"all", "trainers", "nobody"}

# camelCase → snake_case Mapping (iOS sendet camelCase)
_CAMEL_TO_SNAKE = {
    "lastNameFull": "last_name_full",
    "profilePhotoVisibility": "profile_photo_visibility",
    "dogsVisibility": "dogs_visibility",
    "dogOwnersVisibility": "dog_owners_visibility",
    "phoneVisibility": "phone_visibility",
    "lastSeenVisibility": "last_seen_visibility",
}


@api_auth_bp.route("/profile/visibility", methods=["GET"])
@jwt_or_session_required
@limiter.limit("60 per hour")
def get_visibility():
    """Profil-Sichtbarkeitseinstellungen abrufen."""
    user = get_current_api_user()
    pv = ProfileVisibility.query.filter_by(user_id=user.id).first()
    if not pv:
        pv = ProfileVisibility(user_id=user.id)
        db.session.add(pv)
        db.session.commit()
    return jsonify({
        "discoverable": pv.discoverable,
        "last_name_full": pv.last_name_full,
        "profile_photo_visibility": pv.profile_photo_visibility,
        "dogs_visibility": pv.dogs_visibility,
        "dog_owners_visibility": pv.dog_owners_visibility,
        "phone_visibility": pv.phone_visibility,
        "last_seen_visibility": pv.last_seen_visibility,
    })


@api_auth_bp.route("/profile/visibility", methods=["PATCH"])
@jwt_or_session_required
@limiter.limit("30 per hour")
def update_visibility():
    """Profil-Sichtbarkeitseinstellungen aktualisieren."""
    user = get_current_api_user()
    data = request.get_json(force=True)
    # camelCase-Keys normalisieren
    normalized = {}
    for k, v in data.items():
        normalized[_CAMEL_TO_SNAKE.get(k, k)] = v

    pv = ProfileVisibility.query.filter_by(user_id=user.id).first()
    if not pv:
        pv = ProfileVisibility(user_id=user.id)
        db.session.add(pv)

    for field in VISIBILITY_FIELDS_BOOL:
        if field in normalized:
            setattr(pv, field, bool(normalized[field]))

    for field in VISIBILITY_FIELDS_CHOICE:
        if field in normalized:
            val = normalized[field]
            if val not in VISIBILITY_CHOICES:
                return jsonify({"error": f"Ungültiger Wert für {field}: {val}"}), 400
            setattr(pv, field, val)

    db.session.commit()
    return jsonify({"status": "ok"})
```

Import `ProfileVisibility` und `request` am Anfang der Datei hinzufügen.

**Step 4: Tests laufen lassen → PASS**

```bash
pytest tests/test_profile_visibility.py -v
```

**Step 5: Commit**

```bash
git add blueprints/api_auth.py tests/test_profile_visibility.py
git commit -m "feat: Profile-Visibility GET/PATCH Endpoints mit Tests"
```

---

## Task 8: Last Seen in Konversations-API-Responses

**Files:**
- Modify: `blueprints/api_messaging.py` (in `_serialize_conversation` und GET `/<int:conversation_id>`)
- Modify: `blueprints/messaging.py` (in Web-API Konversations-Detail-Response)

**Logik:** `last_seen` wird aus `User.last_activity_at` des anderen Teilnehmers (bei Direktnachrichten) abgeleitet. Die Sichtbarkeit wird über `ProfileVisibility.last_seen_visibility` gesteuert.

**Step 1: In `_serialize_conversation()` (api_messaging.py ~Zeile 90+)**

Feld `last_seen` hinzufügen — nur bei Direktnachrichten, nur wenn Visibility es erlaubt:

```python
# In _serialize_conversation, nach den bestehenden Feldern:
last_seen = None
if conv.conv_type == "direct":
    other = ConversationMember.query.filter(
        ConversationMember.conversation_id == conv.id,
        ConversationMember.user_id != user_id
    ).first()
    if other and other.user:
        # Visibility-Check
        pv = ProfileVisibility.query.filter_by(user_id=other.user_id).first()
        show = True
        if pv and pv.last_seen_visibility == "nobody":
            show = False
        elif pv and pv.last_seen_visibility == "trainers":
            role = get_api_role()
            show = role in ("admin", "trainer")
        if show and other.user.last_activity_at:
            last_seen = other.user.last_activity_at.isoformat()
```

**Step 2: Auch in `messaging.py` Web-Detail-Response (Zeile ~977-993)**

Gleiches Feld `"last_seen"` zum JSON hinzufügen.

**Step 3: Commit**

```bash
git add blueprints/api_messaging.py blueprints/messaging.py
git commit -m "feat: last_seen Feld in Konversations-API-Responses"
```

---

## Task 9: Shared Courses API-Endpoint

**Files:**
- Modify: `blueprints/api_messaging.py`
- Test: in `tests/test_messaging_block_report.py`

**iOS erwartet:**
- `GET /api/messages/shared-courses/<user_id>` → `{"courses": [{"id": int, "title": str, "schedule": str?}]}`

**Step 1: Tests**

```python
class TestSharedCourses:
    def test_shared_courses_empty(self, app, sample_school, sample_user, logged_in_client):
        resp = logged_in_client.get(f"/api/messages/shared-courses/{sample_user.id}")
        assert resp.status_code == 200
        data = resp.get_json()
        assert data["courses"] == []
```

**Step 2: Endpoint implementieren**

```python
@api_messaging_bp.route("/shared-courses/<int:user_id>")
@jwt_or_session_required
@limiter.limit("60 per hour")
def shared_courses(user_id):
    """Gemeinsame Kurse zwischen aktuellem User und Ziel-User."""
    user = get_current_api_user()
    sid = get_api_school_id()
    # Kurse finden, in denen beide User eingeschrieben sind
    from models import CourseEnrollment, Course
    my_courses = {e.course_id for e in CourseEnrollment.query.filter_by(user_id=user.id).all()}
    their_courses = {e.course_id for e in CourseEnrollment.query.filter_by(user_id=user_id).all()}
    shared_ids = my_courses & their_courses
    courses = Course.query.filter(Course.id.in_(shared_ids), Course.school_id == sid).all() if shared_ids else []
    return jsonify({
        "courses": [{"id": c.id, "title": c.title, "schedule": getattr(c, "schedule", None)} for c in courses]
    })
```

**Step 3: Tests laufen lassen → PASS**

```bash
pytest tests/test_messaging_block_report.py::TestSharedCourses -v
```

**Step 4: Commit**

```bash
git add blueprints/api_messaging.py tests/test_messaging_block_report.py
git commit -m "feat: Shared-Courses API-Endpoint"
```

---

## Task 10: `is_blocked` und `is_muted` in Konversations-Responses

**Files:**
- Modify: `blueprints/api_messaging.py` (`_serialize_conversation` und Detail-Response)

In beiden Serialisierungen hinzufügen:

```python
# is_muted
muted = ConversationMute.query.filter_by(conversation_id=conv.id, user_id=user_id).first()
# ... im return dict:
"is_muted": muted is not None,
```

Für `is_blocked` (nur bei Direktnachrichten):
```python
is_blocked = False
if conv.conv_type == "direct":
    other = ConversationMember.query.filter(
        ConversationMember.conversation_id == conv.id,
        ConversationMember.user_id != user_id
    ).first()
    if other:
        is_blocked = UserBlock.query.filter_by(
            blocker_id=user_id, blocked_id=other.user_id, school_id=sid
        ).first() is not None
# ... im return dict:
"is_blocked": is_blocked,
```

**Commit:**

```bash
git add blueprints/api_messaging.py
git commit -m "feat: is_blocked und is_muted in Konversations-Responses"
```

---

## Task 11: Alle Tests laufen lassen + finaler Review

**Step 1: Alle neuen Tests laufen lassen**

```bash
pytest tests/test_messaging_block_report.py tests/test_profile_visibility.py -v
```

**Step 2: Bestehende Messaging-Tests prüfen (keine Regressions)**

```bash
pytest tests/test_api_messaging.py tests/test_api_messaging_contacts.py tests/test_messaging_archive.py tests/test_message_edit_delete.py tests/test_poll_api.py -v
```

**Step 3: Gesamte Test-Suite**

```bash
pytest tests/ -v --tb=short
```

**Step 4: Finaler Commit falls nötig**

```bash
git add -A
git commit -m "test: alle Privacy/Info-Feature-Tests bestanden"
```

---

## Zusammenfassung der neuen Endpoints

| Endpoint | Methode | Beschreibung |
|----------|---------|--------------|
| `/api/messages/block/<user_id>` | POST | User blockieren |
| `/api/messages/block/<user_id>` | DELETE | Block aufheben |
| `/api/messages/report/<user_id>` | POST | User melden |
| `/api/messages/<conv_id>/clear` | DELETE | Chat für mich leeren |
| `/api/messages/<conv_id>/mute` | POST | Konversation stumm |
| `/api/messages/<conv_id>/mute` | DELETE | Stummschaltung aufheben |
| `/api/messages/muted` | GET | Liste stummgeschalteter IDs |
| `/api/auth/profile/visibility` | GET | Sichtbarkeitseinstellungen |
| `/api/auth/profile/visibility` | PATCH | Sichtbarkeit aktualisieren |
| `/api/messages/shared-courses/<user_id>` | GET | Gemeinsame Kurse |

Neue Response-Felder in Konversations-APIs:
- `last_seen` (ISO8601, nur bei Direktnachrichten, respektiert Visibility)
- `is_blocked` (Boolean)
- `is_muted` (Boolean)
