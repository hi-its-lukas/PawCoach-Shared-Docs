# Backend: 2 verbleibende Fixes fuer iOS-Paritaet

**Datum:** 2026-03-17
**Prioritaet:** HOCH
**Status:** ERLEDIGT ✅

---

## 1. HOCH: `created_at` in Socket.IO Handlern noch deutsches Format

### Problem

Die REST-API sendet `created_at` korrekt als ISO 8601 (`msg.created_at.isoformat()`). Aber die **Socket.IO Event-Handler** in `socketio_handlers.py` nutzen noch `strftime("%d.%m.%Y %H:%M")`.

### Betroffene Stellen

```
socketio_handlers.py, Zeile 193:
    "created_at": msg.created_at.strftime("%d.%m.%Y %H:%M"),

socketio_handlers.py, Zeile 537:
    "created_at": lm.created_at.strftime("%d.%m.%Y %H:%M"),
```

### Fix

```python
# Zeile 193 (new_message Event):
"created_at": msg.created_at.isoformat(),

# Zeile 537 (conv_updated Event):
"created_at": lm.created_at.isoformat() if lm else None,
```

### Auswirkung ohne Fix

iOS parst das deutsche Format als Fallback (funktioniert), aber:
- Inkonsistenz zwischen REST (ISO 8601) und Socket.IO (deutsch)
- Andere zukuenftige Clients (Android) werden daran scheitern
- Timezone-Information geht verloren

---

## 2. KRITISCH: `message_edited` und `message_deleted` Socket.IO Events werden nie emittet

### Problem

Das Backend hat die Funktionen `notify_message_edited()` und `notify_message_deleted()` in `socketio_handlers.py` (Zeilen 498-520) definiert, aber sie werden **nirgends aufgerufen**.

Die REST-API Endpoints fuer Edit und Delete fuehren die DB-Aenderungen durch, emittieren aber KEINE Socket.IO Events.

### Betroffene Stellen

```
blueprints/api_messaging.py:

1. PUT /<message_id>/edit (ca. Zeile 400-410):
   → Nachricht wird bearbeitet, aber kein emit("message_edited", ...)

2. DELETE /<message_id> (ca. Zeile 430-440):
   → Nachricht wird geloescht, aber kein emit("message_deleted", ...)
```

### Fix

Nach der DB-Aenderung in beiden Endpoints die Socket.IO Events emittieren:

```python
# In der edit-Route, NACH db.session.commit():
try:
    from socketio_handlers import notify_message_edited
    notify_message_edited(msg.conversation_id, msg.id, msg.body, msg.edited_at.isoformat())
except Exception:
    pass  # Socket.IO optional

# In der delete-Route, NACH db.session.commit():
try:
    from socketio_handlers import notify_message_deleted
    notify_message_deleted(msg.conversation_id, msg.id, user.name)
except Exception:
    pass  # Socket.IO optional
```

Falls `notify_message_edited` / `notify_message_deleted` nicht existieren, hier der Code:

```python
def notify_message_edited(conv_id, message_id, body, edited_at):
    """Benachrichtigt alle Mitglieder ueber eine bearbeitete Nachricht."""
    socketio.emit(
        "message_edited",
        {"message_id": message_id, "body": body, "edited_at": edited_at},
        room=f"conv_{conv_id}",
    )

def notify_message_deleted(conv_id, message_id, deleted_by_name):
    """Benachrichtigt alle Mitglieder ueber eine geloeschte Nachricht."""
    socketio.emit(
        "message_deleted",
        {"message_id": message_id, "deleted_by_name": deleted_by_name},
        room=f"conv_{conv_id}",
    )
```

### Auswirkung ohne Fix

Wenn User A eine Nachricht bearbeitet oder loescht, sehen User B und C die Aenderung **nicht in Echtzeit**. Erst nach einem manuellen Reload der Konversation wird die Aenderung sichtbar.

---

## Checkliste

- [x] Fix 1: `socketio_handlers.py` → `msg.created_at.isoformat()` ✅ 2026-03-17
- [x] Fix 1: `socketio_handlers.py` → `lm.created_at.isoformat()` ✅ 2026-03-17
- [x] Fix 2: `api_messaging.py` Edit-Route → `notify_message_edited()` ✅ 2026-03-17
- [x] Fix 2: `api_messaging.py` Delete-Route → `notify_message_deleted()` ✅ 2026-03-17
- [x] Testen: 92 Messaging-Tests gruen ✅
- [ ] Testen: iOS-Chat Edit/Delete → Aenderung erscheint bei anderem User live (iOS-Entwickler)
