# Gesamt-Plan: Alle Feature-Lücken schliessen

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Alle identifizierten Lücken zwischen iOS-App, Backend-API und Web-Frontend schliessen.

**Quellen:** Feature-Audit vom 2026-03-14 (Web ↔ Backend, iOS ↔ Backend)

**Tech Stack:** Flask, Marshmallow, Flask-Limiter, SQLAlchemy, Jinja2, TailwindCSS, Vanilla JS, Socket.IO, Leaflet.js, Stripe.js, Swift/SwiftUI

---

## Uebersicht nach Prioritaet

### BLOCK A — iOS-kritische Backend-Lücken (sofort)
| Task | Beschreibung | Aufwand |
|------|-------------|---------|
| A1 | Pfad-Alias `/api/schools/join/{code}` | klein |
| A2 | `GET /api/admin/pending-registrations` (Liste) | mittel |
| A3 | `POST /api/admin/pending-registrations/{id}/reject` | klein |

### BLOCK B — Fehlende REST APIs (Web-only → API)
| Task | Beschreibung | Aufwand |
|------|-------------|---------|
| B1 | Setup Wizard API | klein |
| B2 | Accounting Dashboard API | mittel |
| B3 | Trainer Availability Admin API | mittel |
| B4 | Manual Customer Booking API | gross |
| B5 | Stripe Connect Onboarding API | mittel |
| B6 | Price Management Platform Admin API | mittel |

### BLOCK C — Fehlende Web-UIs (API-only → UI)
| Task | Beschreibung | Aufwand |
|------|-------------|---------|
| C1 | Poll-UI im Chat | gross |
| C2 | Live-Location-Karte im Chat | gross |
| C3 | Payment-Methods-Verwaltungsseite | mittel |
| C4 | Kontaktvorschlaege im Compose-Modal | klein |

### BLOCK D — iOS-Stubs vervollstaendigen
| Task | Beschreibung | Aufwand |
|------|-------------|---------|
| D1 | Stripe PaymentSheet Integration (iOS) | mittel |
| D2 | Passkey-Verwaltung UI (iOS) | mittel |

---

## BLOCK A — iOS-kritische Backend-Lücken

### Task A1: Pfad-Alias `/api/schools/join/{code}`

**Problem:** iOS ruft `GET /api/schools/join/{code}` auf, Backend hat `GET /api/public/schools/by-code/{code}`.

**iOS-Erwartung** (aus `RegistrationModels.swift`):
```json
{"id": 1, "name": "Schule", "city": "Berlin", "requires_approval": true}
```

**Files:**
- Modify: `blueprints/api_public.py` — Alias-Route hinzufuegen
- Test: `tests/test_api_public.py`

**Step 1: Failing Test**

```python
# In tests/test_api_public.py
def test_school_join_alias(client, active_school):
    """iOS-kompatibler Pfad /api/schools/join/{code} muss funktionieren."""
    resp = client.get(f"/api/schools/join/{active_school.join_code}")
    assert resp.status_code == 200
    data = resp.get_json()
    assert data["name"] == active_school.name
    assert "requires_approval" in data
```

**Step 2: Implementierung**

```python
@api_public_bp.route("/schools/join/<code>")
@limiter.limit("30 per minute")
def school_by_join_code(code: str):
    """Alias fuer /schools/by-code/{code} — iOS-Kompatibilitaet."""
    return school_by_code(code)
```

**Hinweis:** Die bestehende `school_by_code()`-Response muss `requires_approval` enthalten.
Pruefen ob das Feld bereits in der Response ist, sonst hinzufuegen:

```python
# In der bestehenden school_by_code-Response ergaenzen:
"requires_approval": school.customer_approval_required,
```

**Step 3: Commit**
```bash
git add blueprints/api_public.py tests/test_api_public.py
git commit -m "feat: Pfad-Alias /api/schools/join fuer iOS-Kompatibilitaet"
```

---

### Task A2: Pending-Registrations Liste

**Problem:** iOS ruft `GET /api/admin/pending-registrations` auf. Backend hat nur `PUT /api/admin/users/{id}/approve` in `api_admin/users.py:424`, aber keinen Listen-Endpoint fuer ausstehende Registrierungen.

**iOS-Erwartung** (aus `RegistrationModels.swift`):
```json
[
  {"id": 1, "name": "Max Mustermann", "email": "max@example.com", "registered_at": "2026-03-14T10:00:00"}
]
```

**Files:**
- Modify: `blueprints/api_admin/users.py` — Neuen Endpoint hinzufuegen
- Test: `tests/test_customer_approval.py` — Test ergaenzen

**Step 1: Failing Test**

```python
# In tests/test_customer_approval.py
def test_list_pending_registrations(client, approval_required_school):
    """GET /api/admin/pending-registrations muss ausstehende Kunden auflisten."""
    school, admin = approval_required_school
    token = login_api_user(client, admin)
    resp = client.get("/api/admin/pending-registrations",
                      headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 200
    data = resp.get_json()
    assert isinstance(data, list)


def test_list_pending_only_own_school(client, approval_required_school, other_school):
    """Admin darf nur Pending-Registrierungen seiner Schule sehen."""
    school, admin = approval_required_school
    other, other_admin = other_school
    token = login_api_user(client, other_admin)
    resp = client.get("/api/admin/pending-registrations",
                      headers={"Authorization": f"Bearer {token}"})
    data = resp.get_json()
    # Keine Kunden der anderen Schule
    for reg in data:
        assert reg.get("school_id", other.id) == other.id
```

**Step 2: Implementierung**

```python
# In blueprints/api_admin/users.py
@api_admin_bp.route("/pending-registrations", methods=["GET"])
@jwt_or_session_required
@limiter.limit("60 per hour")
def list_pending_registrations():
    """Listet alle Kunden mit approved=False fuer die aktuelle Schule."""
    user, err = _require_admin()
    if err:
        return err
    school_id = _get_admin_school_id(user)

    pending_roles = (
        UserSchoolRole.query
        .filter_by(school_id=school_id, role="customer", approved=False)
        .all()
    )

    result = []
    for role in pending_roles:
        u = role.user
        result.append({
            "id": u.id,
            "name": u.name,
            "email": u.email,
            "registered_at": role.created_at.isoformat() if hasattr(role, "created_at") and role.created_at else None,
        })

    return jsonify(result)
```

**Step 3: Commit**
```bash
git add blueprints/api_admin/users.py tests/test_customer_approval.py
git commit -m "feat: GET /api/admin/pending-registrations Endpoint"
```

---

### Task A3: Pending-Registration Reject

**Problem:** iOS ruft `POST /api/admin/pending-registrations/{id}/reject` auf. Backend hat keinen Reject-Endpoint.

**iOS-Erwartung:**
- Request: `POST` mit optionalem Body `{"reason": "..."}`
- Response: `{"status": "ok"}`

**Files:**
- Modify: `blueprints/api_admin/users.py`
- Test: `tests/test_customer_approval.py`

**Step 1: Failing Test**

```python
def test_reject_pending_registration(client, approval_required_school, pending_customer):
    """POST /api/admin/pending-registrations/{id}/reject muss Kunden ablehnen."""
    school, admin = approval_required_school
    token = login_api_user(client, admin)
    resp = client.post(f"/api/admin/pending-registrations/{pending_customer.id}/reject",
                       headers={"Authorization": f"Bearer {token}"},
                       json={"reason": "Kein Platz"})
    assert resp.status_code == 200
    assert resp.get_json()["status"] == "ok"


def test_reject_removes_from_pending_list(client, approval_required_school, pending_customer):
    """Nach Reject darf der Kunde nicht mehr in der Pending-Liste sein."""
    school, admin = approval_required_school
    token = login_api_user(client, admin)
    client.post(f"/api/admin/pending-registrations/{pending_customer.id}/reject",
                headers={"Authorization": f"Bearer {token}"})
    resp = client.get("/api/admin/pending-registrations",
                      headers={"Authorization": f"Bearer {token}"})
    ids = [r["id"] for r in resp.get_json()]
    assert pending_customer.id not in ids


def test_reject_wrong_school_denied(client, approval_required_school, other_school_customer):
    """Admin darf Kunden anderer Schulen nicht ablehnen."""
    school, admin = approval_required_school
    token = login_api_user(client, admin)
    resp = client.post(f"/api/admin/pending-registrations/{other_school_customer.id}/reject",
                       headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 404
```

**Step 2: Implementierung**

```python
# In blueprints/api_admin/users.py

# Approve-Alias fuer iOS-Kompatibilitaet (bestehendes approve nutzen)
@api_admin_bp.route("/pending-registrations/<int:user_id>/approve", methods=["POST"])
@jwt_or_session_required
@limiter.limit("30 per hour")
def approve_pending_registration(user_id):
    """Alias fuer PUT /api/admin/users/{id}/approve — iOS-Kompatibilitaet."""
    return approve_user(user_id)


@api_admin_bp.route("/pending-registrations/<int:user_id>/reject", methods=["POST"])
@jwt_or_session_required
@limiter.limit("30 per hour")
def reject_pending_registration(user_id):
    """Lehnt eine ausstehende Kundenregistrierung ab."""
    user, err = _require_admin()
    if err:
        return err
    school_id = _get_admin_school_id(user)

    role = UserSchoolRole.query.filter_by(
        user_id=user_id, school_id=school_id, role="customer"
    ).first()
    if not role:
        return jsonify({"error": "Registrierung nicht gefunden"}), 404

    target = db.session.get(User, user_id)
    data = request.get_json(silent=True) or {}
    reason = data.get("reason")

    # Rolle entfernen (Ablehnung = kein Zugang)
    db.session.delete(role)

    # Optional: Ablehnungs-E-Mail senden
    if target and target.email:
        try:
            from email_service import send_event_email
            send_event_email(
                SchoolProfile.query.get(school_id),
                target,
                "customer_rejected",
                {"user_name": target.name, "reason": reason or ""},
            )
        except Exception:
            pass  # E-Mail-Versand darf nicht blockieren

    db.session.commit()
    return jsonify({"status": "ok", "message": "Registrierung abgelehnt"})
```

**Step 3: Commit**
```bash
git add blueprints/api_admin/users.py tests/test_customer_approval.py
git commit -m "feat: Pending-Registration Reject + Approve-Alias fuer iOS"
```

---

## BLOCK B — Fehlende REST APIs ✅ ERLEDIGT

> Detail-Spezifikationen: siehe `plans/2026-03-14-fehlende-rest-apis.md`

### Task B1: Setup Wizard API ✅
- `GET /api/admin/setup/status`
- `POST /api/admin/setup/complete`
- `POST /api/admin/setup/skip`
- **Files:** `blueprints/api_admin/setup.py`, `api_schemas.py`

### Task B2: Accounting Dashboard API ✅
- `GET /api/admin/accounting/jobs` (paginiert, filterbar)
- `POST /api/admin/accounting/jobs/{id}/retry`
- `POST /api/admin/accounting/retry-all`
- `POST /api/admin/accounting/sync-now`
- `POST /api/admin/accounting/test-connection`
- **Files:** `blueprints/api_admin/accounting.py`

### Task B3: Trainer Availability Admin API ✅
- `GET /api/admin/availability` (filterbar nach trainer_id, Datum)
- `POST /api/admin/availability`
- `PUT /api/admin/availability/{id}`
- `DELETE /api/admin/availability/{id}`
- **Files:** `blueprints/api_admin/availability.py`, `api_schemas.py`

### Task B4: Manual Customer Booking API ✅
- `POST /api/admin/bookings/direct` (Session + Booking)
- `GET /api/admin/customers/{id}/dogs`
- **Files:** `blueprints/api_admin/bookings.py`, `api_schemas.py`

### Task B5: Stripe Connect Onboarding API ✅
- `GET /api/admin/stripe-connect/status`
- `POST /api/admin/stripe-connect/onboard` (URL statt Redirect)
- `POST /api/admin/stripe-connect/refresh-status`
- `GET /api/admin/stripe-connect/dashboard-url`
- **Files:** `blueprints/api_admin/stripe_connect.py`

### Task B6: Price Management Platform Admin API ✅
- `GET /api/platform-admin/prices`
- `PUT /api/platform-admin/prices/plans/{plan_id}`
- `PUT /api/platform-admin/prices/addons/{price_id}`
- **Files:** `blueprints/api_platform_admin/prices.py`, `api_schemas.py`

---

## BLOCK C — Fehlende Web-UIs

> Detail-Spezifikationen: siehe `plans/2026-03-14-fehlende-web-uis.md`

### Task C1: Poll-UI im Chat
- Poll-Rendering in `appendMessageBubble()` fuer `message_type === 'poll'`
- Poll-Erstellungs-Modal mit dynamischen Fragen/Optionen
- Abstimmungs-Submit und Ergebnis-Anzeige
- **Files:** `templates/messages/_chat_js.html`, `templates/messages/_chat_panel.html`

### Task C2: Live-Location-Karte im Chat
- Leaflet.js-Karten-Panel (ausklappbar) fuer aktive Location-Sessions
- Socket.IO-Listener fuer `live_location_started`, `location_update`, `live_location_ended`
- `navigator.geolocation.watchPosition()` fuer Position-Updates
- Start/Stop-Buttons fuer Trainer/Admin
- **Files:** `templates/messages/_chat_js.html`, `templates/messages/_chat_panel.html`

### Task C3: Payment-Methods-Verwaltungsseite
- Neue Admin-Seite mit Kartenliste, Hinzufuegen (Stripe Elements), Entfernen, Standard-setzen
- Nutzt bestehende API `/api/admin/payment-methods/*`
- **Files:** `templates/admin/payment_methods.html`, `blueprints/saas.py` (Route), `templates/admin/admin_base.html` (Nav-Link)

### Task C4: Kontaktvorschlaege im Compose-Modal
- `/api/messages/contacts` vorladen, client-seitig filtern
- Rollen-Badges und Schulnamen in Ergebnisliste
- **Files:** `templates/messages/_chat_js.html`, `templates/messages/_modal_new_chat.html`

---

## BLOCK D — iOS-Stubs vervollstaendigen

### Task D1: Stripe PaymentSheet Integration (iOS)

**Problem:** `PurchaseView.swift:125` hat TODO: "Stripe PaymentSheet hier einbinden". Backend liefert `client_secret` via `POST /api/customer/checkout`, aber iOS zeigt kein Stripe Payment UI.

**Files:**
- Modify: `PawCoach/Features/Customer/PurchaseView.swift`
- Add: Stripe iOS SDK Dependency (SPM)

**Implementierung:**
```swift
// In purchase() Methode:
if let checkout = await viewModel.createCheckout(...) {
    // Stripe PaymentSheet konfigurieren
    var config = PaymentSheet.Configuration()
    config.merchantDisplayName = "PawCoach"
    let paymentSheet = PaymentSheet(
        paymentIntentClientSecret: checkout.clientSecret,
        configuration: config
    )
    // PaymentSheet praesentieren
    paymentSheet.present(from: rootViewController) { result in
        switch result {
        case .completed:
            showSuccess = true
        case .canceled:
            break
        case .failed(let error):
            errorMessage = error.localizedDescription
        }
    }
}
```

**Voraussetzungen:**
- Stripe iOS SDK via SPM hinzufuegen
- `STRIPE_PUBLISHABLE_KEY` in iOS-App konfigurieren
- Backend muss `publishable_key` in Checkout-Response liefern

### Task D2: Passkey-Verwaltung UI (iOS)

**Problem:** `TwoFactorManagementView.swift:41` hat TODO: "PasskeyManager integrieren". Backend-Endpoints existieren bereits (`/api/auth/passkey/register/*`).

**Files:**
- Modify: `PawCoach/Features/Settings/TwoFactorManagementView.swift`
- Add: `PawCoach/Features/Settings/PasskeyManagementView.swift`

**Implementierung:**
- Liste vorhandener Passkeys anzeigen (`GET /api/auth/passkey/list` — ggf. Backend ergaenzen)
- Neuen Passkey registrieren via `ASAuthorizationPlatformPublicKeyCredentialProvider`
- Passkey loeschen (`DELETE /api/auth/passkey/{id}` — ggf. Backend ergaenzen)
- Nutzt Apple AuthenticationServices Framework

**Backend-Ergaenzungen noetig:**
- `GET /api/auth/passkey/list` — Liste aller registrierten Passkeys des Users
- `DELETE /api/auth/passkey/{id}` — Passkey entfernen

---

## Implementierungs-Reihenfolge (empfohlen)

### Phase 1 — iOS-Blocker (1-2 Tage)
1. **A1**: Pfad-Alias `/api/schools/join/{code}` ← 30 Min
2. **A2**: Pending-Registrations Liste ← 1-2 Std
3. **A3**: Reject-Endpoint + Approve-Alias ← 1-2 Std

### Phase 2 — REST APIs (3-5 Tage)
4. **B1**: Setup Wizard API ← 1-2 Std
5. **B2**: Accounting Dashboard API ← 2-3 Std
6. **B3**: Trainer Availability Admin API ← 2-3 Std
7. **B4**: Manual Customer Booking API ← 3-4 Std
8. **B5**: Stripe Connect Onboarding API ← 2-3 Std
9. **B6**: Price Management API ← 2-3 Std

### Phase 3 — Web-UIs (3-5 Tage)
10. **C3**: Payment-Methods-Seite ← 2-3 Std
11. **C4**: Kontaktvorschlaege ← 1-2 Std
12. **C1**: Poll-UI im Chat ← 4-5 Std
13. **C2**: Live-Location-Karte ← 4-5 Std

### Phase 4 — iOS-Stubs (2-3 Tage)
14. **D1**: Stripe PaymentSheet ← 3-4 Std
15. **D2**: Passkey-Verwaltung ← 3-4 Std

---

## Sicherheitshinweise (gueltig fuer alle Tasks)

- Alle API-Endpoints: `@jwt_or_session_required` + `@limiter.limit()`
- Tenant-Isolation: Jeder Endpoint prueft `school_id`
- Eingabevalidierung: Marshmallow-Schemas serverseitig
- Keine PII in Logs oder Error-Responses
- Web-UIs: Event-Delegation (`data-action`), CSP-Nonces, CSRF-Token
- iOS: Keine Secrets im App-Bundle, HTTPS-Pinning beibehalten

## Detail-Spezifikationen

- Block B Details: `plans/2026-03-14-fehlende-rest-apis.md`
- Block C Details: `plans/2026-03-14-fehlende-web-uis.md`
