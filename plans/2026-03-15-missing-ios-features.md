# Fehlende iOS-Features Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Alle noch fehlenden iOS-App-Features im Backend und Web-Frontend implementieren.

**Architecture:** 8 unabhaengige Features: (1) delivered_at/read_at auf Message + Zustellstatus + Nachrichteninfo-Endpoint, (2) Chat-Suche Backend+Web, (3) Medien-Zaehler-Endpoint, (4) Sichtbarkeitsfilter in Kontakt-Serialisierung, (5) Web-UI fuer Sichtbarkeitseinstellungen, (6) Halter eigener Hunde in Kontaktliste, (7) Mute-Check bei Push (vorbereitet), (8) Contacts Visibility auf Profildetails anwenden.

**Tech Stack:** Flask, SQLAlchemy, Alembic, Jinja2, Vanilla JS, Socket.IO, pytest

---

### Task 1: delivered_at / read_at auf Message-Model + Migration

**Files:**
- Modify: `models/messaging.py:78-104` (Message-Klasse)
- Create: `migrations/versions/m1n2o3p4q5r6_add_delivered_read_timestamps.py`
- Test: `tests/test_delivery_status.py`

**Step 1: Migration erstellen**

```python
# migrations/versions/m1n2o3p4q5r6_add_delivered_read_timestamps.py
"""Add delivered_at and read_at to message."""
from alembic import op
import sqlalchemy as sa

revision = "m1n2o3p4q5r6"
down_revision = "l3m4n5o6p7q8"  # letzte existierende Migration
branch_labels = None
depends_on = None

def _column_exists(table, column):
    from sqlalchemy import inspect
    bind = op.get_bind()
    insp = inspect(bind)
    cols = [c["name"] for c in insp.get_columns(table)]
    return column in cols

def upgrade():
    if not _column_exists("message", "delivered_at"):
        op.add_column("message", sa.Column("delivered_at", sa.DateTime(), nullable=True))
    if not _column_exists("message", "read_at"):
        op.add_column("message", sa.Column("read_at", sa.DateTime(), nullable=True))

def downgrade():
    op.drop_column("message", "read_at")
    op.drop_column("message", "delivered_at")
```

**Step 2: Model-Felder hinzufuegen**

In `models/messaging.py`, nach `deleted_by_name` (Zeile 97):
```python
delivered_at = db.Column(db.DateTime, nullable=True)
read_at = db.Column(db.DateTime, nullable=True)
```

**Step 3: Serialisierung aktualisieren**

In `blueprints/api_messaging.py`, `_serialize_message()` erweitern um:
```python
"delivered_at": msg.delivered_at.isoformat() if msg.delivered_at else None,
"read_at": msg.read_at.isoformat() if msg.read_at else None,
"delivery_status": _get_delivery_status(msg),
```

Neue Hilfsfunktion:
```python
def _get_delivery_status(msg):
    if msg.read_at:
        return "read"
    if msg.delivered_at:
        return "delivered"
    return "sent"
```

**Step 4: Mark-as-read Endpoint erweitern**

In `blueprints/api_messaging.py`, beim mark_as_read Endpoint: Alle ungelesenen Nachrichten des anderen Users als `read_at = now` setzen.

```python
# Im bestehenden mark_as_read: nach last_read_at Update
Message.query.filter(
    Message.conversation_id == conv_id,
    Message.sender_id != user.id,
    Message.read_at.is_(None),
).update({"read_at": datetime.now(UTC)})
```

**Step 5: Deliver-Endpoint hinzufuegen**

```python
@api_messaging_bp.route("/<int:conv_id>/delivered", methods=["POST"])
@jwt_or_session_required
@limiter.limit("120 per hour")
def mark_delivered(conv_id):
    """Alle Nachrichten als zugestellt markieren."""
    user = get_current_api_user()
    member = ConversationMember.query.filter_by(
        conversation_id=conv_id, user_id=user.id
    ).first()
    if not member:
        return jsonify({"error": "Kein Mitglied"}), 403
    Message.query.filter(
        Message.conversation_id == conv_id,
        Message.sender_id != user.id,
        Message.delivered_at.is_(None),
    ).update({"delivered_at": datetime.now(UTC)})
    db.session.commit()
    return "", 204
```

**Step 6: Nachrichteninfo-Endpoint**

```python
@api_messaging_bp.route("/info/<int:message_id>")
@jwt_or_session_required
@limiter.limit("60 per hour")
def message_info(message_id):
    """Zustelldetails einer eigenen Nachricht."""
    user = get_current_api_user()
    msg = db.session.get(Message, message_id)
    if not msg or msg.sender_id != user.id:
        return jsonify({"error": "Nicht gefunden"}), 404
    return jsonify({
        "id": msg.id,
        "delivered_at": msg.delivered_at.isoformat() if msg.delivered_at else None,
        "read_at": msg.read_at.isoformat() if msg.read_at else None,
        "delivery_status": _get_delivery_status(msg),
    })
```

**Step 7: Tests schreiben und ausfuehren**

```python
# tests/test_delivery_status.py
class TestDeliveryStatus:
    def test_new_message_has_sent_status(self, ...)
    def test_mark_delivered(self, ...)
    def test_mark_read_sets_read_at(self, ...)
    def test_message_info_own_message(self, ...)
    def test_message_info_other_message_forbidden(self, ...)
    def test_delivery_status_in_message_response(self, ...)
```

Run: `python3 -m pytest tests/test_delivery_status.py -v`

**Step 8: Commit**
```bash
git add models/messaging.py migrations/versions/m1n2o3p4q5r6_add_delivered_read_timestamps.py blueprints/api_messaging.py tests/test_delivery_status.py
git commit -m "feat: delivered_at/read_at auf Message + Zustellstatus + Nachrichteninfo"
```

---

### Task 2: Chat-Suche (Nachrichten-Inhalt)

**Files:**
- Modify: `blueprints/api_messaging.py` (neuer Endpoint)
- Modify: `blueprints/messaging.py` (Web-Route)
- Modify: `templates/messages/_chat_js.html` (Such-UI)
- Test: `tests/test_chat_search.py`

**Step 1: API-Endpoint fuer Nachrichtensuche**

```python
@api_messaging_bp.route("/<int:conv_id>/search")
@jwt_or_session_required
@limiter.limit("30 per hour")
def search_messages(conv_id):
    """Nachrichten in einer Konversation durchsuchen."""
    user = get_current_api_user()
    member = ConversationMember.query.filter_by(
        conversation_id=conv_id, user_id=user.id
    ).first()
    if not member:
        return jsonify({"error": "Kein Mitglied"}), 403
    q = (request.args.get("q") or "").strip()
    if not q or len(q) < 2:
        return jsonify({"results": []})
    results = Message.query.filter(
        Message.conversation_id == conv_id,
        Message.body.ilike(f"%{q}%"),
        Message.deleted_at.is_(None),
    ).order_by(Message.created_at.desc()).limit(50).all()
    return jsonify({
        "results": [_serialize_message(m) for m in results]
    })
```

**Step 2: Web-Route (messaging.py)**

```python
# In messaging.py, neuer API-Endpoint:
@messaging_bp.route("/api/<int:conv_id>/search")
@login_required
@school_required
def api_search_messages(conv_id):
    # Analog zum API-Endpoint
```

**Step 3: Web-UI Such-Button und Panel**

Im Chat-Header: Such-Icon-Button. Beim Klick: Such-Input-Feld einblenden.
Ergebnisse als scrollbare Liste, Klick springt zur Nachricht.

**Step 4: Tests**

```python
class TestChatSearch:
    def test_search_finds_matching_messages(self, ...)
    def test_search_excludes_deleted(self, ...)
    def test_search_requires_membership(self, ...)
    def test_search_min_length(self, ...)
```

**Step 5: Commit**
```bash
git commit -m "feat: Chat-Suche fuer Nachrichten-Inhalt (Backend + Web-UI)"
```

---

### Task 3: Medien/Links/Doks-Zaehler

**Files:**
- Modify: `blueprints/api_messaging.py` (neuer Endpoint)
- Test: `tests/test_media_counter.py`

**Step 1: Endpoint**

```python
@api_messaging_bp.route("/<int:conv_id>/media")
@jwt_or_session_required
@limiter.limit("60 per hour")
def shared_media(conv_id):
    """Geteilte Medien einer Konversation zaehlen und auflisten."""
    user = get_current_api_user()
    member = ConversationMember.query.filter_by(
        conversation_id=conv_id, user_id=user.id
    ).first()
    if not member:
        return jsonify({"error": "Kein Mitglied"}), 403
    from models.messaging import MessageAttachment
    attachments = (
        MessageAttachment.query
        .join(Message, MessageAttachment.message_id == Message.id)
        .filter(Message.conversation_id == conv_id, Message.deleted_at.is_(None))
        .order_by(MessageAttachment.created_at.desc())
        .all()
    )
    media = []
    links = []
    docs = []
    for a in attachments:
        item = {"id": a.id, "file_url": a.file_url, "file_name": a.file_name,
                "file_type": a.file_type, "file_size": a.file_size,
                "created_at": a.created_at.isoformat() if a.created_at else None}
        if a.file_type and a.file_type.startswith("image/"):
            media.append(item)
        elif a.file_type and a.file_type.startswith(("video/", "audio/")):
            media.append(item)
        else:
            docs.append(item)
    # Links aus Nachrichtentexten extrahieren (einfach)
    import re
    url_pattern = re.compile(r'https?://\S+')
    link_msgs = Message.query.filter(
        Message.conversation_id == conv_id,
        Message.deleted_at.is_(None),
        Message.body.op('~')('https?://'),
    ).all()
    for m in link_msgs:
        for url in url_pattern.findall(m.body):
            links.append({"url": url, "message_id": m.id,
                          "created_at": m.created_at.isoformat()})
    return jsonify({
        "media_count": len(media), "docs_count": len(docs), "links_count": len(links),
        "media": media[:50], "docs": docs[:50], "links": links[:50],
    })
```

**Step 2: Tests + Commit**

---

### Task 4: Sichtbarkeitsfilter in Kontakt-Serialisierung

**Files:**
- Modify: `blueprints/api_messaging.py` (contacts endpoint + _serialize_conversation)
- Test: `tests/test_contacts_discoverable.py` (erweitern)

**Step 1: Kontakt-Name nach last_name_full filtern**

Im contacts endpoint, bei der Serialisierung:
```python
def _apply_visibility_to_contact(contact_dict, viewer_role, contact_user_id):
    """ProfileVisibility auf Kontakt-Details anwenden."""
    vis = ProfileVisibility.query.filter_by(user_id=contact_user_id).first()
    if not vis or viewer_role in ("admin", "trainer"):
        return contact_dict  # Staff sieht immer alles
    # last_name_full
    if not vis.last_name_full and contact_dict.get("name"):
        parts = contact_dict["name"].rsplit(" ", 1)
        if len(parts) == 2:
            contact_dict["name"] = f"{parts[0]} {parts[1][0]}."
    return contact_dict
```

**Step 2: Tests + Commit**

---

### Task 5: Web-UI fuer Sichtbarkeitseinstellungen

**Files:**
- Modify: `templates/auth/privacy_settings.html`
- Modify: `blueprints/auth.py` (Route-Handler erweitern)
- Test: manuell (Template-Rendering)

**Step 1: Formular in privacy_settings.html einfuegen**

Neuer Abschnitt "Profil-Sichtbarkeit" mit Toggle/Select-Feldern fuer alle 7 Einstellungen. CSP-konform (keine inline-Handler).

**Step 2: Route erweitern**

Im auth.py privacy_settings Route: ProfileVisibility laden und an Template uebergeben. POST-Handler zum Speichern.

**Step 3: Commit**

---

### Task 6: Halter eigener Hunde in Kontaktliste

**Files:**
- Modify: `blueprints/api_messaging.py` (contacts endpoint)
- Test: `tests/test_contacts_discoverable.py` (erweitern)

**Step 1: Co-Owner zur Kontaktliste hinzufuegen**

Fuer Kunden: Alle Halter der eigenen Hunde als Kontakte einfuegen (auch wenn nicht discoverable).

```python
# Nach dem school_roles Loop, vor der Platform-Admin-Logik:
if school_id and not is_staff:
    for dog in user.dogs:
        for owner in dog.owners:
            if owner.id != user.id and owner.id not in contacts_map:
                sr = UserSchoolRole.query.filter_by(
                    user_id=owner.id, school_id=school_id
                ).first()
                if sr:
                    contacts_map[owner.id] = {
                        "id": owner.id, "name": owner.name,
                        "role": sr.role, "roles": [sr.role],
                        "group": None, "school_name": None,
                    }
```

**Step 2: Tests + Commit**

---

### Task 7: Mute-Check vorbereiten (fuer kuenftige Push-Notifications)

**Files:**
- Modify: `push_notifications.py`

**Step 1: Hilfsfunktion**

```python
def is_conversation_muted(user_id, conversation_id):
    """Prueft ob eine Konversation fuer einen User stummgeschaltet ist."""
    from models.messaging import ConversationMute
    return ConversationMute.query.filter_by(
        user_id=user_id, conversation_id=conversation_id
    ).first() is not None
```

Diese Funktion wird aufgerufen wenn Push-Notifications fuer Nachrichten implementiert werden.

**Step 2: Commit**

---

### Task 8: Web-Frontend Nachrichten-Suche UI

**Files:**
- Modify: `templates/messages/_chat_js.html`
- Modify: `templates/messages/_chat_panel.html`

**Step 1: Such-Button im Chat-Header**

Such-Icon neben den Chat-Actions.

**Step 2: Such-Overlay**

Eingabefeld + Ergebnis-Liste. Bei Klick auf Ergebnis: Scroll zur Nachricht.

**Step 3: Commit**
