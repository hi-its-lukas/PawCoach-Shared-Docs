# Duale Registrierungs-Flows — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Zwei getrennte Registrierungs-Flows: SaaS (Hundeschule gruenden) und Kunden-Registrierung (bestehender Schule beitreten) mit Deep-Link-Support und Admin-Genehmigungsflow.

**Architecture:** Gemeinsamer Einstiegs-Screen `RegisterChoiceView` verzweigt zu bestehendem `RegisterView` (SaaS) und neuem `CustomerRegisterView` (Kunde). Neuer `DeepLinkHandler` verarbeitet Universal Links. Admin-Seite wird um Schul-Code-Verwaltung und Genehmigungs-Tab erweitert.

**Tech Stack:** SwiftUI, Swift 5.9, iOS 17.0+, CoreImage (QR-Code), XcodeGen

**Design-Doc:** `plans/2026-03-12-registration-flows-design.md`

---

### Task 1: Neue Endpoints in Endpoints.swift

**Files:**
- Modify: `PawCoach/Core/Network/Endpoints.swift`

**Step 1: Endpoints hinzufuegen**

Nach dem bestehenden `setPassword`-Endpoint (Zeile 96) folgende Endpoints einfuegen:

```swift
// MARK: - Customer Registration

static func registerCustomer(email: String, password: String, name: String, schoolCode: String) -> Endpoint {
    Endpoint(
        path: "/api/auth/register-customer",
        method: "POST",
        body: jsonBody(["email": email, "password": password, "name": name, "school_code": schoolCode])
    )
}

// MARK: - School Discovery

static func searchSchools(query: String) -> Endpoint {
    Endpoint(
        path: "/api/schools/search",
        queryItems: [URLQueryItem(name: "q", value: query)]
    )
}

static func joinSchool(code: String) -> Endpoint {
    Endpoint(path: "/api/schools/join/\(code)")
}

// MARK: - Pending Registrations (Admin)

static func pendingRegistrations() -> Endpoint {
    Endpoint(path: "/api/admin/pending-registrations")
}

static func approveRegistration(id: Int) -> Endpoint {
    Endpoint(path: "/api/admin/pending-registrations/\(id)/approve", method: "POST")
}

static func rejectRegistration(id: Int, reason: String? = nil) -> Endpoint {
    var dict: [String: Any] = [:]
    if let reason { dict["reason"] = reason }
    return Endpoint(
        path: "/api/admin/pending-registrations/\(id)/reject",
        method: "POST",
        body: dict.isEmpty ? nil : jsonBody(dict)
    )
}

static func regenerateSchoolCode() -> Endpoint {
    Endpoint(path: "/api/admin/settings/regenerate-school-code", method: "POST")
}
```

**Step 2: Build pruefen**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodebuild build -project PawCoach.xcodeproj -scheme PawCoach -destination 'platform=iOS Simulator,name=iPhone 17 Pro Max' 2>&1 | tail -5`
Expected: BUILD SUCCEEDED

**Step 3: Commit**

```bash
git add PawCoach/Core/Network/Endpoints.swift
git commit -m "feat: Endpoints fuer Kunden-Registrierung, Schulsuche und Admin-Genehmigung"
```

---

### Task 2: Response-Models fuer Schulsuche und Registrierung

**Files:**
- Create: `PawCoach/Features/Auth/RegistrationModels.swift`

**Step 1: Models erstellen**

```swift
import Foundation

/// Ergebnis der Schulsuche.
struct SchoolSearchResult: Codable, Identifiable {
    let id: Int
    let name: String
    let city: String?
    let code: String
}

/// Ergebnis der Code-Validierung.
struct SchoolJoinInfo: Codable {
    let id: Int
    let name: String
    let city: String?
    let requiresApproval: Bool

    enum CodingKeys: String, CodingKey {
        case id, name, city
        case requiresApproval = "requires_approval"
    }
}

/// Response der Kunden-Registrierung.
struct CustomerRegistrationResponse: Codable {
    let status: String // "confirmation_sent" oder "pending_approval"
}

/// Pending Registration fuer Admin-Ansicht.
struct PendingRegistration: Codable, Identifiable {
    let id: Int
    let name: String
    let email: String
    let registeredAt: String?

    enum CodingKeys: String, CodingKey {
        case id, name, email
        case registeredAt = "registered_at"
    }
}
```

**Step 2: XcodeGen ausfuehren**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodegen generate`
Expected: `Generated PawCoach project`

**Step 3: Build pruefen**

Run: `xcodebuild build -project PawCoach.xcodeproj -scheme PawCoach -destination 'platform=iOS Simulator,name=iPhone 17 Pro Max' 2>&1 | tail -5`
Expected: BUILD SUCCEEDED

**Step 4: Commit**

```bash
git add PawCoach/Features/Auth/RegistrationModels.swift
git commit -m "feat: Response-Models fuer Schulsuche und Kunden-Registrierung"
```

---

### Task 3: RegisterChoiceView — Auswahlscreen

**Files:**
- Create: `PawCoach/Features/Auth/RegisterChoiceView.swift`
- Modify: `PawCoach/Features/Auth/LoginView.swift:212-218`

**Step 1: RegisterChoiceView erstellen**

```swift
import SwiftUI

/// Auswahlscreen: Hundeschule gruenden oder als Kunde beitreten.
struct RegisterChoiceView: View {
    var prefilledSchoolCode: String?

    var body: some View {
        ZStack {
            Color.pawcoachBackground
                .ignoresSafeArea()

            ScrollView {
                VStack(spacing: 32) {
                    Spacer().frame(height: 40)
                    header
                    choiceCards
                    Spacer()
                }
                .padding(.horizontal, 24)
            }
        }
        .navigationTitle("Registrieren")
        .navigationBarTitleDisplayMode(.inline)
    }

    // MARK: - Header

    private var header: some View {
        VStack(spacing: 16) {
            Image("PawCoachLogo")
                .resizable()
                .scaledToFit()
                .frame(height: 60)
                .accessibilityHidden(true)

            Text("Wie moechtest du starten?")
                .font(.title2)
                .fontWeight(.bold)
                .foregroundStyle(Color.pawcoachText)
        }
    }

    // MARK: - Kacheln

    private var choiceCards: some View {
        VStack(spacing: 16) {
            // Hundeschule gruenden
            NavigationLink {
                RegisterView()
            } label: {
                choiceCard(
                    icon: "building.2.fill",
                    title: "Hundeschule gruenden",
                    subtitle: "Du bist Hundeschul-Betreiber und moechtest PawCoach nutzen.",
                    color: .pawcoachGreen
                )
            }

            // Als Kunde beitreten
            NavigationLink {
                CustomerRegisterView(prefilledSchoolCode: prefilledSchoolCode)
            } label: {
                choiceCard(
                    icon: "person.badge.plus",
                    title: "Als Kunde beitreten",
                    subtitle: "Du moechtest dich bei einer bestehenden Hundeschule anmelden.",
                    color: .pawcoachAccent
                )
            }
        }
    }

    private func choiceCard(icon: String, title: String, subtitle: String, color: Color) -> some View {
        HStack(spacing: 16) {
            Image(systemName: icon)
                .font(.system(size: 32))
                .foregroundStyle(color)
                .frame(width: 50)

            VStack(alignment: .leading, spacing: 4) {
                Text(title)
                    .font(.headline)
                    .foregroundStyle(Color.pawcoachText)
                Text(subtitle)
                    .font(.caption)
                    .foregroundStyle(Color.pawcoachTextSecondary)
                    .multilineTextAlignment(.leading)
            }

            Spacer()

            Image(systemName: "chevron.right")
                .foregroundStyle(Color.pawcoachTextTertiary)
        }
        .padding(20)
        .background(Color.pawcoachCardBackground)
        .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.lg)))
        .shadow(color: .black.opacity(0.06), radius: 12, x: 0, y: 4)
    }
}
```

**Step 2: LoginView anpassen — NavigationLink aendern**

In `PawCoach/Features/Auth/LoginView.swift`, Zeile 212-218 den bestehenden Link zu `RegisterView()` durch `RegisterChoiceView()` ersetzen:

Aendern:
```swift
            NavigationLink {
                RegisterView()
            } label: {
```
Zu:
```swift
            NavigationLink {
                RegisterChoiceView()
            } label: {
```

**Step 3: XcodeGen und Build**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodegen generate && xcodebuild build -project PawCoach.xcodeproj -scheme PawCoach -destination 'platform=iOS Simulator,name=iPhone 17 Pro Max' 2>&1 | tail -5`
Expected: BUILD SUCCEEDED

**Step 4: Commit**

```bash
git add PawCoach/Features/Auth/RegisterChoiceView.swift PawCoach/Features/Auth/LoginView.swift
git commit -m "feat: RegisterChoiceView als Einstieg mit SaaS/Kunden-Auswahl"
```

---

### Task 4: CustomerRegisterView — Schule finden (Step 1)

**Files:**
- Create: `PawCoach/Features/Auth/CustomerRegisterView.swift`

**Step 1: CustomerRegisterView mit Schulsuche erstellen**

```swift
import SwiftUI

/// Kunden-Registrierung: Schule finden → Persoenliche Daten → Bestaetigung.
struct CustomerRegisterView: View {
    var prefilledSchoolCode: String?

    @State private var step: RegistrationStep = .findSchool
    @State private var schoolCode = ""
    @State private var searchQuery = ""
    @State private var searchResults: [SchoolSearchResult] = []
    @State private var selectedSchool: SchoolJoinInfo?
    @State private var isSearching = false
    @State private var isValidating = false
    @State private var errorMessage: String?

    // Step 2 — Persoenliche Daten
    @State private var name = ""
    @State private var email = ""
    @State private var password = ""
    @State private var passwordConfirm = ""
    @State private var isRegistering = false
    @State private var registrationResult: String?
    @State private var showResendButton = false
    @FocusState private var focusedField: Field?

    private let api = APIClient.shared

    private enum RegistrationStep {
        case findSchool
        case personalData
        case success
    }

    private enum Field: Hashable {
        case schoolCode, search, name, email, password, passwordConfirm
    }

    var body: some View {
        ZStack {
            Color.pawcoachBackground
                .ignoresSafeArea()

            ScrollView {
                VStack(spacing: 32) {
                    Spacer().frame(height: 20)

                    switch step {
                    case .findSchool:
                        findSchoolView
                    case .personalData:
                        personalDataView
                    case .success:
                        successView
                    }

                    Spacer()
                }
                .padding(.horizontal, 24)
            }
        }
        .navigationTitle("Kunde registrieren")
        .navigationBarTitleDisplayMode(.inline)
        .task {
            if let code = prefilledSchoolCode {
                schoolCode = code
                await validateSchoolCode()
            }
        }
    }

    // MARK: - Step 1: Schule finden

    private var findSchoolView: some View {
        VStack(spacing: 20) {
            // Header
            VStack(spacing: 8) {
                Image(systemName: "magnifyingglass")
                    .font(.system(size: 40))
                    .foregroundStyle(Color.pawcoachAccent)
                Text("Finde deine Hundeschule")
                    .font(.title3)
                    .fontWeight(.bold)
                    .foregroundStyle(Color.pawcoachText)
                Text("Gib den Schul-Code ein oder suche nach dem Namen.")
                    .font(.subheadline)
                    .foregroundStyle(Color.pawcoachTextSecondary)
                    .multilineTextAlignment(.center)
            }

            // Code-Eingabe
            VStack(alignment: .leading, spacing: 6) {
                Text("Schul-Code")
                    .font(.subheadline)
                    .fontWeight(.medium)
                    .foregroundStyle(Color.pawcoachText)

                HStack(spacing: 8) {
                    TextField("z.B. FIDO-2026", text: $schoolCode)
                        .textInputAutocapitalization(.characters)
                        .autocorrectionDisabled()
                        .focused($focusedField, equals: .schoolCode)
                        .submitLabel(.go)
                        .onSubmit { Task { await validateSchoolCode() } }
                        .padding(14)
                        .background(Color.pawcoachInputBackground)
                        .overlay(
                            RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md))
                                .stroke(Color.pawcoachBorder, lineWidth: 1)
                        )
                        .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))

                    Button(action: { Task { await validateSchoolCode() } }) {
                        if isValidating {
                            ProgressView()
                                .tint(Color.pawcoachAccent)
                        } else {
                            Image(systemName: "arrow.right.circle.fill")
                                .font(.title2)
                                .foregroundStyle(Color.pawcoachAccent)
                        }
                    }
                    .disabled(schoolCode.trimmingCharacters(in: .whitespaces).isEmpty || isValidating)
                }
            }

            // Trennlinie
            HStack {
                Rectangle().frame(height: 1).foregroundStyle(Color.pawcoachBorder)
                Text("oder suchen")
                    .font(.caption)
                    .foregroundStyle(Color.pawcoachTextTertiary)
                Rectangle().frame(height: 1).foregroundStyle(Color.pawcoachBorder)
            }

            // Namenssuche
            VStack(alignment: .leading, spacing: 6) {
                Text("Schulname")
                    .font(.subheadline)
                    .fontWeight(.medium)
                    .foregroundStyle(Color.pawcoachText)

                TextField("Hundeschule suchen...", text: $searchQuery)
                    .autocorrectionDisabled()
                    .focused($focusedField, equals: .search)
                    .submitLabel(.search)
                    .onSubmit { Task { await searchSchools() } }
                    .onChange(of: searchQuery) { _, newValue in
                        if newValue.count >= 2 {
                            Task { await searchSchools() }
                        } else {
                            searchResults = []
                        }
                    }
                    .padding(14)
                    .background(Color.pawcoachInputBackground)
                    .overlay(
                        RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md))
                            .stroke(Color.pawcoachBorder, lineWidth: 1)
                    )
                    .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))
            }

            // Suchergebnisse
            if isSearching {
                ProgressView("Suche...")
            }

            ForEach(searchResults) { school in
                Button {
                    schoolCode = school.code
                    Task { await validateSchoolCode() }
                } label: {
                    HStack {
                        VStack(alignment: .leading, spacing: 2) {
                            Text(school.name)
                                .font(.body)
                                .fontWeight(.medium)
                                .foregroundStyle(Color.pawcoachText)
                            if let city = school.city {
                                Text(city)
                                    .font(.caption)
                                    .foregroundStyle(Color.pawcoachTextSecondary)
                            }
                        }
                        Spacer()
                        Image(systemName: "chevron.right")
                            .foregroundStyle(Color.pawcoachTextTertiary)
                    }
                    .padding(12)
                    .background(Color.pawcoachCardBackground)
                    .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))
                }
            }

            // Ausgewaehlte Schule
            if let school = selectedSchool {
                selectedSchoolCard(school)
            }

            // Fehlermeldung
            if let error = errorMessage {
                HStack(spacing: 8) {
                    Image(systemName: "exclamationmark.triangle.fill")
                        .foregroundStyle(Color.error)
                        .font(.subheadline)
                    Text(error)
                        .font(.callout)
                        .foregroundStyle(Color.error)
                }
                .frame(maxWidth: .infinity, alignment: .leading)
                .padding(12)
                .background(Color.error.opacity(0.08))
                .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.sm)))
            }
        }
        .padding(24)
        .background(Color.pawcoachCardBackground)
        .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.lg)))
        .shadow(color: .black.opacity(0.06), radius: 12, x: 0, y: 4)
    }

    private func selectedSchoolCard(_ school: SchoolJoinInfo) -> some View {
        VStack(spacing: 12) {
            HStack {
                Image(systemName: "checkmark.circle.fill")
                    .foregroundStyle(Color.pawcoachGreen)
                VStack(alignment: .leading, spacing: 2) {
                    Text(school.name)
                        .font(.headline)
                        .foregroundStyle(Color.pawcoachText)
                    if let city = school.city {
                        Text(city)
                            .font(.caption)
                            .foregroundStyle(Color.pawcoachTextSecondary)
                    }
                }
                Spacer()
                Button("Aendern") {
                    selectedSchool = nil
                }
                .font(.caption)
                .foregroundStyle(Color.pawcoachAccent)
            }

            if school.requiresApproval {
                Label("Genehmigung durch die Schule erforderlich", systemImage: "info.circle")
                    .font(.caption)
                    .foregroundStyle(.orange)
            }

            Button(action: { step = .personalData }) {
                Text("Weiter")
                    .fontWeight(.semibold)
                    .frame(maxWidth: .infinity)
                    .frame(height: 50)
            }
            .foregroundStyle(.white)
            .background(Color.pawcoachGreen)
            .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))
        }
        .padding(16)
        .background(Color.pawcoachGreen.opacity(0.08))
        .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))
    }

    // MARK: - Step 2: Persoenliche Daten

    private var personalDataView: some View {
        VStack(spacing: 20) {
            // Header mit Schulinfo
            if let school = selectedSchool {
                HStack {
                    Image(systemName: "building.2.fill")
                        .foregroundStyle(Color.pawcoachAccent)
                    Text(school.name)
                        .font(.subheadline)
                        .fontWeight(.medium)
                    Spacer()
                    Button("Aendern") {
                        step = .findSchool
                    }
                    .font(.caption)
                    .foregroundStyle(Color.pawcoachAccent)
                }
                .padding(12)
                .background(Color.pawcoachAccent.opacity(0.08))
                .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.sm)))
            }

            Text("Deine Daten")
                .font(.title3)
                .fontWeight(.bold)
                .foregroundStyle(Color.pawcoachText)

            // Name
            fieldView(label: "Name", placeholder: "Dein Name", text: $name, field: .name, nextField: .email, contentType: .name)

            // E-Mail
            fieldView(label: "E-Mail", placeholder: "name@beispiel.de", text: $email, field: .email, nextField: .password, contentType: .emailAddress, keyboard: .emailAddress, capitalize: false)

            // Passwort
            VStack(alignment: .leading, spacing: 6) {
                Text("Passwort")
                    .font(.subheadline)
                    .fontWeight(.medium)
                    .foregroundStyle(Color.pawcoachText)

                SecureField("Mindestens 8 Zeichen", text: $password)
                    .textContentType(.newPassword)
                    .focused($focusedField, equals: .password)
                    .submitLabel(.next)
                    .onSubmit { focusedField = .passwordConfirm }
                    .padding(14)
                    .background(Color.pawcoachInputBackground)
                    .overlay(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)).stroke(Color.pawcoachBorder, lineWidth: 1))
                    .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))
            }

            // Passwort bestaetigen
            VStack(alignment: .leading, spacing: 6) {
                Text("Passwort bestaetigen")
                    .font(.subheadline)
                    .fontWeight(.medium)
                    .foregroundStyle(Color.pawcoachText)

                SecureField("Passwort wiederholen", text: $passwordConfirm)
                    .textContentType(.newPassword)
                    .focused($focusedField, equals: .passwordConfirm)
                    .submitLabel(.done)
                    .padding(14)
                    .background(Color.pawcoachInputBackground)
                    .overlay(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)).stroke(Color.pawcoachBorder, lineWidth: 1))
                    .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))
            }

            // Fehlermeldung
            if let error = errorMessage {
                HStack(spacing: 8) {
                    Image(systemName: "exclamationmark.triangle.fill")
                        .foregroundStyle(Color.error)
                        .font(.subheadline)
                    Text(error)
                        .font(.callout)
                        .foregroundStyle(Color.error)
                }
                .frame(maxWidth: .infinity, alignment: .leading)
                .padding(12)
                .background(Color.error.opacity(0.08))
                .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.sm)))
            }

            // Registrieren Button
            Button(action: { Task { await performRegister() } }) {
                Group {
                    if isRegistering {
                        ProgressView().tint(.white)
                    } else {
                        Text("Registrieren")
                            .fontWeight(.semibold)
                    }
                }
                .frame(maxWidth: .infinity)
                .frame(height: 50)
            }
            .foregroundStyle(.white)
            .background((!isPersonalDataValid || isRegistering) ? Color.pawcoachGreen.opacity(0.5) : Color.pawcoachGreen)
            .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))
            .disabled(!isPersonalDataValid || isRegistering)

            // Zurueck
            Button("Zurueck zur Schulauswahl") {
                step = .findSchool
            }
            .font(.subheadline)
            .foregroundStyle(Color.pawcoachAccent)
        }
        .padding(24)
        .background(Color.pawcoachCardBackground)
        .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.lg)))
        .shadow(color: .black.opacity(0.06), radius: 12, x: 0, y: 4)
    }

    private func fieldView(
        label: String, placeholder: String, text: Binding<String>,
        field: Field, nextField: Field?, contentType: UITextContentType,
        keyboard: UIKeyboardType = .default, capitalize: Bool = true
    ) -> some View {
        VStack(alignment: .leading, spacing: 6) {
            Text(label)
                .font(.subheadline)
                .fontWeight(.medium)
                .foregroundStyle(Color.pawcoachText)

            TextField(placeholder, text: text)
                .textContentType(contentType)
                .keyboardType(keyboard)
                .autocorrectionDisabled()
                .textInputAutocapitalization(capitalize ? .words : .never)
                .focused($focusedField, equals: field)
                .submitLabel(nextField != nil ? .next : .done)
                .onSubmit { if let nextField { focusedField = nextField } }
                .padding(14)
                .background(Color.pawcoachInputBackground)
                .overlay(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)).stroke(Color.pawcoachBorder, lineWidth: 1))
                .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))
        }
    }

    // MARK: - Step 3: Erfolg

    private var successView: some View {
        VStack(spacing: 20) {
            Image(systemName: registrationResult == "pending_approval" ? "clock.badge.checkmark.fill" : "envelope.badge.fill")
                .font(.system(size: 48))
                .foregroundStyle(Color.pawcoachGreen)

            Text(registrationResult == "pending_approval" ? "Anmeldung eingereicht" : "Bestaetigungs-E-Mail gesendet")
                .font(.title3)
                .fontWeight(.bold)
                .foregroundStyle(Color.pawcoachText)

            if registrationResult == "pending_approval" {
                Text("Wir haben eine E-Mail an **\(email)** gesendet. Bitte bestaetige deine E-Mail-Adresse. Danach wird deine Anmeldung von der Hundeschule geprueft.")
                    .font(.subheadline)
                    .foregroundStyle(Color.pawcoachTextSecondary)
                    .multilineTextAlignment(.center)
            } else {
                Text("Wir haben eine E-Mail an **\(email)** gesendet. Bitte klicke auf den Link in der E-Mail, um dein Konto zu bestaetigen.")
                    .font(.subheadline)
                    .foregroundStyle(Color.pawcoachTextSecondary)
                    .multilineTextAlignment(.center)
            }

            if showResendButton {
                Button(action: { Task { await resendConfirmation() } }) {
                    Text("Bestaetigungs-E-Mail erneut senden")
                        .fontWeight(.medium)
                        .frame(maxWidth: .infinity)
                        .frame(height: 50)
                }
                .foregroundStyle(Color.pawcoachAccent)
                .background(Color.pawcoachAccentLight)
                .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))
                .overlay(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)).stroke(Color.pawcoachAccent.opacity(0.3), lineWidth: 1))
            }

            NavigationLink {
                LoginView()
            } label: {
                Text("Zum Login")
                    .fontWeight(.semibold)
                    .foregroundStyle(Color.pawcoachAccent)
            }
        }
        .padding(24)
        .background(Color.pawcoachCardBackground)
        .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.lg)))
        .shadow(color: .black.opacity(0.06), radius: 12, x: 0, y: 4)
        .task {
            try? await Task.sleep(for: .seconds(10))
            showResendButton = true
        }
    }

    // MARK: - Validation

    private var isPersonalDataValid: Bool {
        name.isNotBlank &&
        email.isValidEmail &&
        password.count >= 8 &&
        password == passwordConfirm
    }

    // MARK: - API Actions

    private func validateSchoolCode() async {
        let code = schoolCode.trimmingCharacters(in: .whitespaces)
        guard !code.isEmpty else { return }

        isValidating = true
        errorMessage = nil
        defer { isValidating = false }

        do {
            let info = try await api.request(.joinSchool(code: code), as: SchoolJoinInfo.self)
            selectedSchool = info
            searchResults = []
        } catch let error as APIError {
            errorMessage = error.errorDescription ?? "Ungueltiger Schul-Code"
            selectedSchool = nil
        } catch {
            errorMessage = "Verbindungsfehler. Bitte versuche es erneut."
        }
    }

    private func searchSchools() async {
        let query = searchQuery.trimmingCharacters(in: .whitespaces)
        guard query.count >= 2 else { return }

        isSearching = true
        defer { isSearching = false }

        do {
            searchResults = try await api.request(.searchSchools(query: query), as: [SchoolSearchResult].self)
        } catch {
            searchResults = []
        }
    }

    private func performRegister() async {
        guard isPersonalDataValid, let school = selectedSchool else { return }

        isRegistering = true
        errorMessage = nil
        defer { isRegistering = false }

        do {
            let response = try await api.request(
                .registerCustomer(email: email, password: password, name: name, schoolCode: schoolCode),
                as: CustomerRegistrationResponse.self
            )
            registrationResult = response.status
            step = .success
        } catch let error as APIError {
            errorMessage = error.errorDescription ?? "Registrierung fehlgeschlagen"
        } catch {
            errorMessage = "Verbindungsfehler. Bitte versuche es erneut."
        }
    }

    private func resendConfirmation() async {
        do {
            _ = try await api.request(.resendConfirmation(email: email), as: StatusResponse.self)
        } catch let error as APIError {
            errorMessage = error.errorDescription ?? "Erneutes Senden fehlgeschlagen"
        } catch {
            errorMessage = "Verbindungsfehler"
        }
    }
}
```

**Step 2: XcodeGen und Build**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodegen generate && xcodebuild build -project PawCoach.xcodeproj -scheme PawCoach -destination 'platform=iOS Simulator,name=iPhone 17 Pro Max' 2>&1 | tail -5`
Expected: BUILD SUCCEEDED

**Step 3: Commit**

```bash
git add PawCoach/Features/Auth/CustomerRegisterView.swift
git commit -m "feat: CustomerRegisterView mit Schulsuche und Registrierungsflow"
```

---

### Task 5: DeepLinkHandler — Universal Links verarbeiten

**Files:**
- Create: `PawCoach/App/DeepLinkHandler.swift`
- Modify: `PawCoach/App/PawCoachApp.swift`
- Modify: `PawCoach/App/AppState.swift:30-34`

**Step 1: DeepLinkHandler erstellen**

```swift
import Foundation

/// Verarbeitet Universal Links und URL-Schemes.
enum DeepLinkRoute: Equatable {
    case joinSchool(code: String)
    case setPassword(token: String)
    case confirmEmail(token: String)
    case message(conversationId: Int)
    case session(sessionId: Int)
    case booking(bookingId: Int)

    /// Parst eine URL in eine DeepLinkRoute.
    static func from(url: URL) -> DeepLinkRoute? {
        let path = url.pathComponents

        // /join/:code
        if path.count >= 3, path[1] == "join" {
            return .joinSchool(code: path[2])
        }

        // /set-password/:token
        if path.count >= 3, path[1] == "set-password" {
            return .setPassword(token: path[2])
        }

        // /confirm/:token
        if path.count >= 3, path[1] == "confirm" {
            return .confirmEmail(token: path[2])
        }

        // /messages/:id
        if path.count >= 3, path[1] == "messages", let id = Int(path[2]) {
            return .message(conversationId: id)
        }

        // /sessions/:id
        if path.count >= 3, path[1] == "sessions", let id = Int(path[2]) {
            return .session(sessionId: id)
        }

        // /bookings/:id
        if path.count >= 3, path[1] == "bookings", let id = Int(path[2]) {
            return .booking(bookingId: id)
        }

        return nil
    }
}
```

**Step 2: AppState erweitern — DeepLink enum ersetzen**

In `PawCoach/App/AppState.swift`, den bestehenden `DeepLink` enum (Zeilen 30-34) ersetzen und eine Property fuer den Registrierungs-Deep-Link hinzufuegen:

Aendern:
```swift
enum DeepLink: Equatable {
    case message(conversationId: Int)
    case session(sessionId: Int)
    case booking(bookingId: Int)
}
```
Zu:
```swift
typealias DeepLink = DeepLinkRoute
```

Neue Property nach `pendingDeepLink` (ca. Zeile 28):
```swift
var pendingJoinCode: String?
```

**Step 3: PawCoachApp.swift — .onOpenURL hinzufuegen**

In `PawCoachApp.swift`, nach dem bestehenden `.task { ... }` Block, `.onOpenURL` hinzufuegen:

```swift
.onOpenURL { url in
    guard let route = DeepLinkRoute.from(url: url) else { return }
    switch route {
    case .joinSchool(let code):
        appState.pendingJoinCode = code
    case .confirmEmail(let token):
        Task {
            let authManager = AuthManager()
            await authManager.confirmEmail(token: token)
        }
    case .setPassword:
        // Handled by navigation state
        appState.pendingDeepLink = route
    case .message, .session, .booking:
        appState.pendingDeepLink = route
    }
}
```

**Step 4: XcodeGen und Build**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodegen generate && xcodebuild build -project PawCoach.xcodeproj -scheme PawCoach -destination 'platform=iOS Simulator,name=iPhone 17 Pro Max' 2>&1 | tail -5`
Expected: BUILD SUCCEEDED

**Step 5: Commit**

```bash
git add PawCoach/App/DeepLinkHandler.swift PawCoach/App/PawCoachApp.swift PawCoach/App/AppState.swift
git commit -m "feat: DeepLinkHandler fuer Universal Links (join, set-password, confirm)"
```

---

### Task 6: SetPasswordView — Passwort setzen per Einladungstoken

**Files:**
- Create: `PawCoach/Features/Auth/SetPasswordView.swift`

**Step 1: SetPasswordView erstellen**

```swift
import SwiftUI

/// Passwort setzen fuer eingeladene Nutzer (ueber Token-Link).
struct SetPasswordView: View {
    let token: String

    @State private var password = ""
    @State private var passwordConfirm = ""
    @State private var isLoading = false
    @State private var errorMessage: String?
    @State private var success = false
    @FocusState private var focusedField: Field?

    private let api = APIClient.shared

    private enum Field: Hashable {
        case password, passwordConfirm
    }

    var body: some View {
        ZStack {
            Color.pawcoachBackground
                .ignoresSafeArea()

            ScrollView {
                VStack(spacing: 32) {
                    Spacer().frame(height: 40)

                    // Header
                    VStack(spacing: 16) {
                        Image(systemName: "key.fill")
                            .font(.system(size: 48))
                            .foregroundStyle(Color.pawcoachGreen)

                        Text("Passwort festlegen")
                            .font(.title2)
                            .fontWeight(.bold)
                            .foregroundStyle(Color.pawcoachText)

                        Text("Lege ein Passwort fuer deinen Account fest.")
                            .font(.subheadline)
                            .foregroundStyle(Color.pawcoachTextSecondary)
                            .multilineTextAlignment(.center)
                    }

                    if success {
                        // Erfolg
                        VStack(spacing: 16) {
                            Image(systemName: "checkmark.circle.fill")
                                .font(.system(size: 48))
                                .foregroundStyle(Color.pawcoachGreen)
                            Text("Passwort gespeichert!")
                                .font(.headline)
                            Text("Du kannst dich jetzt anmelden.")
                                .font(.subheadline)
                                .foregroundStyle(Color.pawcoachTextSecondary)
                            NavigationLink {
                                LoginView()
                            } label: {
                                Text("Zum Login")
                                    .fontWeight(.semibold)
                                    .foregroundStyle(Color.pawcoachAccent)
                            }
                        }
                        .padding(24)
                        .background(Color.pawcoachCardBackground)
                        .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.lg)))
                        .shadow(color: .black.opacity(0.06), radius: 12, x: 0, y: 4)
                    } else {
                        // Formular
                        VStack(spacing: 20) {
                            VStack(alignment: .leading, spacing: 6) {
                                Text("Neues Passwort")
                                    .font(.subheadline)
                                    .fontWeight(.medium)
                                SecureField("Mindestens 8 Zeichen", text: $password)
                                    .textContentType(.newPassword)
                                    .focused($focusedField, equals: .password)
                                    .submitLabel(.next)
                                    .onSubmit { focusedField = .passwordConfirm }
                                    .padding(14)
                                    .background(Color.pawcoachInputBackground)
                                    .overlay(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)).stroke(Color.pawcoachBorder, lineWidth: 1))
                                    .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))
                            }

                            VStack(alignment: .leading, spacing: 6) {
                                Text("Passwort bestaetigen")
                                    .font(.subheadline)
                                    .fontWeight(.medium)
                                SecureField("Passwort wiederholen", text: $passwordConfirm)
                                    .textContentType(.newPassword)
                                    .focused($focusedField, equals: .passwordConfirm)
                                    .submitLabel(.done)
                                    .padding(14)
                                    .background(Color.pawcoachInputBackground)
                                    .overlay(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)).stroke(Color.pawcoachBorder, lineWidth: 1))
                                    .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))
                            }

                            if let error = errorMessage {
                                HStack(spacing: 8) {
                                    Image(systemName: "exclamationmark.triangle.fill")
                                        .foregroundStyle(Color.error)
                                    Text(error)
                                        .font(.callout)
                                        .foregroundStyle(Color.error)
                                }
                                .frame(maxWidth: .infinity, alignment: .leading)
                                .padding(12)
                                .background(Color.error.opacity(0.08))
                                .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.sm)))
                            }

                            Button(action: { Task { await setPassword() } }) {
                                Group {
                                    if isLoading {
                                        ProgressView().tint(.white)
                                    } else {
                                        Text("Passwort speichern")
                                            .fontWeight(.semibold)
                                    }
                                }
                                .frame(maxWidth: .infinity)
                                .frame(height: 50)
                            }
                            .foregroundStyle(.white)
                            .background((!isFormValid || isLoading) ? Color.pawcoachGreen.opacity(0.5) : Color.pawcoachGreen)
                            .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))
                            .disabled(!isFormValid || isLoading)
                        }
                        .padding(24)
                        .background(Color.pawcoachCardBackground)
                        .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.lg)))
                        .shadow(color: .black.opacity(0.06), radius: 12, x: 0, y: 4)
                    }

                    Spacer()
                }
                .padding(.horizontal, 24)
            }
        }
        .navigationTitle("Passwort festlegen")
        .navigationBarTitleDisplayMode(.inline)
    }

    private var isFormValid: Bool {
        password.count >= 8 && password == passwordConfirm
    }

    private func setPassword() async {
        guard isFormValid else { return }
        guard password == passwordConfirm else {
            errorMessage = "Die Passwoerter stimmen nicht ueberein."
            return
        }

        isLoading = true
        errorMessage = nil
        defer { isLoading = false }

        do {
            _ = try await api.request(.setPassword(token: token, password: password), as: StatusResponse.self)
            success = true
        } catch let error as APIError {
            if error.errorDescription?.contains("abgelaufen") == true || error.errorDescription?.contains("expired") == true {
                errorMessage = "Dieser Link ist abgelaufen. Bitte fordere einen neuen an."
            } else {
                errorMessage = error.errorDescription ?? "Passwort setzen fehlgeschlagen"
            }
        } catch {
            errorMessage = "Verbindungsfehler. Bitte versuche es erneut."
        }
    }
}
```

**Step 2: XcodeGen und Build**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodegen generate && xcodebuild build -project PawCoach.xcodeproj -scheme PawCoach -destination 'platform=iOS Simulator,name=iPhone 17 Pro Max' 2>&1 | tail -5`
Expected: BUILD SUCCEEDED

**Step 3: Commit**

```bash
git add PawCoach/Features/Auth/SetPasswordView.swift
git commit -m "feat: SetPasswordView fuer Einladungs-Token-Flow"
```

---

### Task 7: Admin — Schul-Code-Verwaltung in SchoolSettingsView

**Files:**
- Modify: `PawCoach/Features/Admin/SchoolSettingsView.swift`
- Modify: `PawCoach/Features/Admin/AdminViewModel.swift`

**Step 1: AdminViewModel erweitern**

Folgende Properties und Funktionen zum AdminViewModel hinzufuegen:

```swift
// Properties
var schoolCode: String = ""
var customerSelfRegistration: String = "auto_approve"
var selfRegistrationEnabled: Bool = true
var pendingRegistrations: [PendingRegistration] = []

// Schul-Code regenerieren
func regenerateSchoolCode() async {
    isLoading = true
    defer { isLoading = false }

    do {
        struct CodeResponse: Codable { let schoolCode: String
            enum CodingKeys: String, CodingKey { case schoolCode = "school_code" }
        }
        let response = try await api.request(.regenerateSchoolCode(), as: CodeResponse.self)
        schoolCode = response.schoolCode
    } catch {
        errorMessage = "Code-Generierung fehlgeschlagen"
    }
}

// Pending Registrations laden
func loadPendingRegistrations() async {
    do {
        pendingRegistrations = try await api.request(.pendingRegistrations(), as: [PendingRegistration].self)
    } catch {
        errorMessage = "Beitrittsanfragen laden fehlgeschlagen"
    }
}

// Registrierung genehmigen
func approveRegistration(id: Int) async {
    do {
        _ = try await api.request(.approveRegistration(id: id), as: StatusResponse.self)
        pendingRegistrations.removeAll { $0.id == id }
    } catch {
        errorMessage = "Genehmigung fehlgeschlagen"
    }
}

// Registrierung ablehnen
func rejectRegistration(id: Int, reason: String? = nil) async {
    do {
        _ = try await api.request(.rejectRegistration(id: id, reason: reason), as: StatusResponse.self)
        pendingRegistrations.removeAll { $0.id == id }
    } catch {
        errorMessage = "Ablehnung fehlgeschlagen"
    }
}
```

**Step 2: SchoolSettingsView — Schul-Code-Sektion hinzufuegen**

Neue Section in der bestehenden Form/List hinzufuegen:

```swift
Section("Kunden-Registrierung") {
    // Schul-Code
    LabeledContent("Schul-Code") {
        HStack {
            Text(viewModel.schoolCode.isEmpty ? "–" : viewModel.schoolCode)
                .font(.body.monospaced())
                .fontWeight(.semibold)
            Button {
                UIPasteboard.general.string = viewModel.schoolCode
            } label: {
                Image(systemName: "doc.on.doc")
                    .font(.caption)
            }
        }
    }

    // QR-Code
    if !viewModel.schoolCode.isEmpty {
        NavigationLink {
            SchoolQRCodeView(code: viewModel.schoolCode)
        } label: {
            Label("QR-Code anzeigen", systemImage: "qrcode")
        }
    }

    // Regenerieren
    Button("Neuen Code generieren") {
        Task { await viewModel.regenerateSchoolCode() }
    }

    // Selbstregistrierung an/aus
    Toggle("Selbstregistrierung erlauben", isOn: $viewModel.selfRegistrationEnabled)

    // Genehmigungsmodus
    if viewModel.selfRegistrationEnabled {
        Picker("Genehmigung", selection: $viewModel.customerSelfRegistration) {
            Text("Sofort freischalten").tag("auto_approve")
            Text("Genehmigung erforderlich").tag("require_approval")
        }
    }
}
```

**Step 3: Build und Commit**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodebuild build -project PawCoach.xcodeproj -scheme PawCoach -destination 'platform=iOS Simulator,name=iPhone 17 Pro Max' 2>&1 | tail -5`
Expected: BUILD SUCCEEDED

```bash
git add PawCoach/Features/Admin/SchoolSettingsView.swift PawCoach/Features/Admin/AdminViewModel.swift
git commit -m "feat: Schul-Code-Verwaltung und Selbstregistrierungs-Config in Admin-Settings"
```

---

### Task 8: SchoolQRCodeView — QR-Code-Anzeige

**Files:**
- Create: `PawCoach/Features/Admin/SchoolQRCodeView.swift`

**Step 1: QR-Code-View erstellen**

```swift
import SwiftUI
import CoreImage.CIFilterBuiltins

/// Zeigt QR-Code fuer den Schul-Beitrittscode.
struct SchoolQRCodeView: View {
    let code: String

    var body: some View {
        VStack(spacing: 24) {
            Text("Kunden einladen")
                .font(.title2)
                .fontWeight(.bold)

            Text("Lasse deine Kunden diesen QR-Code scannen, um sich bei deiner Hundeschule zu registrieren.")
                .font(.subheadline)
                .foregroundStyle(.secondary)
                .multilineTextAlignment(.center)
                .padding(.horizontal)

            if let image = generateQRCode(from: "https://app.pawcoach.de/join/\(code)") {
                Image(uiImage: image)
                    .interpolation(.none)
                    .resizable()
                    .scaledToFit()
                    .frame(width: 250, height: 250)
                    .padding()
                    .background(.white)
                    .clipShape(RoundedRectangle(cornerRadius: 16))
                    .shadow(color: .black.opacity(0.1), radius: 8)
            }

            Text(code)
                .font(.title3.monospaced())
                .fontWeight(.bold)
                .foregroundStyle(Color.pawcoachGreen)

            // Teilen-Button
            ShareLink(
                item: "https://app.pawcoach.de/join/\(code)",
                subject: Text("PawCoach Einladung"),
                message: Text("Registriere dich bei unserer Hundeschule mit diesem Link:")
            ) {
                Label("Link teilen", systemImage: "square.and.arrow.up")
                    .fontWeight(.medium)
                    .frame(maxWidth: .infinity)
                    .frame(height: 50)
            }
            .foregroundStyle(.white)
            .background(Color.pawcoachGreen)
            .clipShape(RoundedRectangle(cornerRadius: CGFloat(Theme.Radius.md)))
            .padding(.horizontal, 24)

            Spacer()
        }
        .padding(.top, 24)
        .navigationTitle("QR-Code")
        .navigationBarTitleDisplayMode(.inline)
    }

    private func generateQRCode(from string: String) -> UIImage? {
        let context = CIContext()
        let filter = CIFilter.qrCodeGenerator()
        filter.message = Data(string.utf8)
        filter.correctionLevel = "M"

        guard let ciImage = filter.outputImage else { return nil }
        let transform = CGAffineTransform(scaleX: 10, y: 10)
        let scaledImage = ciImage.transformed(by: transform)

        guard let cgImage = context.createCGImage(scaledImage, from: scaledImage.extent) else { return nil }
        return UIImage(cgImage: cgImage)
    }
}
```

**Step 2: XcodeGen und Build**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodegen generate && xcodebuild build -project PawCoach.xcodeproj -scheme PawCoach -destination 'platform=iOS Simulator,name=iPhone 17 Pro Max' 2>&1 | tail -5`
Expected: BUILD SUCCEEDED

**Step 3: Commit**

```bash
git add PawCoach/Features/Admin/SchoolQRCodeView.swift
git commit -m "feat: SchoolQRCodeView mit QR-Code-Generierung und Share-Link"
```

---

### Task 9: Admin — Beitrittsanfragen-Tab in AdminRequestsView

**Files:**
- Modify: `PawCoach/Features/Admin/AdminRequestsView.swift`

**Step 1: Neuen Tab "Beitrittsanfragen" hinzufuegen**

Einen 5. Tab zum bestehenden TabView/Picker in `AdminRequestsView` hinzufuegen:

```swift
// Neuer Tab-Case (zum bestehenden enum hinzufuegen)
case registrations

// Tab-Label
Text("Beitritte").tag(RequestTab.registrations)

// Tab-Content
case .registrations:
    if viewModel.pendingRegistrations.isEmpty {
        ContentUnavailableView(
            "Keine Beitrittsanfragen",
            systemImage: "person.badge.clock",
            description: Text("Neue Kunden-Anmeldungen erscheinen hier.")
        )
    } else {
        ForEach(viewModel.pendingRegistrations) { reg in
            HStack {
                VStack(alignment: .leading, spacing: 4) {
                    Text(reg.name)
                        .font(.body)
                        .fontWeight(.medium)
                    Text(reg.email)
                        .font(.caption)
                        .foregroundStyle(.secondary)
                    if let date = reg.registeredAt {
                        Text(date)
                            .font(.caption2)
                            .foregroundStyle(.secondary)
                    }
                }

                Spacer()

                Button {
                    Task { await viewModel.approveRegistration(id: reg.id) }
                } label: {
                    Image(systemName: "checkmark.circle.fill")
                        .foregroundStyle(Color.pawcoachGreen)
                        .font(.title2)
                }

                Button {
                    Task { await viewModel.rejectRegistration(id: reg.id) }
                } label: {
                    Image(systemName: "xmark.circle.fill")
                        .foregroundStyle(Color.error)
                        .font(.title2)
                }
            }
        }
    }
```

Beim `.task` des Views auch `loadPendingRegistrations()` aufrufen.

**Step 2: Build und Commit**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodebuild build -project PawCoach.xcodeproj -scheme PawCoach -destination 'platform=iOS Simulator,name=iPhone 17 Pro Max' 2>&1 | tail -5`
Expected: BUILD SUCCEEDED

```bash
git add PawCoach/Features/Admin/AdminRequestsView.swift
git commit -m "feat: Beitrittsanfragen-Tab in AdminRequestsView mit Approve/Reject"
```

---

### Task 10: RegisterView — Untertitel hinzufuegen

**Files:**
- Modify: `PawCoach/Features/Auth/RegisterView.swift:58-61`

**Step 1: Header anpassen**

In `RegisterView.swift`, den Header (Zeile 58-61) erweitern:

Aendern:
```swift
            Text("Konto erstellen")
                .font(.title2)
                .fontWeight(.bold)
                .foregroundStyle(Color.pawcoachText)
```
Zu:
```swift
            Text("Hundeschule registrieren")
                .font(.title2)
                .fontWeight(.bold)
                .foregroundStyle(Color.pawcoachText)

            Text("Du erstellst eine neue Hundeschule auf PawCoach.")
                .font(.subheadline)
                .foregroundStyle(Color.pawcoachTextSecondary)
                .multilineTextAlignment(.center)
```

**Step 2: Build und Commit**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodebuild build -project PawCoach.xcodeproj -scheme PawCoach -destination 'platform=iOS Simulator,name=iPhone 17 Pro Max' 2>&1 | tail -5`
Expected: BUILD SUCCEEDED

```bash
git add PawCoach/Features/Auth/RegisterView.swift
git commit -m "fix: RegisterView Header klarstellen — Hundeschule registrieren"
```

---

### Task 11: Integration — Deep-Link-Navigation in ContentView/LoginView

**Files:**
- Modify: `PawCoach/Features/Auth/LoginView.swift`

**Step 1: pendingJoinCode weiterleiten**

In `LoginView.swift`, den `RegisterChoiceView`-NavigationLink (Task 3) so erweitern, dass `pendingJoinCode` weitergegeben wird:

```swift
NavigationLink {
    RegisterChoiceView(prefilledSchoolCode: appState.pendingJoinCode)
} label: {
    Text("Noch kein Konto? **Registrieren**")
        .font(.subheadline)
        .foregroundStyle(Color.pawcoachAccent)
}
```

Wenn `pendingJoinCode` gesetzt ist, sollte die App automatisch zum Registrierungs-Screen navigieren. Dafuer einen `.onAppear`-Modifier hinzufuegen:

```swift
@State private var showRegister = false

// Im body, nach dem NavigationStack:
.navigationDestination(isPresented: $showRegister) {
    RegisterChoiceView(prefilledSchoolCode: appState.pendingJoinCode)
}
.onAppear {
    if appState.pendingJoinCode != nil {
        showRegister = true
    }
}
```

**Step 2: Build und Commit**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodebuild build -project PawCoach.xcodeproj -scheme PawCoach -destination 'platform=iOS Simulator,name=iPhone 17 Pro Max' 2>&1 | tail -5`
Expected: BUILD SUCCEEDED

```bash
git add PawCoach/Features/Auth/LoginView.swift
git commit -m "feat: Deep-Link-Navigation — pendingJoinCode an RegisterChoiceView weiterleiten"
```

---

### Task 12: Finaler Build und Gesamttest

**Step 1: XcodeGen ausfuehren**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodegen generate`

**Step 2: Clean Build**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodebuild clean build -project PawCoach.xcodeproj -scheme PawCoach -destination 'platform=iOS Simulator,name=iPhone 17 Pro Max' 2>&1 | tail -5`
Expected: BUILD SUCCEEDED

**Step 3: Unit Tests ausfuehren**

Run: `cd /Users/luhengl/Documents/GitHub/PawCoach-iOS && xcodebuild test -project PawCoach.xcodeproj -scheme PawCoach -destination 'platform=iOS Simulator,name=iPhone 17 Pro Max' 2>&1 | tail -10`
Expected: TEST SUCCEEDED

**Step 4: Finaler Commit (falls noetig)**

```bash
git add -A
git commit -m "chore: XcodeGen-Projekt aktualisiert fuer Registrierungs-Flows"
```
