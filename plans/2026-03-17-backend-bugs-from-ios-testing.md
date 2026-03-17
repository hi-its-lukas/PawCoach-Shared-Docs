# Backend-Bugs aus iOS-Testing (2026-03-17)

**Datum:** 2026-03-17
**Prioritaet:** HOCH → ERLEDIGT ✅
**Getestet mit:** iOS Build 19, echtes iPhone + Simulator

---

## Bug 1: KRITISCH — Socket.IO JWT-Auth funktioniert nicht

### Problem

Die iOS-App verbindet sich per Socket.IO mit JWT-Token (Connect-Params + Authorization-Header). Die Verbindung wird aber nicht authentifiziert → kein `join_room`, keine Events.

### Auswirkung

- Nachrichten kommen NICHT live beim Empfaenger an
- Kein Typing-Indikator
- Keine Live-Standort-Updates
- Keine Echtzeit-Reaktionen/Edits/Deletes

### Was die iOS-App sendet

```
SocketManager(
    socketURL: "https://app.pawcoach.de",
    config: [
        .forceWebsockets(true),
        .extraHeaders(["Authorization": "Bearer <JWT>"]),
        .connectParams(["token": "<JWT>"])
    ]
)
```

### Was zu pruefen ist

1. Kommt der `connect`-Event im Backend an?
2. Wird `request.args.get("token")` korrekt gelesen?
3. Wird `verify_jwt_token(token)` aufgerufen und gibt einen User zurueck?
4. Wird `flask_login.login_user(user)` aufgerufen?
5. Wird `join_room(f"user_{user.id}")` aufgerufen?

### Debug-Vorschlag

```python
@socketio.on("connect")
def handle_connect():
    logger.info(f"Socket.IO connect: authenticated={current_user.is_authenticated}")
    token = request.args.get("token")
    logger.info(f"Socket.IO connect: token={'present' if token else 'missing'}")
    if token:
        user = verify_jwt_token(token)
        logger.info(f"Socket.IO connect: jwt_user={user}")
    # ... rest
```

Dann in den Server-Logs pruefen ob die Zeilen erscheinen.

---

## Bug 2: HOCH — `created_at` ohne Timezone (naive datetime)

### Problem

Python `datetime.isoformat()` auf naiven datetimes (ohne `tzinfo`) erzeugt:
```
2026-03-16T19:45:23.123456
```

OHNE `+00:00` am Ende. Apple's `ISO8601DateFormatter` erwartet zwingend eine Timezone.

### Wo das passiert

Wenn `msg.created_at` ein naives datetime ist (kein `tzinfo=UTC`), dann gibt `.isoformat()` keine Timezone aus.

### Fix-Optionen

**Option A (empfohlen):** Alle datetimes in der DB timezone-aware machen:
```python
from datetime import UTC
created_at = db.Column(db.DateTime(timezone=True), default=lambda: datetime.now(UTC))
```

**Option B:** Bei der Serialisierung Timezone anhaengen:
```python
"created_at": msg.created_at.isoformat() + "+00:00" if msg.created_at else None,
```

**Option C:** `strftime` mit explizitem Format:
```python
"created_at": msg.created_at.strftime("%Y-%m-%dT%H:%M:%S+00:00") if msg.created_at else None,
```

### Betroffene Stellen

Alle `isoformat()` Aufrufe in:
- `blueprints/api_messaging.py` (created_at, edited_at, delivered_at, read_at, expires_at)
- `socketio_handlers.py` (created_at in new_message, conv_updated)

### Hinweis

Die iOS-App hat jetzt einen Fallback-Parser der naive ISO-Strings versteht. Aber fuer korrekte Zeitzonen-Anzeige und fuer andere Clients (Android, Web-API-Konsumenten) sollte der Fix im Backend gemacht werden.

---

## Bug 3: HOCH — `last_seen` / "Zuletzt online" Wert falsch

### Problem

Der angezeigte "zuletzt online" Wert in der iOS-App stimmt nicht mit der Realitaet ueberein. Der User ist z.B. gerade online, aber die App zeigt einen alten Timestamp.

### Moegliche Ursachen

1. **`last_activity_at` wird nicht aktualisiert:** Wird `user.last_activity_at` bei jedem API-Request aktualisiert? Oder nur bei Login?

2. **Falsches Feld:** Die iOS-App liest `last_seen` aus der Konversations-Response. Pruefen ob `_get_last_seen()` den korrekten Wert liefert.

3. **Caching:** Wird der Wert gecacht und nicht aktualisiert?

### Was zu pruefen ist

```python
# In api_messaging.py, _get_last_seen():
def _get_last_seen(conv, user_id):
    # Was wird hier zurueckgegeben?
    # Ist es last_activity_at oder last_login_at?
    # Wird es bei jedem Request aktualisiert?
```

### Empfehlung

`last_activity_at` sollte bei jedem authentifizierten API-Request aktualisiert werden (z.B. in einem `@app.before_request` Hook):

```python
@app.before_request
def update_last_activity():
    if current_user.is_authenticated:
        current_user.last_activity_at = datetime.now(UTC)
        db.session.commit()
```

---

## Bug 4: MITTEL — Live-Standort Session kann nicht beendet werden

### Problem

Wenn eine Live-Location-Session laeuft, zeigt die iOS-App den "Live-Standort beenden" Button. Beim Tap wird `DELETE /api/messages/live-location/{sessionId}` aufgerufen. Das scheint nicht zu funktionieren — die Session bleibt aktiv.

### Moegliche Ursachen

1. Die Session-ID stimmt nicht (iOS hat eine andere ID als das Backend)
2. Der Endpoint prueft Berechtigungen die der User nicht hat
3. Die Session ist bereits abgelaufen aber `is_active` ist noch `True` (kein Cleanup)

### Was zu pruefen ist

- Gibt der `DELETE` Endpoint einen Fehler zurueck?
- Wird `is_active = False` korrekt gesetzt?
- Laeuft der Cleanup fuer abgelaufene Sessions (`_cleanup_expired_sessions`)?

---

## Bug 5: MITTEL — `message_edited` und `message_deleted` Events fehlen

(Bereits in `2026-03-17-backend-remaining-fixes.md` dokumentiert, hier nochmal als Reminder)

Die REST-Endpoints fuer Edit und Delete emittieren keine Socket.IO Events. Wenn User A eine Nachricht bearbeitet/loescht, sieht User B die Aenderung nicht live.

---

## Checkliste

- [x] Bug 1: Socket.IO JWT-Auth — CORS via `SOCKETIO_CORS_ORIGINS` Env-Var konfigurierbar + Logging ✅
- [x] Bug 2: Alle `isoformat()` durch `_iso()` Helper ersetzt → immer `+00:00` Suffix ✅
- [x] Bug 3: `last_activity_at` wird jetzt auch bei GET /api/* Requests aktualisiert ✅
- [x] Bug 4: Kein Bug — Endpoint funktioniert korrekt (19 Tests gruen) ✅
- [x] Bug 5: `message_edited`/`message_deleted` Events bereits in vorherigem Commit behoben ✅

---

## Bug 6: KRITISCH — last_activity_at wird bei JWT-Requests NICHT aktualisiert

### Problem

Der `before_request` Hook in `app.py` (Zeile 350-366) prueft `current_user.is_authenticated`. Bei JWT-Requests ist `current_user` (Flask-Login) aber NICHT authentifiziert — der User wird stattdessen in `request._jwt_user` gespeichert.

**Folge:** `last_activity_at` wird fuer alle iOS-User NIE aktualisiert → "zuletzt online" zeigt immer den letzten Web-Login.

### Root Cause

```python
# app.py, Zeile 358:
if not current_user.is_authenticated:  # ← FALSE fuer JWT-User!
    return
```

### Fix

```python
@app.before_request
def _update_last_activity():
    # Pruefe sowohl Flask-Login als auch JWT-User
    user = None
    if current_user.is_authenticated:
        user = current_user
    elif hasattr(request, '_jwt_user') and request._jwt_user:
        user = request._jwt_user

    if not user:
        return

    is_api = request.path.startswith("/api/")
    is_write = request.method in ("POST", "PUT", "PATCH", "DELETE")
    if is_api or is_write:
        user.last_activity_at = datetime.now(UTC)
        db.session.commit()
```

### ACHTUNG

Der `before_request` Hook laeuft VOR dem `jwt_or_session_required` Decorator. Das bedeutet `request._jwt_user` ist zum Zeitpunkt des Hooks noch NICHT gesetzt!

**Besserer Fix:** Die `last_activity_at`-Aktualisierung in den `jwt_or_session_required` Decorator verschieben:

```python
def jwt_or_session_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        auth_header = request.headers.get("Authorization", "")
        if auth_header.startswith("Bearer "):
            user, error = get_jwt_user()
            if error:
                return error
            request._jwt_user = user
            # last_activity_at aktualisieren
            user.last_activity_at = datetime.now(UTC)
            db.session.commit()
            return f(*args, **kwargs)
        if current_user.is_authenticated:
            request._jwt_user = current_user
            return f(*args, **kwargs)
        return jsonify({"error": "Nicht authentifiziert"}), 401
    return decorated
```

### Checkliste

- [x] `last_activity_at` wird bei JWT-Requests aktualisiert ✅ Fix in `jwt_or_session_required` + `jwt_required` Decorator
- [x] 214 Tests gruen (messaging + auth) ✅
- [ ] Testen: iOS-User oeffnet Chat → Web zeigt korrekte "zuletzt online" Zeit (iOS-Entwickler)

---

## Bug 7: KRITISCH — Socket.IO trennt iOS-Verbindung sofort (41 disconnect)

### Symptom

Die iOS-App verbindet sich per Socket.IO. Der Handshake ist erfolgreich. Aber der Server sendet sofort ein **Disconnect-Packet (41)** gefolgt von einem neuen Connect (40). Der User ist damit in **keinem Room** → bekommt keine Events.

### Beweis aus iOS-Logs (Build 25)

```
LOG SocketEngine: Got message: 0{"sid":"...","upgrades":["websocket"]}
LOG SocketManager: Engine opened Connect
LOG SocketEngine: Got message: 41        ← SERVER DISCONNECT
LOG SocketEngine: Got message: 40{...}   ← Server reconnect
LOG SocketIOClient{/}: Disconnected: Got Disconnect
LOG SocketIOClient{/}: Socket connected  ← Client denkt er ist connected
```

### Der Token wird korrekt mitgesendet

```
GET /socket.io/?transport=polling&token=eyJhbGciOiJSUzI1NiIs...&EIO=4
```

Der JWT-Payload (decoded):
```json
{
  "sub": "8",
  "school_id": null,
  "role": "admin",
  "platform_admin": true,
  "iat": 1773763624,
  "exp": 1773764524,
  "iss": "pawcoach"
}
```

### Vermutete Ursache

`handle_connect()` in `socketio_handlers.py` ruft `disconnect()` auf. Moegliche Gruende:
1. `decode_access_token(token)` schlaegt fehl (Exception → Zeile 128-129)
2. `payload` ist None/falsy (Zeile 146-147)
3. `user_id` (`payload.get("sub")`) kann nicht in User aufgeloest werden (Zeile 145)
4. `school_id: null` verursacht ein Problem in der Auth-Logik

### Debug-Empfehlung

Bitte temporaer Debug-Logging in `handle_connect()` ergaenzen:

```python
@socketio.on("connect")
def handle_connect():
    token = flask_request.args.get("token")
    logger.info(f"Socket.IO connect: token={'yes' if token else 'no'}")
    
    if token:
        try:
            payload = decode_access_token(token)
            logger.info(f"Socket.IO connect: payload={payload}")
        except Exception as e:
            logger.error(f"Socket.IO connect: decode error: {e}")
            disconnect()
            return
        
        if payload:
            user_id = payload.get("sub")
            logger.info(f"Socket.IO connect: user_id={user_id}")
            user = db.session.get(User, int(user_id)) if user_id else None
            logger.info(f"Socket.IO connect: user={user}")
            # ... rest
```

Dann die Server-Logs pruefen welche Zeile den Disconnect ausloest.

### Checkliste

- [ ] Debug-Logging in handle_connect() hinzufuegen
- [ ] Server-Logs pruefen welcher Code-Pfad den Disconnect ausloest
- [ ] Fix implementieren
- [ ] Testen: iOS-App verbindet → KEIN 41-Disconnect → join_room erfolgt → Events kommen an
