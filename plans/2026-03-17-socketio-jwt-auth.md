# Backend-Anforderung: JWT-Auth fuer Socket.IO

**Datum:** 2026-03-17
**Prioritaet:** KRITISCH — Blockiert iOS Echtzeit-Features
**Betrifft:** `socketio_handlers.py`, iOS WebSocketManager

---

## Problem

Die iOS-App nutzt JWT-basierte Authentifizierung (Access Token + Refresh Token). Der Socket.IO-Server (`socketio_handlers.py`) nutzt aktuell **Flask-Login Session-Auth** (`current_user.is_authenticated`). iOS hat keine Flask-Session → Socket.IO-Verbindung wird sofort getrennt.

**Ohne diesen Fix funktioniert auf iOS NICHTS in Echtzeit:**
- Keine Live-Nachrichten (Polling als Fallback)
- Kein Typing-Indikator
- Keine Live-Standort-Updates
- Keine Echtzeit-Reaktionen

---

## Loesung: JWT in Socket.IO Connect-Handler

### Was der iOS-Client sendet

```
Socket.IO Connect URL: https://app.pawcoach.de
Connect-Params: {"token": "eyJhbGciOiJIUzI1NiIs..."}
Extra-Headers: {"Authorization": "Bearer eyJhbGciOiJIUzI1NiIs..."}
```

Der iOS-Client sendet den JWT Access Token sowohl als Connect-Parameter als auch als Authorization-Header.

### Was im Backend geaendert werden muss

In `socketio_handlers.py`, den `handle_connect()` anpassen:

```python
from flask import request
from auth_helpers import verify_jwt_token  # oder wo auch immer die JWT-Verifizierung liegt
import flask_login

@socketio.on("connect")
def handle_connect():
    """Authentifizierten User in seinen persoenlichen Room joinen.
    Unterstuetzt sowohl Session-Auth (Web) als auch JWT-Auth (iOS).
    """
    # 1. Versuch: Session-Auth (Web-Clients)
    if current_user.is_authenticated:
        join_room(f"user_{current_user.id}")
        return

    # 2. Versuch: JWT-Auth (iOS/Mobile-Clients)
    token = request.args.get("token")
    if not token:
        auth_header = request.headers.get("Authorization", "")
        if auth_header.startswith("Bearer "):
            token = auth_header[7:]

    if token:
        user = verify_jwt_token(token)  # Euer bestehender JWT-Decoder
        if user:
            flask_login.login_user(user)
            join_room(f"user_{user.id}")
            return

    # Keine gueltige Auth → trennen
    disconnect()
```

### Wichtig

- `verify_jwt_token(token)` muss das gleiche User-Objekt zurueckgeben das auch `/api/auth/me` nutzt
- Der User muss per `flask_login.login_user(user)` eingeloggt werden, damit `current_user` in allen weiteren Event-Handlern funktioniert
- Die Session lebt nur fuer die Dauer der Socket.IO-Verbindung
- Wenn der JWT ablaeuft, muss der iOS-Client die Socket.IO-Verbindung trennen und mit neuem Token reconnecten

### Token-Refresh Flow (iOS-seitig)

```
1. Access Token laeuft ab (nach 15 Min)
2. iOS bemerkt Token-Ablauf
3. iOS ruft POST /api/auth/refresh → neuer Access Token
4. iOS trennt Socket.IO
5. iOS verbindet Socket.IO mit neuem Token
6. iOS rejoint alle offenen Konversationen
```

---

## Web-Frontend: Keine Aenderung noetig

Das Web-Frontend nutzt weiterhin Session-Auth. Der `current_user.is_authenticated` Check greift zuerst — wenn das klappt (Web), wird der JWT-Check uebersprungen.

---

## Testen

### 1. Web (Regression)

```
- Chat oeffnen → Nachrichten senden → funktioniert wie bisher
- Typing-Indikator → funktioniert wie bisher
```

### 2. iOS (neues Feature)

```bash
# Socket.IO-Verbindung mit JWT testen
import socketio
sio = socketio.Client()
sio.connect("https://app.pawcoach.de", auth={"token": "eyJ..."})
# Erwartung: connect-Event ohne Fehler
sio.emit("join_conv", {"conversation_id": 1})
sio.emit("send_message", {"conversation_id": 1, "body": "Test"})
# Erwartung: new_message Event zurueck
```

---

## Zusaetzliche Empfehlung: created_at Format

Aktuell sendet der Socket.IO-Handler `created_at` im deutschen Format:

```python
"created_at": msg.created_at.strftime("%d.%m.%Y %H:%M"),
```

**Problem:** iOS (und jeder andere API-Client) erwartet ISO 8601. Das deutsche Format fuehrt zu Parsing-Fehlern bei der Uhrzeitanzeige.

**Empfehlung:** Auf ISO 8601 umstellen:

```python
"created_at": msg.created_at.isoformat(),
```

Dies betrifft alle `strftime("%d.%m.%Y %H:%M")` Aufrufe in:
- `socketio_handlers.py` (Zeile 193)
- `messaging.py` (Zeilen 717, 808, 812, 844)

Das Web-Frontend kann ISO 8601 genauso gut parsen (`new Date(isoString)`), und iOS/Android brauchen es zwingend.

---

## Checkliste fuer Backend-Entwickler

- [x] `handle_connect()` um JWT-Auth erweitern (Code oben) ✅ 2026-03-17
- [x] `verify_jwt_token()` Funktion finden/erstellen die JWT → User-Objekt liefert ✅ nutzt `decode_access_token()` aus `jwt_auth.py`
- [x] Testen: Web-Chat funktioniert weiterhin (Session-Auth) ✅ 92 Messaging-Tests gruen
- [ ] Testen: Socket.IO-Verbindung mit JWT-Token funktioniert (iOS-Entwickler muss testen)
- [x] `created_at` Format auf ISO 8601 umgestellt ✅ REST-API fertig — **ACHTUNG: Socket.IO Handler noch nicht!** Siehe `plans/2026-03-17-backend-remaining-fixes.md`
- [ ] Optional: CORS fuer Socket.IO konfigurieren falls noetig
