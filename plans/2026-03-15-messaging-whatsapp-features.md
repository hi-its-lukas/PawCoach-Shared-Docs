# Messaging WhatsApp-Features Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** PawCoach Messaging auf WhatsApp-Niveau bringen — inline Bubble-Zeiten, Nachrichteninfo, Kontaktinfo-Seite, Hunde-Kontakte, Sichtbarkeitseinstellungen, Zitieren, Suche, Zuletzt-Online.

**Architecture:** Bestehende Messaging-Views erweitern (MessageBubble, MessageDetailView, NewConversationView), neue Views erstellen (ContactInfoView, ChatSearchView), MessagingViewModel + Models erweitern. Backend-Endpoints vorbereiten wo nötig, UserDefaults für lokale Einstellungen.

**Tech Stack:** SwiftUI, Swift 5.9, @Observable ViewModels, UserDefaults, bestehender APIClient

**Design-Doc:** `plans/2026-03-15-messaging-whatsapp-features-design.md`

---

## Phase 1 — Rein Frontend (sofort umsetzbar)

### Task 1: Bubble-Layout — Uhrzeit + Häkchen inline

**Files:**
- Modify: `PawCoach/Features/Messaging/MessageDetailView.swift` (MessageBubble struct, ca. Zeile 598-651)

**Step 1: MessageBubble body umbauen — Zeit+Status inline in die Bubble**

Aktuell sind Text-Bubble und Zeit-HStack separate Views in einer VStack. Neu: Ein einzelner Text-Block mit Overlay für Zeit+Häkchen unten rechts, INNERHALB der Bubble-Hintergrundfarbe.

```swift
// In MessageBubble, den body ersetzen:
var body: some View {
    HStack {
        if isOwnMessage { Spacer(minLength: 60) }

        VStack(alignment: isOwnMessage ? .trailing : .leading, spacing: 4) {
            if showSenderName, !isOwnMessage, let name = message.senderName {
                Text(name)
                    .font(.caption2)
                    .foregroundStyle(.secondary)
            }

            if message.hasMedia {
                mediaBubble
            }

            if !message.body.isEmpty {
                // Text + inline Zeit in einer Bubble
                HStack(alignment: .bottom, spacing: 0) {
                    Text(message.body)
                        .font(.body)

                    // Unsichtbarer Platzhalter fuer die Zeit
                    Text(timeStatusPlaceholder)
                        .font(.caption2)
                        .foregroundStyle(.clear)
                        .padding(.leading, 4)
                }
                .overlay(alignment: .bottomTrailing) {
                    timeStatusView
                        .padding(.bottom, 1)
                }
                .padding(.horizontal, 12)
                .padding(.vertical, 8)
                .background(
                    isOwnMessage ? Color.pawcoachGreen : Color(.systemGray5),
                    in: .rect(cornerRadius: CGFloat(Theme.Radius.lg))
                )
                .foregroundStyle(isOwnMessage ? .white : .primary)
            } else if !message.hasMedia {
                // Nur Zeit (fuer reine Media-Nachrichten ohne Text)
                timeStatusView
            }
        }

        if !isOwnMessage { Spacer(minLength: 60) }
    }
}
```

**Step 2: timeStatusView und timeStatusPlaceholder Properties hinzufügen**

```swift
// Neues computed property in MessageBubble:
private var timeStatusView: some View {
    HStack(spacing: 3) {
        if message.isEdited {
            Text("(bearb.)")
                .font(.caption2)
        }
        if let time = formattedTime {
            Text(time)
                .font(.caption2)
        }
        if isOwnMessage {
            deliveryStatusIcon
        }
    }
    .foregroundStyle(isOwnMessage ? .white.opacity(0.7) : .secondary)
}

// Platzhalter-String damit Text um die Zeit fliesst:
private var timeStatusPlaceholder: String {
    var placeholder = ""
    if message.isEdited { placeholder += "(bearb.) " }
    if let time = formattedTime { placeholder += time }
    if isOwnMessage { placeholder += " ✓✓" }
    return placeholder
}
```

**Step 3: Alten separaten Zeit-HStack entfernen**

Den gesamten `HStack(spacing: 4) { ... }` Block mit Zeit/Bearbeitet/Status unter der Bubble löschen (alter Code Zeile 632-646).

**Step 4: Build & Verify**

Run: `xcodebuild -project PawCoach.xcodeproj -scheme PawCoach -destination 'generic/platform=iOS' -quiet build 2>&1 | grep "error:"`
Expected: Keine Fehler

**Step 5: Commit**

```
feat: Bubble-Layout mit inline Uhrzeit + Häkchen (WhatsApp-Stil)
```

---

### Task 2: Nachrichteninfo-Sheet verbessern

**Files:**
- Modify: `PawCoach/Features/Messaging/MessageDetailView.swift` (MessageInfoSheet struct, ca. Zeile 777+)

**Step 1: MessageInfoSheet komplett neu gestalten (WhatsApp-Stil)**

```swift
struct MessageInfoSheet: View {
    let message: Message
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 0) {
                    // Nachricht als Bubble oben
                    MessageBubble(
                        message: message,
                        isOwnMessage: true,
                        showSenderName: false
                    )
                    .padding()
                    .padding(.top, 8)

                    // Zustellstatus-Liste
                    VStack(spacing: 0) {
                        statusRow(
                            icon: "checkmark",
                            iconDouble: true,
                            label: "Gelesen",
                            time: readTime,
                            color: .pawcoachGreen
                        )

                        Divider().padding(.leading, 56)

                        statusRow(
                            icon: "checkmark",
                            iconDouble: true,
                            label: "Zugestellt",
                            time: deliveredTime,
                            color: .secondary
                        )
                    }
                    .background(Color(.systemBackground))
                    .clipShape(.rect(cornerRadius: 10))
                    .padding(.horizontal)
                    .padding(.top, 12)
                }
            }
            .background(Color(.systemGroupedBackground))
            .navigationTitle("Nachrichteninfo")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Fertig") { dismiss() }
                }
            }
        }
    }

    private var deliveredTime: String? {
        guard let status = message.deliveryStatus,
              status == .delivered || status == .read else { return nil }
        return formattedDateTime
    }

    private var readTime: String? {
        guard message.deliveryStatus == .read else { return nil }
        // Backend liefert noch keinen separaten read_at Timestamp
        return nil
    }

    private var formattedDateTime: String? {
        guard let iso = message.createdAt,
              let date = DateFormatters.parseISO8601(iso) else { return nil }
        return DateFormatters.dateTimeDE.string(from: date)
    }

    private func statusRow(icon: String, iconDouble: Bool, label: String, time: String?, color: Color) -> some View {
        HStack(spacing: 12) {
            HStack(spacing: -3) {
                Image(systemName: icon)
                if iconDouble {
                    Image(systemName: icon)
                }
            }
            .font(.body)
            .foregroundStyle(color)
            .frame(width: 32)

            Text(label)
                .font(.body)

            Spacer()

            if let time {
                Text(time)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            } else {
                Text("○ ○ ○")
                    .font(.caption)
                    .foregroundStyle(.tertiary)
            }
        }
        .padding(.horizontal, 16)
        .padding(.vertical, 14)
    }
}
```

**Step 2: Build & Verify**

Run: `xcodebuild ... -quiet build 2>&1 | grep "error:"`

**Step 3: Commit**

```
feat: Nachrichteninfo-Sheet im WhatsApp-Stil (Bubble + Zustellstatus)
```

---

### Task 3: Suche im Chat

**Files:**
- Modify: `PawCoach/Features/Messaging/MessageDetailView.swift` (Toolbar + neuer State)

**Step 1: State-Variablen für Suche hinzufügen**

In `MessageDetailView` hinzufügen:
```swift
@State private var isSearching = false
@State private var searchText = ""
@State private var searchResultIndices: [Int] = []
@State private var currentSearchIndex = 0
```

**Step 2: Lupe-Button in Toolbar hinzufügen**

Im bestehenden Toolbar-Menu (Zeile 92+), vor dem Menu-Button:
```swift
Button {
    withAnimation { isSearching.toggle() }
} label: {
    Image(systemName: "magnifyingglass")
        .foregroundStyle(Color.pawcoachGreen)
}
```

**Step 3: Suchleiste einbauen**

Unter der NavigationBar, vor `messagesList`:
```swift
if isSearching {
    HStack(spacing: 8) {
        Image(systemName: "magnifyingglass")
            .foregroundStyle(.secondary)
        TextField("Im Chat suchen...", text: $searchText)
            .textFieldStyle(.plain)
            .focused($isInputFocused)

        if !searchResultIndices.isEmpty {
            Text("\(currentSearchIndex + 1)/\(searchResultIndices.count)")
                .font(.caption)
                .foregroundStyle(.secondary)
                .fixedSize()

            Button { navigateSearch(direction: -1) } label: {
                Image(systemName: "chevron.up")
            }
            Button { navigateSearch(direction: 1) } label: {
                Image(systemName: "chevron.down")
            }
        }

        Button("Abbrechen") {
            isSearching = false
            searchText = ""
            searchResultIndices = []
        }
        .font(.subheadline)
    }
    .padding(.horizontal)
    .padding(.vertical, 8)
    .background(Color(.systemBackground))
}
```

**Step 4: Such-Logik implementieren**

```swift
private func navigateSearch(direction: Int) {
    guard !searchResultIndices.isEmpty else { return }
    currentSearchIndex = (currentSearchIndex + direction + searchResultIndices.count) % searchResultIndices.count
}
```

Mit `.onChange(of: searchText)` die `searchResultIndices` aktualisieren:
```swift
.onChange(of: searchText) { _, query in
    guard !query.isEmpty else {
        searchResultIndices = []
        return
    }
    searchResultIndices = viewModel.messages.indices.filter {
        viewModel.messages[$0].body.localizedCaseInsensitiveContains(query)
    }
    currentSearchIndex = searchResultIndices.isEmpty ? 0 : searchResultIndices.count - 1
}
```

**Step 5: Treffer hervorheben und dorthin scrollen**

In der `messagesList`, den ScrollViewReader nutzen um zum Treffer zu scrollen. Bei `.onChange(of: currentSearchIndex)`:
```swift
.onChange(of: currentSearchIndex) { _, idx in
    guard searchResultIndices.indices.contains(idx) else { return }
    let msgId = viewModel.messages[searchResultIndices[idx]].id
    withAnimation { proxy.scrollTo(msgId, anchor: .center) }
}
```

In der MessageBubble prüfen ob die Nachricht ein Suchtreffer ist und ggf. Hintergrund hervorheben (gelber Rand o.ä.).

**Step 6: Build & Verify, Commit**

```
feat: Chat-Suche mit Navigation zwischen Treffern
```

---

### Task 4: Nachricht zitieren (Reply) — Frontend

**Files:**
- Modify: `PawCoach/Features/Messaging/MessagingModels.swift` (Message struct)
- Modify: `PawCoach/Features/Messaging/MessageDetailView.swift` (Reply-UI)

**Step 1: Message-Model um Reply-Felder erweitern**

```swift
// In Message struct hinzufügen:
let replyToId: Int?
let replyToBody: String?
let replyToSenderName: String?
```

CodingKeys:
```swift
case replyToId = "reply_to_id"
case replyToBody = "reply_to_body"
case replyToSenderName = "reply_to_sender_name"
```

In `init(from decoder:)` optional decodieren. In `localMessage()` als nil setzen.

**Step 2: Reply-State in MessageDetailView**

```swift
@State private var replyingToMessage: Message?
```

**Step 3: Swipe-Right Gesture für Reply**

Die bestehende `SwipeableMessageRow` erweitern mit einer `onSwipeRight`-Closure. Swipe nach rechts → Reply-Modus aktivieren:
```swift
// Swipe rechts: Reply-Icon einblenden
if onSwipeRight != nil {
    HStack {
        Image(systemName: "arrowshape.turn.up.left.fill")
            .font(.title3)
            .foregroundStyle(.secondary)
            .opacity(min(1, offset / 60))
        Spacer()
    }
    .padding(.leading, 8)
}
```

**Step 4: Reply-Vorschau über dem Eingabefeld**

```swift
// Über inputBar:
if let reply = replyingToMessage {
    HStack(spacing: 8) {
        Rectangle()
            .fill(Color.pawcoachGreen)
            .frame(width: 3)
        VStack(alignment: .leading, spacing: 2) {
            Text(reply.senderName ?? "")
                .font(.caption)
                .fontWeight(.semibold)
                .foregroundStyle(Color.pawcoachGreen)
            Text(reply.body)
                .font(.caption)
                .foregroundStyle(.secondary)
                .lineLimit(2)
        }
        Spacer()
        Button { replyingToMessage = nil } label: {
            Image(systemName: "xmark.circle.fill")
                .foregroundStyle(.secondary)
        }
    }
    .padding(.horizontal)
    .padding(.vertical, 6)
    .background(Color(.systemGray6))
}
```

**Step 5: Zitat-Anzeige in der Bubble**

In MessageBubble, vor dem Nachrichtentext:
```swift
if let replyBody = message.replyToBody {
    VStack(alignment: .leading, spacing: 2) {
        if let name = message.replyToSenderName {
            Text(name)
                .font(.caption2)
                .fontWeight(.semibold)
                .foregroundStyle(isOwnMessage ? .white.opacity(0.9) : Color.pawcoachGreen)
        }
        Text(replyBody)
            .font(.caption2)
            .lineLimit(2)
            .foregroundStyle(isOwnMessage ? .white.opacity(0.7) : .secondary)
    }
    .padding(8)
    .frame(maxWidth: .infinity, alignment: .leading)
    .background(
        isOwnMessage ? Color.white.opacity(0.15) : Color(.systemGray4).opacity(0.5),
        in: .rect(cornerRadius: 6)
    )
    .padding(.horizontal, 8)
    .padding(.top, 8)
}
```

**Step 6: sendMessage erweitern für reply_to**

In `sendMessage` und `Endpoint.sendMessage` den `reply_to` Parameter hinzufügen:
```swift
static func sendMessage(conversationId: Int, content: String, replyToId: Int? = nil) -> Endpoint {
    var params: [String: Any] = ["body": content]
    if let replyToId { params["reply_to_id"] = replyToId }
    return Endpoint(path: "/api/messages/\(conversationId)", method: "POST", body: jsonBody(params))
}
```

**Step 7: Build & Verify, Commit**

```
feat: Nachrichten zitieren mit Swipe-Right und Reply-Vorschau
```

---

### Task 5: Stumm schalten (lokal)

**Files:**
- Create: `PawCoach/Features/Messaging/MuteManager.swift`

**Step 1: MuteManager erstellen**

```swift
import Foundation

enum MuteManager {
    private static let key = "mutedConversations"

    static func isMuted(_ conversationId: Int) -> Bool {
        let muted = UserDefaults.standard.array(forKey: key) as? [Int] ?? []
        return muted.contains(conversationId)
    }

    static func toggle(_ conversationId: Int) {
        var muted = UserDefaults.standard.array(forKey: key) as? [Int] ?? []
        if let idx = muted.firstIndex(of: conversationId) {
            muted.remove(at: idx)
        } else {
            muted.append(conversationId)
        }
        UserDefaults.standard.set(muted, forKey: key)
    }
}
```

**Step 2: Build & Verify, Commit**

```
feat: MuteManager fuer stumm geschaltete Konversationen (UserDefaults)
```

---

## Phase 2 — Neue Views + einfaches Backend

### Task 6: Kontaktinfo-Seite

**Files:**
- Create: `PawCoach/Features/Messaging/ContactInfoView.swift`
- Modify: `PawCoach/Features/Messaging/MessageDetailView.swift` (Navigation)
- Modify: `PawCoach/Features/Messaging/MessagingViewModel.swift` (Daten laden)

**Step 1: ContactInfoView erstellen**

```swift
import SwiftUI

struct ContactInfoView: View {
    let conversation: Conversation
    let currentUserId: Int?
    @State private var viewModel = MessagingViewModel.shared
    @State private var showBlockConfirm = false
    @State private var showReportConfirm = false
    @State private var showClearConfirm = false
    @State private var mediaCount = 0
    @Environment(\.dismiss) private var dismiss

    private var otherMember: ConversationMember? {
        conversation.members?.first { $0.userId != currentUserId }
    }

    var body: some View {
        List {
            headerSection
            mediaSection
            // dogsSection (Task 7)
            // coursesSection (Task 9)
            notificationSection
            dangerSection
        }
        .listStyle(.insetGrouped)
        .navigationTitle("Kontaktinfo")
        .navigationBarTitleDisplayMode(.inline)
        .task { await loadData() }
        .alert("Chat leeren?", isPresented: $showClearConfirm) {
            Button("Leeren", role: .destructive) {
                Task { await clearChat() }
            }
            Button("Abbrechen", role: .cancel) {}
        } message: {
            Text("Alle Nachrichten in diesem Chat werden unwiderruflich geloescht.")
        }
        .alert("Blockieren?", isPresented: $showBlockConfirm) {
            Button("Blockieren", role: .destructive) {
                Task { await blockUser() }
            }
            Button("Abbrechen", role: .cancel) {}
        } message: {
            Text("Diese Person kann dir keine Nachrichten mehr senden.")
        }
        .alert("Melden?", isPresented: $showReportConfirm) {
            Button("Melden", role: .destructive) {
                Task { await reportUser() }
            }
            Button("Abbrechen", role: .cancel) {}
        } message: {
            Text("Diese Person wird dem Admin gemeldet.")
        }
    }

    // MARK: - Header

    private var headerSection: some View {
        Section {
            VStack(spacing: 8) {
                // Avatar
                ZStack {
                    Circle()
                        .fill(Color.pawcoachGreen.opacity(0.15))
                        .frame(width: 80, height: 80)
                    Text(initials)
                        .font(.title)
                        .fontWeight(.semibold)
                        .foregroundStyle(Color.pawcoachGreen)
                }

                Text(conversation.displayName ?? conversation.subject ?? "Chat")
                    .font(.title2)
                    .fontWeight(.bold)

                if let role = otherMember?.role {
                    Text(role.capitalized)
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }

                // Zuletzt online (Task 10)
            }
            .frame(maxWidth: .infinity)
            .listRowBackground(Color.clear)
        }
    }

    // MARK: - Media

    private var mediaSection: some View {
        Section {
            HStack {
                Label("Medien, Links, Doks", systemImage: "photo.on.rectangle")
                Spacer()
                Text("\(mediaCount)")
                    .foregroundStyle(.secondary)
                Image(systemName: "chevron.right")
                    .font(.caption)
                    .foregroundStyle(.tertiary)
            }
        }
    }

    // MARK: - Notifications

    private var notificationSection: some View {
        Section {
            Toggle(isOn: Binding(
                get: { MuteManager.isMuted(conversation.id) },
                set: { _ in MuteManager.toggle(conversation.id) }
            )) {
                Label("Stummgeschaltet", systemImage: "bell.slash")
            }

            Button {
                showClearConfirm = true
            } label: {
                Label("Chat leeren", systemImage: "trash")
                    .foregroundStyle(.primary)
            }
        }
    }

    // MARK: - Danger

    private var dangerSection: some View {
        Section {
            Button {
                showBlockConfirm = true
            } label: {
                Label("Blockieren", systemImage: "hand.raised.fill")
                    .foregroundStyle(.red)
            }

            Button {
                showReportConfirm = true
            } label: {
                Label("Melden", systemImage: "exclamationmark.triangle.fill")
                    .foregroundStyle(.red)
            }
        }
    }

    // MARK: - Helpers

    private var initials: String {
        let name = conversation.displayName ?? ""
        let parts = name.split(separator: " ")
        if parts.count >= 2 {
            return "\(parts[0].prefix(1))\(parts[1].prefix(1))"
        }
        return String(name.prefix(2)).uppercased()
    }

    private func loadData() async {
        mediaCount = viewModel.messages.filter { $0.hasMedia }.count
    }

    private func clearChat() async {
        // Lokal leeren + ggf. Backend-Call
        viewModel.messages.removeAll()
    }

    private func blockUser() async {
        guard let userId = otherMember?.userId else { return }
        // TODO: Backend-Endpoint POST /api/messages/block/<user_id>
        do {
            try await APIClient.shared.requestVoid(
                Endpoint(path: "/api/messages/block/\(userId)", method: "POST")
            )
        } catch {
            viewModel.errorMessage = "Blockieren fehlgeschlagen"
        }
    }

    private func reportUser() async {
        guard let userId = otherMember?.userId else { return }
        // TODO: Backend-Endpoint POST /api/messages/report/<user_id>
        do {
            try await APIClient.shared.requestVoid(
                Endpoint(path: "/api/messages/report/\(userId)", method: "POST")
            )
        } catch {
            viewModel.errorMessage = "Melden fehlgeschlagen"
        }
    }
}
```

**Step 2: ConversationMember um `role` erweitern (falls nicht vorhanden)**

In `MessagingModels.swift`, `ConversationMember` prüfen und ggf. `role` hinzufügen:
```swift
struct ConversationMember: Identifiable, Decodable {
    let id: Int
    let userId: Int
    let userName: String?
    let role: String?

    enum CodingKeys: String, CodingKey {
        case id, role
        case userId = "user_id"
        case userName = "user_name"
    }
}
```

**Step 3: Navigation in MessageDetailView — Tap auf Name öffnet Kontaktinfo**

Den `.navigationTitle(displayTitle)` durch einen Toolbar-Button ersetzen:
```swift
.toolbar {
    ToolbarItem(placement: .principal) {
        Button {
            if let conv = conversation {
                showContactInfo = true
            }
        } label: {
            Text(displayTitle)
                .font(.headline)
                .foregroundStyle(.primary)
        }
    }
}
```

State: `@State private var showContactInfo = false`

NavigationLink oder Sheet:
```swift
.navigationDestination(isPresented: $showContactInfo) {
    if let conv = conversation {
        ContactInfoView(conversation: conv, currentUserId: appState.currentUser?.id)
    }
}
```

**Step 4: Build & Verify, Commit**

```
feat: Kontaktinfo-Seite mit Avatar, Medien, Stumm, Blockieren, Melden
```

---

### Task 7: Hunde in der Kontaktauswahl

**Files:**
- Modify: `PawCoach/Features/Messaging/NewConversationView.swift`
- Modify: `PawCoach/Features/Messaging/MessagingViewModel.swift` (Hunde laden)
- Modify: `PawCoach/Features/Messaging/MessagingModels.swift` (DogContact Model)

**Step 1: DogContact Model erstellen**

In `MessagingModels.swift`:
```swift
struct DogContact: Identifiable, Decodable {
    let id: Int
    let name: String
    let breed: String?
    let owners: [DogOwner]?

    struct DogOwner: Identifiable, Decodable {
        let id: Int
        let name: String
    }

    var ownerNames: String {
        owners?.map(\.name).joined(separator: ", ") ?? ""
    }

    var ownerIds: [Int] {
        owners?.map(\.id) ?? []
    }
}

struct DogContactsResponse: Decodable {
    let dogs: [DogContact]
}
```

**Step 2: MessagingViewModel um Hunde-Laden erweitern**

```swift
var dogContacts: [DogContact] = []

func loadDogContacts(isAdmin: Bool) async {
    do {
        let endpoint = isAdmin
            ? Endpoint(path: "/api/admin/dogs")
            : Endpoint(path: "/api/customer/dogs")
        let response = try await api.request(endpoint, as: DogContactsResponse.self)
        dogContacts = response.dogs.filter { !($0.owners?.isEmpty ?? true) }
    } catch {
        logger.error("Hunde-Kontakte laden fehlgeschlagen: \(error)")
    }
}
```

**Step 3: NewConversationView um Hunde-Sektion erweitern**

```swift
// Neues Property:
@Environment(AppState.self) private var appState

private var isAdminOrTrainer: Bool {
    let role = appState.currentUser?.role ?? ""
    return role == "admin" || role == "trainer"
}

// Body erweitern — zwei Sektionen:
private var contactList: some View {
    List {
        // Personen-Sektion
        Section("Personen") {
            ForEach(filteredContacts) { contact in
                // ... bestehender Contact-Button ...
            }
        }

        // Hunde-Sektion
        if !filteredDogs.isEmpty {
            Section("Hunde") {
                ForEach(filteredDogs) { dog in
                    Button {
                        dismiss()
                        Task {
                            try? await Task.sleep(for: .milliseconds(300))
                            await startDogGroupConversation(dog: dog)
                        }
                    } label: {
                        HStack(spacing: 12) {
                            Image(systemName: "pawprint.circle.fill")
                                .font(.title2)
                                .foregroundStyle(Color.pawcoachGreen)
                            VStack(alignment: .leading, spacing: 2) {
                                Text(dog.name)
                                    .font(.body)
                                    .foregroundStyle(.primary)
                                Text(dog.ownerNames)
                                    .font(.caption)
                                    .foregroundStyle(.secondary)
                            }
                            Spacer()
                            Image(systemName: "chevron.right")
                                .font(.caption)
                                .foregroundStyle(.tertiary)
                        }
                    }
                }
            }
        }
    }
    .listStyle(.plain)
}

private var filteredDogs: [DogContact] {
    if searchText.isEmpty { return viewModel.dogContacts }
    return viewModel.dogContacts.filter {
        $0.name.localizedCaseInsensitiveContains(searchText) ||
        $0.ownerNames.localizedCaseInsensitiveContains(searchText)
    }
}
```

**Step 4: Gruppen-Konversation für Hund starten**

```swift
private func startDogGroupConversation(dog: DogContact) async {
    guard !dog.ownerIds.isEmpty else { return }
    // Gruppenkonversation mit allen Haltern erstellen
    // Betreff = Hundename
    // Nutzt bestehenden createGroupConversation Endpoint
    // onContactSelected wird hier nicht genutzt — stattdessen direkt Gruppe erstellen
}
```

**Step 5: .task erweitern um Hunde zu laden**

```swift
.task {
    await viewModel.loadContacts()
    await viewModel.loadDogContacts(isAdmin: isAdminOrTrainer)
}
```

**Step 6: Build & Verify, Commit**

```
feat: Hunde-Sektion in Kontaktauswahl mit Halter-Anzeige
```

---

### Task 8: Chat leeren (Backend-Integration)

**Files:**
- Modify: `PawCoach/Core/Network/Endpoint+Messaging.swift`
- Modify: `PawCoach/Features/Messaging/ContactInfoView.swift`

**Step 1: Endpoint hinzufügen**

```swift
static func clearConversation(id: Int) -> Endpoint {
    Endpoint(path: "/api/messages/\(id)/clear", method: "DELETE")
}
```

**Step 2: clearChat() in ContactInfoView aktualisieren**

```swift
private func clearChat() async {
    do {
        try await APIClient.shared.requestVoid(.clearConversation(id: conversation.id))
    } catch {
        // Lokal trotzdem leeren falls Backend-Endpoint noch nicht existiert
    }
    viewModel.messages.removeAll()
    dismiss()
}
```

**Step 3: Build & Verify, Commit**

```
feat: Chat leeren mit Backend-Integration
```

---

### Task 9: Zuletzt Online

**Files:**
- Modify: `PawCoach/Features/Messaging/MessageDetailView.swift` (Header)
- Modify: `PawCoach/Features/Messaging/ContactInfoView.swift`
- Modify: `PawCoach/Features/Messaging/MessagingModels.swift`

**Step 1: LastSeen-Daten zum Conversation-Model hinzufügen**

In `MessagingModels.swift` — `Conversation`:
```swift
let lastSeen: String?  // ISO8601 Timestamp

// CodingKey:
case lastSeen = "last_seen"
```

**Step 2: Anzeige im Chat-Header**

Unter dem Namen in der Toolbar:
```swift
ToolbarItem(placement: .principal) {
    VStack(spacing: 1) {
        Text(displayTitle)
            .font(.headline)
        if let lastSeen = conversation?.lastSeen,
           let date = DateFormatters.parseISO8601(lastSeen) {
            Text(lastSeenText(date: date))
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
    }
}

private func lastSeenText(date: Date) -> String {
    if date.timeIntervalSinceNow > -60 {
        return "online"
    }
    if Calendar.current.isDateInToday(date) {
        return "zuletzt online um \(DateFormatters.timeDE.string(from: date))"
    }
    return "zuletzt online \(DateFormatters.dateTimeDE.string(from: date))"
}
```

**Step 3: Auch in ContactInfoView anzeigen**

Im Header unter dem Rollen-Text.

**Step 4: Build & Verify, Commit**

```
feat: Zuletzt-Online Anzeige im Chat-Header und Kontaktinfo
```

---

## Phase 3 — Backend-Erweiterungen

### Task 10: Discoverable-Setting + Opt-in

**Files:**
- Modify: `PawCoach/Features/Settings/SettingsView.swift`
- Create: `PawCoach/Features/Settings/PrivacyVisibilityView.swift`
- Modify: `PawCoach/Core/Network/Endpoint+Messaging.swift`

**Step 1: PrivacyVisibilityView erstellen**

```swift
import SwiftUI

struct PrivacyVisibilityView: View {
    @State private var discoverable = false
    @State private var lastNameVisible = true
    @State private var profilePhotoVisibility = "all"       // all, trainers, nobody
    @State private var dogsVisibility = "trainers"           // all, trainers, nobody
    @State private var dogOwnersVisibility = "trainers"      // all, trainers, nobody
    @State private var phoneVisibility = "nobody"            // all, trainers, nobody
    @State private var lastSeenVisibility = "all"            // all, trainers, nobody
    @State private var isLoading = true

    var body: some View {
        List {
            Section {
                Toggle("Fuer andere Mitglieder sichtbar", isOn: $discoverable)
            } header: {
                Text("Auffindbarkeit")
            } footer: {
                Text("Wenn aktiviert, koennen dich andere Hundeschulmitglieder finden und kontaktieren.")
            }

            Section("Profildetails") {
                Toggle("Nachname vollstaendig anzeigen", isOn: $lastNameVisible)

                Picker("Profilbild", selection: $profilePhotoVisibility) {
                    Text("Alle").tag("all")
                    Text("Nur Trainer").tag("trainers")
                    Text("Niemand").tag("nobody")
                }

                Picker("Meine Hunde", selection: $dogsVisibility) {
                    Text("Alle").tag("all")
                    Text("Nur Trainer").tag("trainers")
                    Text("Niemand").tag("nobody")
                }

                Picker("Weitere Halter meiner Hunde", selection: $dogOwnersVisibility) {
                    Text("Alle").tag("all")
                    Text("Nur Trainer").tag("trainers")
                    Text("Niemand").tag("nobody")
                }

                Picker("Telefonnummer", selection: $phoneVisibility) {
                    Text("Alle").tag("all")
                    Text("Nur Trainer").tag("trainers")
                    Text("Niemand").tag("nobody")
                }

                Picker("Zuletzt online", selection: $lastSeenVisibility) {
                    Text("Alle").tag("all")
                    Text("Nur Trainer").tag("trainers")
                    Text("Niemand").tag("nobody")
                }
            }
        }
        .listStyle(.insetGrouped)
        .navigationTitle("Sichtbarkeit")
        .task { await loadSettings() }
        .onChange(of: discoverable) { _, _ in Task { await saveSettings() } }
        .onChange(of: lastNameVisible) { _, _ in Task { await saveSettings() } }
        .onChange(of: profilePhotoVisibility) { _, _ in Task { await saveSettings() } }
        .onChange(of: dogsVisibility) { _, _ in Task { await saveSettings() } }
        .onChange(of: dogOwnersVisibility) { _, _ in Task { await saveSettings() } }
        .onChange(of: phoneVisibility) { _, _ in Task { await saveSettings() } }
        .onChange(of: lastSeenVisibility) { _, _ in Task { await saveSettings() } }
    }

    private func loadSettings() async {
        // GET /api/auth/profile/visibility
        // TODO: Backend-Integration
        isLoading = false
    }

    private func saveSettings() async {
        // PATCH /api/auth/profile/visibility
        // TODO: Backend-Integration
    }
}
```

**Step 2: In SettingsView verlinken**

In `privacySection`:
```swift
NavigationLink {
    PrivacyVisibilityView()
} label: {
    Label {
        VStack(alignment: .leading, spacing: 2) {
            Text("Sichtbarkeit")
            Text("Wer kann was sehen")
                .font(.caption)
                .foregroundStyle(.secondary)
        }
    } icon: {
        Image(systemName: "eye")
    }
}
```

**Step 3: Endpoints hinzufügen**

```swift
static func profileVisibility() -> Endpoint {
    Endpoint(path: "/api/auth/profile/visibility")
}

static func updateProfileVisibility(body: Data) -> Endpoint {
    Endpoint(path: "/api/auth/profile/visibility", method: "PATCH", body: body)
}
```

**Step 4: Build & Verify, Commit**

```
feat: Sichtbarkeits-Einstellungen (discoverable, Profil-Felder)
```

---

### Task 11: Hunde + Halter in Kontaktinfo anzeigen

**Files:**
- Modify: `PawCoach/Features/Messaging/ContactInfoView.swift`

**Step 1: Hunde-Sektion in ContactInfoView**

```swift
// Neuer State:
@State private var dogs: [DogContact] = []

// Neue Section:
private var dogsSection: some View {
    Section("Hunde") {
        ForEach(dogs) { dog in
            VStack(alignment: .leading, spacing: 4) {
                HStack {
                    Image(systemName: "pawprint.fill")
                        .foregroundStyle(Color.pawcoachGreen)
                    Text(dog.name)
                        .fontWeight(.medium)
                    if let breed = dog.breed {
                        Text("(\(breed))")
                            .foregroundStyle(.secondary)
                    }
                }

                if let owners = dog.owners, !owners.isEmpty {
                    ForEach(owners.filter { $0.id != (otherMember?.userId ?? -1) }) { owner in
                        NavigationLink {
                            // Neuen Chat mit diesem Halter starten
                            // oder Kontaktinfo anzeigen
                        } label: {
                            HStack(spacing: 8) {
                                Image(systemName: "person.circle")
                                    .foregroundStyle(.secondary)
                                Text(owner.name)
                                    .font(.subheadline)
                            }
                            .padding(.leading, 28)
                        }
                    }
                }
            }
        }
    }
}
```

**Step 2: In loadData() Hunde laden**

```swift
private func loadData() async {
    mediaCount = viewModel.messages.filter { $0.hasMedia }.count
    // Hunde des Kontakts laden
    if let userId = otherMember?.userId {
        do {
            let response = try await APIClient.shared.request(
                Endpoint(path: "/api/admin/customers/\(userId)/dogs"),
                as: DogContactsResponse.self
            )
            dogs = response.dogs
        } catch {
            // Nicht jeder User hat Zugriff auf diesen Endpoint
        }
    }
}
```

**Step 3: dogsSection in body einfügen (zwischen mediaSection und notificationSection)**

**Step 4: Build & Verify, Commit**

```
feat: Hunde + weitere Halter in Kontaktinfo-Seite
```

---

### Task 12: Blockieren / Melden (Backend-Endpoints)

**Files:**
- Modify: `PawCoach/Core/Network/Endpoint+Messaging.swift`

**Step 1: Endpoints hinzufügen**

```swift
static func blockUser(userId: Int) -> Endpoint {
    Endpoint(path: "/api/messages/block/\(userId)", method: "POST")
}

static func unblockUser(userId: Int) -> Endpoint {
    Endpoint(path: "/api/messages/block/\(userId)", method: "DELETE")
}

static func reportUser(userId: Int, reason: String? = nil) -> Endpoint {
    var params: [String: Any] = [:]
    if let reason { params["reason"] = reason }
    return Endpoint(
        path: "/api/messages/report/\(userId)",
        method: "POST",
        body: params.isEmpty ? nil : jsonBody(params)
    )
}
```

**Step 2: ContactInfoView Blockieren/Melden mit echten Endpoints verbinden**

Die bestehenden `blockUser()` und `reportUser()` Funktionen aktualisieren um die neuen Endpoint-Funktionen zu nutzen statt inline Endpoints.

**Step 3: Build & Verify, Commit**

```
feat: Blockieren/Melden Endpoints vorbereitet
```

---

### Task 13: Gemeinsame Kurse in Kontaktinfo

**Files:**
- Modify: `PawCoach/Features/Messaging/ContactInfoView.swift`

**Step 1: SharedCourse Model**

```swift
struct SharedCourse: Identifiable, Decodable {
    let id: Int
    let title: String
    let schedule: String?
}
```

**Step 2: Gemeinsame-Kurse-Sektion**

```swift
@State private var sharedCourses: [SharedCourse] = []

private var coursesSection: some View {
    Section("Gemeinsame Kurse") {
        if sharedCourses.isEmpty {
            Text("Keine gemeinsamen Kurse")
                .foregroundStyle(.secondary)
        } else {
            ForEach(sharedCourses) { course in
                HStack {
                    Image(systemName: "book.circle.fill")
                        .foregroundStyle(Color.pawcoachGreen)
                    VStack(alignment: .leading, spacing: 2) {
                        Text(course.title)
                        if let schedule = course.schedule {
                            Text(schedule)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
            }
        }
    }
}
```

**Step 3: Laden in loadData()**

```swift
// Gemeinsame Kurse laden (falls Endpoint existiert)
// GET /api/messages/shared-courses/<user_id>
```

**Step 4: Build & Verify, Commit**

```
feat: Gemeinsame Kurse in Kontaktinfo anzeigen
```

---

## Zusammenfassung der Dateien

| Datei | Aktion |
|-------|--------|
| `PawCoach/Features/Messaging/MessageDetailView.swift` | Modify (Tasks 1-4, 6, 9) |
| `PawCoach/Features/Messaging/MessagingModels.swift` | Modify (Tasks 4, 7, 9) |
| `PawCoach/Features/Messaging/MessagingViewModel.swift` | Modify (Tasks 6, 7) |
| `PawCoach/Features/Messaging/NewConversationView.swift` | Modify (Task 7) |
| `PawCoach/Features/Messaging/ContactInfoView.swift` | Create (Tasks 6, 8, 11, 13) |
| `PawCoach/Features/Messaging/MuteManager.swift` | Create (Task 5) |
| `PawCoach/Features/Settings/PrivacyVisibilityView.swift` | Create (Task 10) |
| `PawCoach/Features/Settings/SettingsView.swift` | Modify (Task 10) |
| `PawCoach/Core/Network/Endpoint+Messaging.swift` | Modify (Tasks 4, 8, 10, 12) |

## Backend-Endpoints (noch zu erstellen im Flask-Backend)

| Endpoint | Methode | Zweck |
|----------|---------|-------|
| `/api/auth/profile/visibility` | GET/PATCH | Sichtbarkeitseinstellungen |
| `/api/messages/block/<user_id>` | POST/DELETE | Blockieren/Entblocken |
| `/api/messages/report/<user_id>` | POST | User melden |
| `/api/messages/<conv_id>/clear` | DELETE | Chat leeren |
| `/api/messages/shared-courses/<user_id>` | GET | Gemeinsame Kurse |
| Message Model | — | `reply_to_id`, `reply_to_body`, `reply_to_sender_name` Felder |
| User Model | — | `discoverable`, `last_seen`, Sichtbarkeits-Felder |
