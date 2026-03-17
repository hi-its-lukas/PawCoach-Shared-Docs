# Backend: Fehlende Realtime-Events fuer iOS

**Datum:** 2026-03-17
**Prioritaet:** HOCH
**Betrifft:** socketio_handlers.py, api_messaging.py

---

## Uebersicht

Die iOS-App (Build 30) hat alle Event-Handler implementiert, aber das Backend emittiert drei kritische Events noch nicht. Ohne diese Events funktioniert kein Live-Update auf iOS.

| Event | iOS-Handler | Backend emittiert? | Auswirkung ohne |
|-------|------------|-------------------|-----------------|
| `new_message` | ✅ ChatStore.handleNewMessage | ✅ Ja | — |
| `message_receipts_updated` | ✅ ChatStore.handleReceiptsUpdated | ❌ NEIN | Keine Haekchen-Updates |
| `presence_update` | ✅ ChatStore.handlePresenceUpdate | ❌ NEIN | Kein "online"/"zuletzt online" |
| `unread_update` | ✅ ChatStore.loadUnreadCount | ✅ Ja | — |
| `user_typing` | ✅ ChatStore.didReceiveTypingIndicator | ✅ Ja | — |
| `message_edited` | ✅ ChatStore.handleMessageEdited | ✅ Ja | — |
| `message_deleted` | ✅ ChatStore.handleMessageDeleted | ✅ Ja | — |

---

## 1. HOCH: `message_receipts_updated` Event

### Wann emittieren

Wenn ein User eine Konversation oeffnet und `POST /api/messages/{conv_id}/delivered` oder `POST /api/messages/{conv_id}/read` aufruft.

### Wer empfaengt

Der **Sender** der betroffenen Nachrichten (ueber `room=f"user_{sender_id}"`).

### Payload (aus Unified Modernization Plan §2.3)

```json
{
  "conversation_id": 45,
  "updates": [
    {
      "message_id": 123,
      "delivered_at": "2026-03-17T12:00:04+00:00",
      "read_at": null,
      "delivery_status": "delivered"
    },
    {
      "message_id": 124,
      "delivered_at": "2026-03-17T12:00:04+00:00",
      "read_at": "2026-03-17T12:00:05+00:00",
      "delivery_status": "read"
    }
  ]
}
```

### Implementierung

In den REST-Endpoints `mark_delivered` und `mark_read` (in `api_messaging.py`), NACH dem DB-Update:

```python
# In mark_delivered / mark_read endpoint, nach db.session.commit():

# Alle betroffenen Nachrichten sammeln
updated_messages = Message.query.filter(
    Message.conversation_id == conv_id,
    Message.sender_id != user.id,  # Nur fremde Nachrichten
    Message.delivered_at != None   # Nur gerade aktualisierte
).all()

if updated_messages:
    updates = []
    for msg in updated_messages:
        updates.append({
            "message_id": msg.id,
            "delivered_at": _iso(msg.delivered_at),
            "read_at": _iso(msg.read_at) if msg.read_at else None,
            "delivery_status": "read" if msg.read_at else "delivered"
        })

    # An den SENDER emittieren (nicht an den Leser!)
    sender_ids = set(msg.sender_id for msg in updated_messages)
    for sender_id in sender_ids:
        socketio.emit(
            "message_receipts_updated",
            {"conversation_id": conv_id, "updates": [u for u in updates if ...]},
            room=f"user_{sender_id}"
        )
```

### Auswirkung auf iOS

Wenn der Empfaenger den Chat oeffnet → `markDelivered` + `markAsRead` → Backend emittiert `message_receipts_updated` → Sender sieht sofort die Doppelhaken (zugestellt) bzw. gruene Doppelhaken (gelesen) + Uhrzeiten.

---

## 2. HOCH: `presence_update` Event

### Wann emittieren

1. Bei `connect` (User ist online)
2. Bei `disconnect` (User ist offline)
3. Optional: Heartbeat alle 30s

### Wer empfaengt

Alle User die eine Direkt-Konversation mit diesem User haben (ueber `room=f"user_{other_id}"`).

### Payload (aus Unified Modernization Plan §2.3)

```json
{
  "user_id": 8,
  "is_online": true,
  "last_seen": "2026-03-17T12:00:00+00:00",
  "updated_at": "2026-03-17T12:00:00+00:00"
}
```

### Implementierung

```python
# In handle_connect(), NACH join_room:
def _emit_presence(user_id, is_online):
    """Presence-Update an alle Direkt-Chat-Partner emittieren."""
    # Alle direkten Konversationspartner finden
    partner_ids = db.session.query(ConversationMember.user_id).join(Conversation).filter(
        Conversation.conv_type == "direct",
        Conversation.id.in_(
            db.session.query(ConversationMember.conversation_id).filter(
                ConversationMember.user_id == user_id
            )
        ),
        ConversationMember.user_id != user_id
    ).distinct().all()

    user = db.session.get(User, user_id)
    payload = {
        "user_id": user_id,
        "is_online": is_online,
        "last_seen": _iso(user.last_activity_at) if user else None,
        "updated_at": _iso(datetime.now(UTC))
    }

    for (pid,) in partner_ids:
        socketio.emit("presence_update", payload, room=f"user_{pid}")


@socketio.on("connect")
def handle_connect():
    # ... bestehender Auth-Code ...
    join_room(f"user_{user.id}")
    _emit_presence(user.id, True)


@socketio.on("disconnect")
def handle_disconnect():
    if current_user.is_authenticated:
        _emit_presence(current_user.id, False)
```

### Auswirkung auf iOS

Chat-Header zeigt "online" oder "zuletzt online um HH:mm" in Echtzeit — ohne Polling.

---

## 3. MITTEL: `new_message` im User-Room

### Aktueller Stand

`new_message` wird in den Konversations-Room emittiert (`room=f"conv_{conv_id}"`). iOS joined den Konversations-Room nur wenn der Chat OFFEN ist (`joinConversation(convId)` in `openChat()`).

### Problem

Wenn der Chat NICHT offen ist, bekommt iOS kein `new_message` Event → kein In-App-Banner, kein Badge-Update.

### Loesung

`new_message` ZUSAETZLICH in den User-Room emittieren (`room=f"user_{member_id}"`). Das passiert bereits fuer `unread_update` — gleiche Logik fuer `new_message`:

```python
# NACH emit an conv-Room:
for member_id in member_ids:
    if member_id != sender_id:
        socketio.emit("new_message", msg_data, room=f"user_{member_id}")
```

### Auswirkung

iOS zeigt In-App-Banner auch wenn der Chat nicht offen ist.

---

## Checkliste

- [x] `message_receipts_updated` in mark_delivered und mark_read emittieren ✅
- [x] `presence_update` bei connect und disconnect emittieren ✅
- [x] `new_message` zusaetzlich in User-Rooms emittieren ✅
- [ ] Testen: iOS Sender sieht Doppelhaken live wenn Empfaenger Chat oeffnet (iOS-Entwickler)
- [ ] Testen: iOS zeigt "online" wenn anderer User verbunden (iOS-Entwickler)
- [ ] Testen: iOS zeigt In-App-Banner wenn Chat nicht offen (iOS-Entwickler)
