# Cross-Platform Sync-Strategie

**Stand:** 2026-03-15
**Ziel:** Sicherstellen, dass Backend, Web-Frontend und iOS-App immer den gleichen technischen Feature-Stand haben.

---

## 1. Problem

Drei unabhaengige Codebases (Backend/Flask, Web/Jinja2+JS, iOS/Swift) entwickeln sich unterschiedlich schnell. Ohne Prozess entstehen Feature-Luecken, inkompatible API-Aenderungen und doppelte Arbeit.

---

## 2. Strategie: Feature-Spec als Single Source of Truth

### 2.1 Feature-Spec-Dokumente

Jede Plattform pflegt ein `docs/CROSS-PLATFORM-FEATURE-SPEC.md`:

| Repo | Datei | Verantwortlich |
|------|-------|----------------|
| Dog-School-Manager (Backend+Web) | `docs/CROSS-PLATFORM-FEATURE-SPEC.md` | Backend-Team |
| PawCoach-iOS | `docs/CROSS-PLATFORM-FEATURE-SPEC.md` | iOS-Entwickler |
| (ggf. Android) | `docs/CROSS-PLATFORM-FEATURE-SPEC.md` | Android-Team |

### 2.2 API-Contract als Schnittstelle

Das Backend definiert die API-Contracts. Alle Clients (Web, iOS, Android) implementieren gegen diese Contracts.

**Regel:** Kein Client darf eine API-Aenderung erfordern, die nicht zuerst im Backend implementiert und dokumentiert wurde.

---

## 3. Prozesse

### 3.1 Feature-Entwicklung (Neues Feature)

```
1. Feature-Request → Design-Dokument (docs/plans/)
2. API-Contract definieren (Endpoints, Request/Response-Schema)
3. Backend implementieren + Tests
4. CROSS-PLATFORM-FEATURE-SPEC.md in Backend-Repo aktualisieren
5. iOS/Web gleichzeitig oder nacheinander implementieren
6. Abgleich-Check: Alle Specs vergleichen
```

### 3.2 API-Aenderungen (Breaking Changes)

**Regel:** Breaking Changes nur ueber API-Versionierung.

```
1. Neue API-Version erstellen (z.B. /api/v2/...)
2. Alte Version fuer mind. 3 Monate beibehalten
3. Deprecation-Header in Responses setzen
4. Alle Clients muessen vor Abschaltung migriert sein
```

### 3.3 Woechentlicher Sync-Check (empfohlen)

**Automatisiert per CI-Job oder manuell:**

1. Feature-Specs aller Repos abrufen
2. Endpoints vergleichen (welche fehlen wo?)
3. Delta-Report generieren
4. In Sprint-Planning priorisieren

---

## 4. Technische Massnahmen

### 4.1 API-Schema-Validierung

**Empfehlung:** OpenAPI/Swagger-Spec fuer alle API-Endpoints.

```yaml
# Beispiel: openapi.yaml (Auszug)
paths:
  /api/messages/{id}:
    get:
      summary: Nachrichten einer Konversation laden
      responses:
        200:
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ConversationDetail'
```

**Vorteile:**
- Automatische Client-Code-Generierung (Swift, TypeScript)
- Schema-Validierung in Tests
- Dokumentation immer aktuell

**Umsetzung (Schritt 1 — niedrige Aufwandsstufe):**
- Flask-Smorest oder Connexion fuer automatische OpenAPI-Generierung
- Oder manuell gepflegte `openapi.yaml` im Repo-Root

### 4.2 Contract-Tests

Tests die sicherstellen, dass API-Responses dem dokumentierten Schema entsprechen:

```python
# tests/test_api_contracts.py
def test_conversation_response_schema(logged_in_client, conversation):
    resp = logged_in_client.get(f"/api/messages/{conversation.id}")
    data = resp.get_json()
    # Pflichtfelder pruefen
    assert "id" in data
    assert "conv_type" in data
    assert "display_name" in data
    assert "last_seen" in data  # Neues Feld
    assert "is_muted" in data   # Neues Feld
    assert "is_blocked" in data # Neues Feld
```

### 4.3 Feature-Flags

Fuer Features die noch nicht auf allen Plattformen verfuegbar sind:

```python
# Backend: Feature-Flag pruefen
FEATURE_FLAGS = {
    "delivery_status": False,  # delivered_at/read_at noch nicht implementiert
    "message_search": False,   # Serverseitige Suche noch nicht da
}
```

API-Endpoint `/api/features` liefert aktive Feature-Flags an Clients.

### 4.4 Deprecation-Tracking

Datei `docs/API-DEPRECATIONS.md` trackt veraltete Endpoints:

```markdown
| Endpoint | Deprecated seit | Entfernung geplant | Ersatz |
|----------|----------------|-------------------|--------|
| (noch keine) | - | - | - |
```

---

## 5. Kommunikations-Artefakte

### 5.1 CHANGELOG pro Release

Jedes Repo pflegt ein `CHANGELOG.md` (Keep a Changelog Format):

```markdown
## [1.2.0] - 2026-03-15
### Added
- Reply/Quote fuer Nachrichten (reply_to_id)
- Block/Unblock/Report User
- ProfileVisibility Einstellungen
### Changed
- Contacts-Endpoint liefert jetzt dogs-Sektion
### Fixed
- Direktnachrichten an blockierte User werden verhindert
```

### 5.2 Release-Notes-Template

Bei jedem Release:
1. **Backend-Aenderungen:** Neue/geaenderte Endpoints
2. **Neue DB-Felder:** Migrationen die gelaufen sind
3. **Breaking Changes:** Was muss in Clients angepasst werden
4. **Feature-Flags:** Was ist neu verfuegbar / was wurde entfernt

---

## 6. Abgleich-Checkliste (bei jedem Release)

```
╔══════════════════════════════════════════════════════════════╗
║  CROSS-PLATFORM SYNC-CHECK                                  ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  □ Feature-Spec im eigenen Repo aktualisiert?               ║
║  □ Neue API-Endpoints in allen Specs dokumentiert?          ║
║  □ Response-Schema-Aenderungen an alle Teams kommuniziert?  ║
║  □ Contract-Tests fuer neue Endpoints geschrieben?          ║
║  □ Feature-Flags fuer unfertige Cross-Platform-Features?    ║
║  □ CHANGELOG aktualisiert?                                  ║
║  □ Breaking Changes mit Deprecation-Zeitraum versehen?      ║
║  □ Delta zwischen iOS-Spec und Backend-Spec geprueft?       ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 7. Priorisierte Umsetzungsschritte

| # | Massnahme | Aufwand | Wirkung | Empfehlung |
|---|-----------|---------|---------|------------|
| 1 | Feature-Specs in beiden Repos pflegen | Niedrig | Hoch | **Sofort starten** |
| 2 | CHANGELOG einfuehren | Niedrig | Mittel | Sofort starten |
| 3 | Contract-Tests schreiben | Mittel | Hoch | Naechster Sprint |
| 4 | API-Deprecation-Tracking | Niedrig | Mittel | Naechster Sprint |
| 5 | OpenAPI-Spec generieren | Mittel | Hoch | Mittelfristig |
| 6 | Feature-Flags Endpoint | Niedrig | Mittel | Bei Bedarf |
| 7 | Automatisierter Spec-Diff CI-Job | Hoch | Hoch | Langfristig |

---

## 8. Aktueller Delta-Status (2026-03-16)

### Im Backend vorhanden, in iOS noch nicht genutzt

| Feature | Status |
|---------|--------|
| Socket.IO Real-Time Events | iOS pollt, kein WebSocket-Client |
| Serverseitige Chat-Suche | iOS nutzt lokale Suche (Server-Endpoint vorbereitet) |

### Plattform-spezifisch (kein Delta)

| Feature | Plattform | Bemerkung |
|---------|-----------|-----------|
| Offline-Modus / PendingAction Queue | iOS-only | Native SwiftData + NWPathMonitor |
| App-Sperre (Face ID / Touch ID) | iOS-only | Native Biometrie |
| Karten-Vorschau (Standort-Nachrichten) | iOS-only | Tap oeffnet Apple Maps |
| SaaS-Billing / Stripe-Checkout | Web-only | Web-only Checkout-Flow |
| Oeffentliche Schulseiten (SEO) | Web-only | Landingpages |
| E-Mail-Vorlagen-Editor (WYSIWYG) | Web-only | Admin-Bereich |

### Bekannter Backend-Bug

| Problem | Beschreibung |
|---------|-------------|
| `/api/messages/{id}/location` blockiert bei aktiver Live-Session | Statischer Standort und Live-Location muessen unabhaengig funktionieren |

### Paritaet erreicht

| Feature | Seit |
|---------|------|
| Reply/Quote (reply_to_id) | 2026-03-15 |
| Block/Unblock | 2026-03-15 |
| Report | 2026-03-15 |
| Clear Chat | 2026-03-15 |
| Mute/Unmute (serverseitig) | 2026-03-16 |
| ProfileVisibility | 2026-03-15 |
| Last Seen | 2026-03-15 |
| Shared Courses | 2026-03-15 |
| Kontaktinfo-Panel | 2026-03-15 |
| Polls | 2026-03-15 |
| Live Location (inkl. Beenden-Button) | 2026-03-16 |
| File Upload | 2026-03-15 |
| Reactions | 2026-03-15 |
| Message Edit/Delete | 2026-03-15 |
| Archive/Unarchive | 2026-03-15 |
| Groups & Broadcast | 2026-03-15 |
| Contacts: dogs-Sektion | 2026-03-16 |
| Zustellstatus (delivered_at/read_at) | 2026-03-15 (Backend), 2026-03-16 (iOS) |
| Nachrichteninfo (GET /info/{id}) | 2026-03-15 (Backend), 2026-03-16 (iOS Swipe-Left) |
| Chat-Suche (Volltextsuche) | 2026-03-15 (Backend+Web), iOS lokal |
| Medien/Links/Doks-Zaehler | 2026-03-15 |
| Sichtbarkeitsfilter (last_name_full) | 2026-03-15 |
| Hunde-Co-Owner in Kontaktliste | 2026-03-15 |
| Push-Notifications fuer Nachrichten | 2026-03-15 |
| Web-UI Sichtbarkeitseinstellungen | 2026-03-15 |
| isMuted/isBlocked/isDeleted Felder | 2026-03-16 |
| Standort als Karten-Vorschau | 2026-03-16 (iOS) |
