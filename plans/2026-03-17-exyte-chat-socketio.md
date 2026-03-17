# Exyte Chat + Socket.IO Integration

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Messaging komplett auf Exyte Chat (UI) + Socket.IO-Client-Swift (Transport) umbauen. Bestehender Chat-Code bleibt als Backup.

**Architecture:** Exyte Chat liefert die SwiftUI Chat-UI (Bubbles, Input, Media). Socket.IO-Client-Swift verbindet sich mit dem Flask-SocketIO Backend fuer Echtzeit-Events. REST-API wird weiterhin fuer initiales Laden, Kontakte und Konversations-Verwaltung genutzt.

**Tech Stack:** Exyte Chat (SPM), Socket.IO-Client-Swift (SPM), SwiftUI, iOS 17+

**Backend-Auth:** Flask-SocketIO nutzt Session-basierte Auth (Flask-Login). iOS muss den Session-Cookie aus dem JWT-Login-Flow extrahieren und fuer Socket.IO mitgeben, ODER das Backend muss JWT-Auth fuer Socket.IO unterstuetzen (bevorzugt — Backend-Entwickler informieren).

---

## Vorbereitungen

### Task 0: Backup erstellen

**Step 1:** Bestehende Messaging-Views sichern

```bash
mkdir -p PawCoach/Features/Messaging/Legacy
git mv PawCoach/Features/Messaging/MessageDetailView.swift PawCoach/Features/Messaging/Legacy/
git mv PawCoach/Features/Messaging/MessageInfoSheet.swift PawCoach/Features/Messaging/Legacy/
git mv PawCoach/Features/Messaging/LocationDetailSheet.swift PawCoach/Features/Messaging/Legacy/
```

NICHT verschieben (werden weiterhin genutzt):
- MessagingModels.swift
- MessagingViewModel.swift
- MessagesListView.swift
- NewConversationView.swift
- ContactInfoView.swift
- MuteManager.swift
- GroupComposeView.swift
- BroadcastComposeView.swift
- ConversationMembersView.swift
- CreatePollSheet.swift
- LiveLocationMapView.swift

**Step 2:** Commit

```
chore: Backup bestehender Chat-Views in Legacy-Ordner
```

---

## Phase 1: Dependencies + Socket.IO

### Task 1: SPM Dependencies hinzufuegen

**Files:**
- Modify: `project.yml`

**Step 1:** Dependencies in project.yml hinzufuegen

Unter `packages:` (oder neu erstellen):
```yaml
packages:
  ExyteChat:
    url: https://github.com/exyte/Chat
    from: "2.0.0"
  SocketIO:
    url: https://github.com/socketio/socket.io-client-swift
    from: "16.1.0"
```

Unter `targets: > PawCoach: > dependencies:` hinzufuegen:
```yaml
dependencies:
  - package: ExyteChat
  - package: SocketIO
    product: SocketIO
```

**Step 2:** XcodeGen + Build

```bash
xcodegen generate
xcodebuild -project PawCoach.xcodeproj -scheme PawCoach -destination 'generic/platform=iOS' -quiet build 2>&1 | grep "error:"
```

**Step 3:** Commit

```
feat: Exyte Chat + Socket.IO SPM Dependencies
```

---

### Task 2: Socket.IO Manager neu schreiben

**Files:**
- Modify: `PawCoach/Core/Network/WebSocketManager.swift`

**Step 1:** WebSocketManager komplett ersetzen

```swift
import Foundation
import os
import SocketIO

/// Socket.IO-basierter Echtzeit-Manager fuer Chat, Typing, Reactions, Live-Location.
@MainActor @Observable
final class WebSocketManager {
    static let shared = WebSocketManager()

    private let logger = Logger(subsystem: "de.pawcoach.app", category: "SocketIO")
    private var manager: SocketManager?
    private var socket: SocketIOClient?
    private(set) var isConnected = false

    // MARK: - Callbacks

    var onNewMessage: ((Message) -> Void)?
    var onUserTyping: ((Int, String) -> Void)?
    var onReactionUpdate: ((Int, [[String: Any]]) -> Void)?
    var onMessageEdited: ((Int, String, String) -> Void)?
    var onMessageDeleted: ((Int, String) -> Void)?
    var onUnreadUpdate: (() -> Void)?
    var onConvUpdated: (([String: Any]) -> Void)?
    var onLiveLocationStarted: ((LiveLocationSession) -> Void)?
    var onLocationUpdate: ((LiveLocationPosition) -> Void)?
    var onLiveLocationEnded: ((Int, Int) -> Void)?
    var onPollUpdate: ((Int, Int) -> Void)?
    var onError: ((String) -> Void)?

    private init() {}

    // MARK: - Verbindung

    func connect(token: String) {
        guard !isConnected else { return }

        let url = AppEnvironment.baseURL
        manager = SocketManager(socketURL: url, config: [
            .log(false),
            .compress,
            .forceWebsockets(true),
            .extraHeaders(["Authorization": "Bearer \(token)"]),
            .connectParams(["token": token])
        ])

        socket = manager?.defaultSocket
        registerHandlers()
        socket?.connect()
        logger.info("Socket.IO Verbindung wird aufgebaut...")
    }

    func disconnect() {
        socket?.disconnect()
        socket = nil
        manager = nil
        isConnected = false
        logger.info("Socket.IO getrennt")
    }

    func reconnect(token: String) {
        disconnect()
        connect(token: token)
    }

    func reconnectIfNeeded() {
        guard !isConnected else { return }
        Task {
            if let token = await TokenStore.shared.accessToken() {
                connect(token: token)
            }
        }
    }

    // MARK: - Events senden

    func joinConversation(_ id: Int) {
        socket?.emit("join_conv", ["conversation_id": id])
    }

    func leaveConversation(_ id: Int) {
        socket?.emit("leave_conv", ["conversation_id": id])
    }

    func sendMessage(conversationId: Int, body: String, replyToId: Int? = nil) {
        var data: [String: Any] = [
            "conversation_id": conversationId,
            "body": body
        ]
        if let replyToId { data["reply_to_id"] = replyToId }
        socket?.emit("send_message", data)
    }

    func sendTypingIndicator(conversationId: Int) {
        socket?.emit("typing", ["conversation_id": conversationId])
    }

    func addReaction(messageId: Int, emoji: String) {
        socket?.emit("add_reaction", ["message_id": messageId, "emoji": emoji])
    }

    func removeReaction(messageId: Int, emoji: String) {
        socket?.emit("remove_reaction", ["message_id": messageId, "emoji": emoji])
    }

    func sendLocationUpdate(sessionId: Int, latitude: Double, longitude: Double, accuracy: Double) {
        socket?.emit("location_update", [
            "session_id": sessionId,
            "latitude": latitude,
            "longitude": longitude,
            "accuracy": accuracy
        ])
    }

    func sendPollResponse(pollId: Int, responses: [[String: Any]]) {
        socket?.emit("poll_response", [
            "poll_id": pollId,
            "responses": responses
        ])
    }

    // MARK: - Event Handler registrieren

    private func registerHandlers() {
        guard let socket else { return }

        socket.on(clientEvent: .connect) { [weak self] _, _ in
            Task { @MainActor in
                self?.isConnected = true
                self?.logger.info("Socket.IO verbunden")
            }
        }

        socket.on(clientEvent: .disconnect) { [weak self] _, _ in
            Task { @MainActor in
                self?.isConnected = false
                self?.logger.info("Socket.IO getrennt")
            }
        }

        socket.on(clientEvent: .error) { [weak self] data, _ in
            Task { @MainActor in
                let msg = data.first as? String ?? "Unbekannter Fehler"
                self?.logger.error("Socket.IO Fehler: \(msg)")
                self?.onError?(msg)
            }
        }

        socket.on("new_message") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any],
                  let jsonData = try? JSONSerialization.data(withJSONObject: dict),
                  let message = try? JSONDecoder.apiDecoder.decode(Message.self, from: jsonData)
            else { return }
            Task { @MainActor in self?.onNewMessage?(message) }
        }

        socket.on("user_typing") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any],
                  let userId = dict["user_id"] as? Int,
                  let userName = dict["user_name"] as? String
            else { return }
            Task { @MainActor in self?.onUserTyping?(userId, userName) }
        }

        socket.on("reaction_update") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any],
                  let messageId = dict["message_id"] as? Int,
                  let reactions = dict["reactions"] as? [[String: Any]]
            else { return }
            Task { @MainActor in self?.onReactionUpdate?(messageId, reactions) }
        }

        socket.on("message_edited") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any],
                  let messageId = dict["message_id"] as? Int,
                  let body = dict["body"] as? String,
                  let editedAt = dict["edited_at"] as? String
            else { return }
            Task { @MainActor in self?.onMessageEdited?(messageId, body, editedAt) }
        }

        socket.on("message_deleted") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any],
                  let messageId = dict["message_id"] as? Int,
                  let deletedByName = dict["deleted_by_name"] as? String
            else { return }
            Task { @MainActor in self?.onMessageDeleted?(messageId, deletedByName) }
        }

        socket.on("unread_update") { [weak self] _, _ in
            Task { @MainActor in self?.onUnreadUpdate?() }
        }

        socket.on("conv_updated") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any] else { return }
            Task { @MainActor in self?.onConvUpdated?(dict) }
        }

        socket.on("live_location_started") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any],
                  let jsonData = try? JSONSerialization.data(withJSONObject: dict),
                  let session = try? JSONDecoder.apiDecoder.decode(LiveLocationSession.self, from: jsonData)
            else { return }
            Task { @MainActor in self?.onLiveLocationStarted?(session) }
        }

        socket.on("location_update") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any],
                  let jsonData = try? JSONSerialization.data(withJSONObject: dict),
                  let position = try? JSONDecoder.apiDecoder.decode(LiveLocationPosition.self, from: jsonData)
            else { return }
            Task { @MainActor in self?.onLocationUpdate?(position) }
        }

        socket.on("live_location_ended") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any],
                  let sessionId = dict["session_id"] as? Int,
                  let convId = dict["conversation_id"] as? Int
            else { return }
            Task { @MainActor in self?.onLiveLocationEnded?(sessionId, convId) }
        }

        socket.on("poll_update") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any],
                  let pollId = dict["poll_id"] as? Int,
                  let total = dict["total_respondents"] as? Int
            else { return }
            Task { @MainActor in self?.onPollUpdate?(pollId, total) }
        }

        socket.on("error") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any],
                  let msg = dict["message"] as? String
            else { return }
            Task { @MainActor in self?.onError?(msg) }
        }
    }
}
```

**Step 2:** AppEnvironment.swift — wsURL durch baseURL ersetzen (Socket.IO macht den Upgrade selbst)

In `AppEnvironment.swift`: Die `wsURL` Property kann bleiben, aber Socket.IO nutzt die HTTP-URL. Sicherstellen dass `baseURL` als URL verfuegbar ist.

**Step 3:** AppState — WebSocket-Aufrufe aktualisieren

Alle Stellen die `WebSocketManager.shared` aufrufen pruefen. Die Handler-basierte API (onMessageReceived etc.) durch die neuen Callback-Properties ersetzen.

**Step 4:** Build + Commit

```
feat: WebSocketManager auf Socket.IO umgebaut
```

---

### Task 3: Socket.IO in MessagingViewModel integrieren

**Files:**
- Modify: `PawCoach/Features/Messaging/MessagingViewModel.swift`

**Step 1:** Socket.IO Callbacks registrieren

Im `init()` oder einer `setupSocketHandlers()` Methode:

```swift
func setupSocketHandlers() {
    let ws = WebSocketManager.shared

    ws.onNewMessage = { [weak self] message in
        // Nur hinzufuegen wenn fuer aktuelle Konversation
        self?.messages.append(message)
    }

    ws.onMessageEdited = { [weak self] messageId, body, editedAt in
        if let idx = self?.messages.firstIndex(where: { $0.id == messageId }) {
            self?.messages[idx].body = body
        }
    }

    ws.onMessageDeleted = { [weak self] messageId, _ in
        self?.messages.removeAll { $0.id == messageId }
    }

    ws.onUnreadUpdate = { [weak self] in
        Task { await self?.loadUnreadCount() }
    }

    ws.onLiveLocationStarted = { [weak self] session in
        self?.activeLiveSession = session
    }

    ws.onLocationUpdate = { [weak self] position in
        self?.handleLocationUpdate(position)
    }

    ws.onLiveLocationEnded = { [weak self] _, _ in
        self?.activeLiveSession = nil
        self?.livePositions = []
    }
}
```

**Step 2:** sendMessage via Socket.IO statt REST

```swift
func sendMessage(conversationId: Int, content: String, ...) async -> Bool {
    // Optimistisch lokale Nachricht anzeigen
    let localMsg = Message.localMessage(...)
    messages.append(localMsg)

    // Via Socket.IO senden (schneller als REST)
    if WebSocketManager.shared.isConnected {
        WebSocketManager.shared.sendMessage(
            conversationId: conversationId,
            body: content,
            replyToId: replyToId
        )
        // Server antwortet mit new_message Event → lokale Nachricht ersetzen
        return true
    }

    // Fallback auf REST wenn Socket.IO nicht verbunden
    do {
        try await api.requestVoid(.sendMessage(...))
        await loadMessages(conversationId: conversationId)
        return true
    } catch { ... }
}
```

**Step 3:** Build + Commit

```
feat: MessagingViewModel Socket.IO Integration
```

---

## Phase 2: Exyte Chat UI

### Task 4: Chat-Adapter erstellen

**Files:**
- Create: `PawCoach/Features/Messaging/ChatAdapter.swift`

Adapter der zwischen PawCoach Message-Model und Exyte Chat ExyteChat.Message konvertiert.

```swift
import ExyteChat
import Foundation

/// Konvertiert zwischen PawCoach Message und Exyte Chat Message.
enum ChatAdapter {
    static func toExyte(messages: [PawCoach.Message], currentUserId: Int) -> [ExyteChat.Message] {
        messages.compactMap { msg in
            let user = ExyteChat.User(
                id: String(msg.senderId),
                name: msg.senderName ?? "Unbekannt",
                avatarURL: nil,
                isCurrentUser: msg.senderId == currentUserId
            )

            var attachments: [ExyteChat.Attachment] = []

            // Location als Custom-Attachment
            if msg.isLocation, let lat = msg.latitude, let lng = msg.longitude {
                // Location als Text-Nachricht mit Koordinaten
            }

            // Medien
            if let urlStr = msg.mediaUrl, let url = URL(string: urlStr) {
                if msg.mediaType == "image" {
                    attachments.append(Attachment(
                        id: String(msg.id),
                        thumbnail: url,
                        full: url,
                        type: .image
                    ))
                }
            }

            let createdAt: Date
            if let iso = msg.createdAt {
                createdAt = DateFormatters.parseISO8601(iso) ?? Date()
            } else {
                createdAt = Date()
            }

            // Reply
            var replyMessage: ReplyMessage?
            if let replyId = msg.replyToId, let replyBody = msg.replyToBody {
                replyMessage = ReplyMessage(
                    id: String(replyId),
                    user: ExyteChat.User(
                        id: "0",
                        name: msg.replyToSenderName ?? "",
                        avatarURL: nil,
                        isCurrentUser: false
                    ),
                    text: replyBody
                )
            }

            return ExyteChat.Message(
                id: String(msg.id),
                user: user,
                createdAt: createdAt,
                text: msg.body,
                attachments: attachments,
                replyMessage: replyMessage
            )
        }
    }
}
```

**Commit:**
```
feat: ChatAdapter fuer Exyte Chat Message-Konvertierung
```

---

### Task 5: Neue ChatDetailView mit Exyte Chat

**Files:**
- Create: `PawCoach/Features/Messaging/ChatDetailView.swift`

Dies ersetzt `MessageDetailView` als neue Chat-Ansicht.

```swift
import SwiftUI
import ExyteChat
import MapKit

struct ChatDetailView: View {
    @State private var conversation: Conversation?
    private let contact: Contact?
    @State private var viewModel = MessagingViewModel.shared
    @Environment(AppState.self) private var appState
    @State private var showContactInfo = false
    @State private var locationToShow: PawCoach.Message?

    init(conversation: Conversation) {
        self._conversation = State(initialValue: conversation)
        self.contact = nil
    }

    init(contact: Contact) {
        self._conversation = State(initialValue: nil)
        self.contact = contact
    }

    private var conversationId: Int? { conversation?.id }
    private var currentUserId: Int { appState.currentUser?.id ?? 0 }

    var body: some View {
        ChatView(
            messages: ChatAdapter.toExyte(
                messages: viewModel.messages,
                currentUserId: currentUserId
            ),
            chatType: .conversation,
            replyMode: .quote
        ) { draft in
            Task { await sendDraft(draft) }
        }
        .showMessageTimeView(true)
        .messageUseMarkdown(false)
        .setAvailableInputs([.full])
        .chatTheme(pawcoachTheme)
        .navigationTitle(displayTitle)
        .navigationBarTitleDisplayMode(.inline)
        .toolbar {
            ToolbarItem(placement: .principal) {
                Button { showContactInfo = true } label: {
                    VStack(spacing: 1) {
                        Text(displayTitle).font(.headline)
                        if let lastSeen = lastSeenText {
                            Text(lastSeen)
                                .font(.caption2)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
            }
        }
        .task {
            if let convId = conversationId {
                await viewModel.loadMessages(conversationId: convId)
                await viewModel.markDelivered(conversationId: convId)
                WebSocketManager.shared.joinConversation(convId)
            }
        }
        .onDisappear {
            if let convId = conversationId {
                WebSocketManager.shared.leaveConversation(convId)
            }
        }
        .navigationDestination(isPresented: $showContactInfo) {
            if let conv = conversation {
                ContactInfoView(conversation: conv, currentUserId: currentUserId)
            }
        }
    }

    // MARK: - Helpers

    private var displayTitle: String {
        conversation?.displayName ?? conversation?.subject ?? contact?.name ?? "Chat"
    }

    private var lastSeenText: String? {
        guard let iso = conversation?.lastSeen,
              let date = DateFormatters.parseISO8601(iso) else { return nil }
        if date.timeIntervalSinceNow > -60 { return "online" }
        if Calendar.current.isDateInToday(date) {
            return "zuletzt online um \(DateFormatters.timeDE.string(from: date))"
        }
        return "zuletzt online \(DateFormatters.dateTimeDE.string(from: date))"
    }

    private var pawcoachTheme: ChatTheme {
        ChatTheme(colors: .init(
            mainBackground: Color(.systemBackground),
            buttonBackground: Color.pawcoachGreen,
            myMessage: Color.pawcoachGreen,
            friendMessage: Color(.systemGray5),
            textMediaPicker: Color.pawcoachGreen
        ))
    }

    private func sendDraft(_ draft: DraftMessage) async {
        guard let convId = conversationId else {
            // Neue Konversation starten
            if let contactId = contact?.id {
                if let newConvId = await viewModel.startNewConversation(
                    recipientId: contactId,
                    content: draft.text,
                    senderId: currentUserId,
                    senderName: appState.currentUser?.name
                ) {
                    conversation = Conversation(convId: newConvId, subject: contact?.name)
                }
            }
            return
        }

        // Via Socket.IO senden
        if WebSocketManager.shared.isConnected {
            WebSocketManager.shared.sendMessage(
                conversationId: convId,
                body: draft.text,
                replyToId: nil // TODO: Reply aus draft extrahieren
            )
        } else {
            _ = await viewModel.sendMessage(
                conversationId: convId,
                content: draft.text,
                senderId: currentUserId,
                senderName: appState.currentUser?.name
            )
        }
    }
}
```

**Commit:**
```
feat: ChatDetailView mit Exyte Chat UI
```

---

### Task 6: MessagesListView auf neue ChatDetailView umstellen

**Files:**
- Modify: `PawCoach/Features/Messaging/MessagesListView.swift`

Alle `NavigationLink` die bisher zu `MessageDetailView` fuehrten auf `ChatDetailView` umstellen.

**Commit:**
```
feat: MessagesListView nutzt ChatDetailView statt Legacy
```

---

### Task 7: Standort-Nachrichten in Exyte Chat

**Files:**
- Modify: `PawCoach/Features/Messaging/ChatDetailView.swift`

Exyte Chat unterstuetzt custom message builders. Fuer Location-Nachrichten einen eigenen Builder erstellen der eine Karten-Vorschau zeigt.

```swift
messageBuilder: { message, posInUser, posInSection, posInComments,
    showContextMenu, messageAction, showAttachment in
    // Pruefe ob es eine Location-Nachricht ist
    let pawcoachMsg = findPawcoachMessage(exyteMsgId: message.id)
    if let msg = pawcoachMsg, msg.isLocation,
       let lat = msg.latitude, let lng = msg.longitude {
        LocationBubbleView(
            latitude: lat, longitude: lng,
            name: msg.locationName ?? msg.locationAddress ?? "Standort",
            isOwnMessage: message.user.isCurrentUser,
            time: message.createdAt
        ) {
            locationToShow = msg
        }
    } else {
        // Default Exyte Chat Bubble
        // (nil zurueckgeben laesst Exyte den Default rendern)
    }
}
```

**Commit:**
```
feat: Location-Nachrichten als Karten-Vorschau in Exyte Chat
```

---

## Phase 3: Typing, Read-Receipts, Standort

### Task 8: Typing-Indikator

**Files:**
- Modify: `PawCoach/Features/Messaging/ChatDetailView.swift`
- Modify: `PawCoach/Features/Messaging/MessagingViewModel.swift`

Im Input-Feld bei Texteingabe `sendTypingIndicator` aufrufen (throttled auf max 1x pro 3 Sekunden).

Bei Empfang von `user_typing` Event eine "schreibt..."-Anzeige unter dem Chat-Titel zeigen.

**Commit:**
```
feat: Typing-Indikator (schreibt...) via Socket.IO
```

---

### Task 9: Standort senden in neuem Chat

**Files:**
- Modify: `PawCoach/Features/Messaging/ChatDetailView.swift`

Toolbar-Menu mit "Standort senden" und "Live-Standort" Buttons. Nutzt bestehende REST-Endpoints (Standort senden ist kein Socket.IO Event).

**Commit:**
```
feat: Standort-Funktionen im neuen Chat
```

---

### Task 10: Live-Standort ueber Socket.IO

**Files:**
- Modify: `PawCoach/Features/Messaging/ChatDetailView.swift`
- Modify: `PawCoach/Features/Messaging/LiveLocationMapView.swift`

Live-Standort starten via REST (`POST /api/messages/{id}/live-location`).
Positions-Updates via Socket.IO senden (`location_update` Event).
Live-Location-Banner und Map-View einbinden.

**Commit:**
```
feat: Live-Standort ueber Socket.IO mit Echtzeit-Updates
```

---

### Task 11: Cleanup + Build + TestFlight

**Files:**
- Modify: `project.yml` (Build-Nummer erhoehen)

**Step 1:** Alle Compile-Fehler fixen
**Step 2:** Legacy-Code sicherstellen (bleibt als Backup)
**Step 3:** Build + Archive + TestFlight Upload
**Step 4:** Push to GitHub

**Commit:**
```
chore: Build N fuer TestFlight (Exyte Chat + Socket.IO)
```

---

## Backend-Anforderungen

Fuer Socket.IO mit JWT-Auth muss das Backend angepasst werden:

### Option A: JWT in Socket.IO Connect-Params (bevorzugt)

```python
@socketio.on("connect")
def handle_connect():
    token = request.args.get("token")
    if token:
        user = verify_jwt_token(token)
        if user:
            flask_login.login_user(user)
            join_room(f"user_{user.id}")
            return
    # Fallback auf Session-Auth
    if current_user.is_authenticated:
        join_room(f"user_{current_user.id}")
        return
    disconnect()
```

### Option B: Extra-Headers

Socket.IO-Client sendet `Authorization: Bearer <token>` Header. Backend muss diesen im connect-Handler pruefen.

**Backend-Entwickler informieren dass JWT-Auth fuer Socket.IO noetig ist.**

---

## Zusammenfassung

| Task | Beschreibung | Abhaengigkeiten |
|------|-------------|-----------------|
| 0 | Backup Legacy-Views | - |
| 1 | SPM Dependencies | - |
| 2 | Socket.IO WebSocketManager | Task 1 |
| 3 | ViewModel Socket.IO Integration | Task 2 |
| 4 | ChatAdapter (Model-Konvertierung) | Task 1 |
| 5 | ChatDetailView (Exyte Chat) | Task 3, 4 |
| 6 | MessagesListView umstellen | Task 5 |
| 7 | Location-Nachrichten | Task 5 |
| 8 | Typing-Indikator | Task 5 |
| 9 | Standort senden | Task 5 |
| 10 | Live-Standort Socket.IO | Task 5 |
| 11 | Cleanup + TestFlight | Alle |
