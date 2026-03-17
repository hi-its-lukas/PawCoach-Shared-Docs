# iOS Realtime Bugs — Codex Review erbeten

**Datum:** 2026-03-17
**Status:** BLOCKIERT — Claude kommt nicht weiter
**Ziel:** ChatGPT Codex soll die Realtime-Architektur reviewen und Fixes vorschlagen

---

## Aktuelle Symptome (Build 34)

1. **Nachrichten kommen nicht live an** — Sender schickt via REST, Empfaenger sieht Nachricht erst nach Chat-Reload
2. **Keine Zustellhaken + Zeit** — delivered_at/read_at werden nie live aktualisiert
3. **"Fehler: Keine Verbindung zum Server"** — erscheint als Alert
4. **Online-Status zeigt nie "online"** — obwohl zwei User gleichzeitig eingeloggt sind
5. **Reaktionen (Emojis) gehen nicht live** — erst nach Reload sichtbar

## Architektur-Ueberblick

```
iOS App
├── ChatStore (@MainActor @Observable, singleton)
│   ├── conversations, messages, presenceState
│   ├── handleNewMessage(), handleReceiptsUpdated(), handlePresenceUpdate()
│   ├── openChat() / closeChat() — join/leave Socket.IO Room
│   └── sendMessage() — IMMER via REST, nie Socket.IO
│
├── WebSocketManager (@MainActor @Observable, singleton)
│   ├── SocketIOClient (socket.io-client-swift v16.1)
│   ├── connect(token:) — bei App-Start nach Auth
│   ├── delegate: SocketEventDelegate → ChatStore
│   ├── registerHandlers() → new_message, presence_update, message_receipts_updated, etc.
│   └── joinConversation() / leaveConversation() — emit join_conv/leave_conv
│
└── AppState
    ├── checkAuth() → WebSocketManager.connect() + delegate = ChatStore
    └── login() → WebSocketManager.connect() + delegate = ChatStore
```

## Bekannte Probleme im Code

### Problem 1: Doppelter connect()-Aufruf

`AppState.checkAuth()` UND `AppState.login()` rufen beide `WebSocketManager.shared.connect()` auf. Bei Login-Flow wird `checkAuth` zuerst aufgerufen (cached User), dann `login`. Das erzeugt zwei Socket.IO-Instanzen.

**Aktueller Fix:** `connect()` raeumt alte Instanz auf. ABER: `Manager is being released` → die erste Instanz wird mid-handshake gekillt → Polling-Error → Reconnect-Loop.

### Problem 2: Socket.IO Polling vs WebSocket Race

Socket.IO startet mit HTTP-Polling, upgraded dann auf WebSocket. Waehrend des Upgrades koennen Polling-Requests auf verschiedene Gunicorn-Worker treffen (trotz Redis message_queue). Das fuehrt zu `Error during long poll request`.

### Problem 3: presence_update kommt nie an

Der Handler registriert `presence_update`, aber in 30 Sekunden Monitoring wurde NIE ein Event empfangen. Moegliche Ursachen:
- Backend emittiert nicht an den richtigen Room (`user_{id}`)
- iOS-User ist nicht im `user_{id}` Room (weil `handle_connect` ihn nicht joined)
- Event-Name stimmt nicht ueberein

### Problem 4: lastSeenText liest aus presenceState

`presenceState` ist ein Dict das nur ueber Socket.IO Events befuellt wird. Wenn keine Events kommen → Dict leer → Fallback auf `conversation.lastSeen` (aus REST).

### Problem 5: "Keine Verbindung zum Server" Error

Kommt wahrscheinlich von der REST-API (nicht Socket.IO). Ein API-Call schlaegt fehl und setzt `errorMessage`. Moeglicherweise Token abgelaufen oder Netzwerk-Problem.

## Relevante Dateien

| Datei | Zweck |
|-------|-------|
| `PawCoach/Core/Network/WebSocketManager.swift` | Socket.IO Client + Event Registration |
| `PawCoach/Features/Messaging/ChatStore.swift` | State Management + Event Handlers |
| `PawCoach/App/AppState.swift` | Auth + Socket.IO Lifecycle |
| `PawCoach/Features/Messaging/MessageDetailView.swift` | Chat UI + lastSeenText |
| `PawCoach/Features/Messaging/MessagingModels.swift` | Message, Conversation Models |

## Backend Socket.IO Events (aus socketio_handlers.py)

```python
# connect: join_room(f"user_{user.id}")
# Presence: emittiert presence_update an Chat-Partner Rooms
# new_message: emittiert an conv_{conv_id} UND user_{member_id} Rooms
# message_receipts_updated: emittiert an user_{sender_id} bei mark_delivered/mark_read
```

## Was Codex pruefen soll

1. **Ist die Socket.IO Verbindung ueberhaupt stabil?** Warum kommt `Error during long poll request`?
2. **Werden Events empfangen?** Wie kann man das verifizieren?
3. **Ist der User im richtigen Room?** Wie stellt man sicher dass `join_room(f"user_{user.id}")` im Backend aufgerufen wird?
4. **Soll Polling deaktiviert werden?** `.forceWebsockets(true)` statt `.forcePolling(false)` — das ueberspringt das Polling komplett
5. **Soll der Doppel-Connect verhindert werden?** Reicht ein Guard oder braucht es eine State-Machine?
6. **Ist die Event-Registrierung korrekt?** Stimmen die Event-Namen mit dem Backend ueberein?
7. **Braucht die App einen Fallback?** Sollte es einen Timer geben der bei fehlendem Socket.IO auf REST pollt?

## Repos

- iOS: https://github.com/hi-its-lukas/PawCoach-iOS (Branch: main, Build 34)
- Backend: https://github.com/hi-its-lukas/Dog-School-Manager
- Shared Docs: https://github.com/hi-its-lukas/PawCoach-Shared-Docs
