# Fehlende Web-UIs Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Web-Frontends fuer 4 API-Features nachrüsten, die bisher nur via REST-API nutzbar sind.

**Architektur:** Alle Features erweitern bestehende Templates und nutzen Event-Delegation (`data-action`), CSP-Nonces, und `fetch()` gegen die existierenden API-Endpoints. Kein neues Backend noetig (APIs existieren bereits).

**Tech Stack:** Jinja2, TailwindCSS, Vanilla JS (Event-Delegation), Socket.IO, Leaflet.js, Stripe.js

**Hinweis:** Message Reactions und Forum Thread Moderation (Pin/Lock) sind bereits im Web-UI implementiert und werden hier NICHT behandelt.

---

## Task 1: Poll-UI im Chat

**Kontext:** `appendMessageBubble()` in `_chat_js.html` behandelt `message_type === 'poll'` nicht. Die API-Serialisierung liefert bereits Poll-Daten mit Fragen, Optionen, etc.

**API-Endpoints (existierend):**
- `POST /api/messages/<conv_id>/poll` — Poll erstellen
- `POST /api/messages/polls/<poll_id>/respond` — Abstimmen
- `GET /api/messages/polls/<poll_id>/results` — Ergebnisse abrufen

**Datenstruktur (aus `_serialize_message`):**
```json
{
  "poll": {
    "id": 1, "title": "...", "is_anonymous": false,
    "is_closed": false, "is_expired": false, "expires_at": null,
    "questions": [{
      "id": 1, "question_text": "...", "question_type": "single_choice",
      "sort_order": 0, "scale_min": null, "scale_max": null,
      "options": [{"id": 1, "option_text": "...", "sort_order": 0}]
    }]
  }
}
```

**Files:**
- Modify: `templates/messages/_chat_js.html` — Poll-Rendering in `appendMessageBubble()`, Response-Handling
- Modify: `templates/messages/_chat_panel.html` — Poll-Erstellungs-Modal, Poll-Button im Input-Bereich
- Test: `tests/test_poll_ui.py`

### Step 1: Failing Test schreiben

```python
# tests/test_poll_ui.py
"""Tests fuer Poll-UI-Rendering und -Interaktion."""
import pytest
from tests.conftest import login_user


def test_poll_message_rendered_in_conversation(client, sample_school_with_messaging):
    """Poll-Nachrichten muessen als Poll-Card gerendert werden, nicht als Text."""
    school, admin, customer = sample_school_with_messaging
    login_user(client, admin)
    # Erstelle Conversation + Poll via API
    conv_resp = client.post("/api/messages/conversations", json={
        "participant_id": customer.id, "message": "test"
    })
    conv_id = conv_resp.get_json()["conversation"]["id"]
    poll_resp = client.post(f"/api/messages/{conv_id}/poll", json={
        "title": "Welcher Termin passt?",
        "questions": [{"question_text": "Tag?", "question_type": "single_choice",
                        "options": [{"option_text": "Montag"}, {"option_text": "Dienstag"}]}]
    })
    assert poll_resp.status_code == 201
    # Lade Conversation-Seite
    page = client.get(f"/messages/{conv_id}")
    assert page.status_code == 200
    html = page.data.decode()
    # Poll-Rendering: appendMessageBubble muss poll-Typ verarbeiten
    assert "renderPollCard" in html or "poll-card" in html


def test_poll_creation_modal_exists(client, sample_school_with_messaging):
    """Poll-Erstellungs-Modal muss im Chat-Panel vorhanden sein."""
    school, admin, _ = sample_school_with_messaging
    login_user(client, admin)
    conv_resp = client.post("/api/messages/conversations", json={
        "participant_id": admin.id, "message": "test"
    })
    conv_id = conv_resp.get_json()["conversation"]["id"]
    page = client.get(f"/messages/{conv_id}")
    html = page.data.decode()
    assert "poll-create-modal" in html or "createPollModal" in html


def test_poll_respond_via_api(client, sample_school_with_messaging):
    """Poll-Abstimmung muss ueber die API funktionieren."""
    school, admin, customer = sample_school_with_messaging
    login_user(client, admin)
    conv_resp = client.post("/api/messages/conversations", json={
        "participant_id": customer.id, "message": "test"
    })
    conv_id = conv_resp.get_json()["conversation"]["id"]
    poll_resp = client.post(f"/api/messages/{conv_id}/poll", json={
        "title": "Test-Poll",
        "questions": [{"question_text": "Frage?", "question_type": "single_choice",
                        "options": [{"option_text": "Ja"}, {"option_text": "Nein"}]}]
    })
    poll_id = poll_resp.get_json()["poll"]["id"]
    question_id = poll_resp.get_json()["poll"]["questions"][0]["id"]
    option_id = poll_resp.get_json()["poll"]["questions"][0]["options"][0]["id"]
    # Als Customer abstimmen
    login_user(client, customer)
    resp = client.post(f"/api/messages/polls/{poll_id}/respond", json={
        "responses": [{"question_id": question_id, "selected_option_ids": [option_id]}]
    })
    assert resp.status_code in (200, 201)
```

### Step 2: Test ausfuehren, Fail verifizieren

```bash
pytest tests/test_poll_ui.py -v
```
Erwartet: FAIL (kein `renderPollCard` / `poll-card` im HTML)

### Step 3: Poll-Rendering in `_chat_js.html` implementieren

In `appendMessageBubble()` nach dem bestehenden `location`-Branch einfuegen:

```javascript
// Poll-Rendering
if (msg.message_type === 'poll' && msg.poll) {
    const poll = msg.poll;
    const pollCard = document.createElement('div');
    pollCard.className = 'poll-card bg-white border border-gray-200 rounded-lg p-4 max-w-sm';
    pollCard.dataset.pollId = poll.id;

    let html = `<h4 class="font-semibold text-sm mb-3">${escapeHtml(poll.title)}</h4>`;

    poll.questions.forEach(q => {
        html += `<div class="mb-3" data-question-id="${q.id}">`;
        html += `<p class="text-sm text-gray-700 mb-2">${escapeHtml(q.question_text)}</p>`;

        if (q.question_type === 'single_choice' || q.question_type === 'multi_choice') {
            const inputType = q.question_type === 'single_choice' ? 'radio' : 'checkbox';
            q.options.forEach(opt => {
                html += `<label class="flex items-center gap-2 py-1 cursor-pointer">
                    <input type="${inputType}" name="q_${q.id}" value="${opt.id}"
                           class="poll-option" data-question-id="${q.id}" data-option-id="${opt.id}">
                    <span class="text-sm">${escapeHtml(opt.option_text)}</span>
                </label>`;
            });
        } else if (q.question_type === 'free_text') {
            html += `<textarea class="poll-free-text w-full border rounded p-2 text-sm"
                       data-question-id="${q.id}" rows="2" placeholder="Antwort..."></textarea>`;
        } else if (q.question_type === 'scale') {
            html += `<input type="range" min="${q.scale_min || 1}" max="${q.scale_max || 10}"
                       class="poll-scale w-full" data-question-id="${q.id}">`;
        }
        html += '</div>';
    });

    if (poll.is_closed || poll.is_expired) {
        html += `<p class="text-xs text-gray-500 mt-2">${poll.is_expired ? 'Abgelaufen' : 'Geschlossen'}</p>`;
        html += `<button class="mt-2 text-sm text-blue-600 hover:underline" data-action="show-poll-results"
                   data-poll-id="${poll.id}">Ergebnisse anzeigen</button>`;
    } else {
        html += `<button class="mt-3 w-full bg-blue-600 text-white text-sm py-2 rounded hover:bg-blue-700"
                   data-action="submit-poll-response" data-poll-id="${poll.id}">Abstimmen</button>`;
        html += `<button class="mt-1 w-full text-sm text-gray-500 hover:underline"
                   data-action="show-poll-results" data-poll-id="${poll.id}">Ergebnisse</button>`;
    }

    pollCard.innerHTML = html;
    bubble.querySelector('.msg-body') ? bubble.querySelector('.msg-body').appendChild(pollCard)
        : bubble.appendChild(pollCard);
}
```

Event-Delegation in `setupEventDelegation()`:

```javascript
// Poll abstimmen
document.addEventListener('click', e => {
    const btn = e.target.closest('[data-action="submit-poll-response"]');
    if (!btn) return;
    const pollId = btn.dataset.pollId;
    const card = btn.closest('.poll-card');
    const responses = [];
    card.querySelectorAll('[data-question-id]').forEach(qDiv => {
        if (qDiv.tagName === 'DIV') {
            const qId = parseInt(qDiv.dataset.questionId);
            const selectedOpts = [];
            qDiv.querySelectorAll('.poll-option:checked').forEach(inp => {
                selectedOpts.push(parseInt(inp.dataset.optionId));
            });
            const freeText = qDiv.querySelector('.poll-free-text');
            const scale = qDiv.querySelector('.poll-scale');
            responses.push({
                question_id: qId,
                selected_option_ids: selectedOpts,
                free_text: freeText ? freeText.value : null,
                scale_value: scale ? parseInt(scale.value) : null
            });
        }
    });
    fetch(`/api/messages/polls/${pollId}/respond`, {
        method: 'POST', credentials: 'same-origin',
        headers: {'Content-Type': 'application/json', 'X-CSRFToken': csrfToken},
        body: JSON.stringify({responses})
    }).then(r => { if (r.ok) btn.textContent = 'Abgestimmt!'; btn.disabled = true; });
});

// Poll-Ergebnisse anzeigen
document.addEventListener('click', e => {
    const btn = e.target.closest('[data-action="show-poll-results"]');
    if (!btn) return;
    const pollId = btn.dataset.pollId;
    fetch(`/api/messages/polls/${pollId}/results`, {credentials: 'same-origin'})
        .then(r => r.json())
        .then(data => { renderPollResults(btn.closest('.poll-card'), data); });
});
```

### Step 4: Poll-Erstellungs-Modal in `_chat_panel.html`

```html
<!-- Poll-Button im Input-Bereich (neben Datei-Upload und Location) -->
<button type="button" data-action="open-poll-modal" title="Umfrage erstellen"
        class="p-2 text-gray-500 hover:text-blue-600">
    <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
              d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2m-6 9l2 2 4-4"/>
    </svg>
</button>

<!-- Poll-Erstellungs-Modal -->
<div id="poll-create-modal" class="hidden fixed inset-0 z-50 bg-black/50 flex items-center justify-center">
    <div class="bg-white rounded-lg shadow-xl w-full max-w-md mx-4 p-6">
        <h3 class="text-lg font-semibold mb-4">Umfrage erstellen</h3>
        <input type="text" id="poll-title" placeholder="Umfrage-Titel"
               class="w-full border rounded px-3 py-2 mb-3 text-sm">
        <div id="poll-questions-container">
            <!-- Dynamisch befuellt -->
        </div>
        <button data-action="add-poll-question"
                class="text-sm text-blue-600 hover:underline mb-4">+ Frage hinzufuegen</button>
        <div class="flex justify-end gap-2 mt-4">
            <button data-action="close-poll-modal"
                    class="px-4 py-2 text-sm text-gray-600 hover:bg-gray-100 rounded">Abbrechen</button>
            <button data-action="submit-poll-create"
                    class="px-4 py-2 text-sm bg-blue-600 text-white rounded hover:bg-blue-700">Erstellen</button>
        </div>
    </div>
</div>
```

### Step 5: Tests ausfuehren, Pass verifizieren

```bash
pytest tests/test_poll_ui.py -v
```
Erwartet: PASS

### Step 6: Commit

```bash
git add templates/messages/_chat_js.html templates/messages/_chat_panel.html tests/test_poll_ui.py
git commit -m "feat: Poll-UI im Chat (Erstellung, Abstimmung, Ergebnisse)"
```

---

## Task 2: Live-Location-Karte im Chat

**Kontext:** Socket.IO-Events `live_location_started`, `live_location_ended`, `location_update` werden vom Backend gesendet. Leaflet.js ist bereits geladen. Es fehlt die Kartenvisualisierung.

**API-Endpoints (existierend):**
- `POST /api/messages/<conv_id>/live-location` — Session starten (Trainer/Admin)
- `DELETE /api/messages/live-location/<session_id>` — Session beenden
- `GET /api/messages/live-location/<session_id>/positions` — Aktuelle Positionen
- `POST /api/messages/<conv_id>/toggle-location-sharing` — Standort teilen (Kunde)
- Socket-Events: `live_location_started`, `live_location_ended`, `location_update`, `share_position`

**Files:**
- Modify: `templates/messages/_chat_js.html` — Socket-Event-Handler, Karten-Rendering, Geolocation-Watcher
- Modify: `templates/messages/_chat_panel.html` — Karten-Panel (ausklappbar), Start/Stop-Buttons
- Test: `tests/test_live_location_ui.py`

### Step 1: Failing Test schreiben

```python
# tests/test_live_location_ui.py
"""Tests fuer Live-Location-Map-UI."""
import pytest
from tests.conftest import login_user


def test_live_location_map_panel_exists(client, sample_school_with_messaging):
    """Karten-Panel-Container muss im Chat vorhanden sein."""
    school, admin, customer = sample_school_with_messaging
    login_user(client, admin)
    conv_resp = client.post("/api/messages/conversations", json={
        "participant_id": customer.id, "message": "test"
    })
    conv_id = conv_resp.get_json()["conversation"]["id"]
    page = client.get(f"/messages/{conv_id}")
    html = page.data.decode()
    assert "live-location-map" in html


def test_live_location_start_button_for_admin(client, sample_school_with_messaging):
    """Admin/Trainer soll einen Start-Button fuer Live-Location sehen."""
    school, admin, _ = sample_school_with_messaging
    login_user(client, admin)
    conv_resp = client.post("/api/messages/conversations", json={
        "participant_id": admin.id, "message": "test"
    })
    conv_id = conv_resp.get_json()["conversation"]["id"]
    page = client.get(f"/messages/{conv_id}")
    html = page.data.decode()
    assert "start-live-location" in html or "toggle-location-sharing" in html
```

### Step 2: Test ausfuehren, Fail verifizieren

```bash
pytest tests/test_live_location_ui.py -v
```

### Step 3: Karten-Panel in `_chat_panel.html` implementieren

```html
<!-- Live-Location-Map-Panel (ausklappbar, oben im Chat) -->
<div id="live-location-map" class="hidden border-b border-gray-200 bg-gray-50">
    <div class="flex items-center justify-between px-4 py-2">
        <span class="text-sm font-medium text-gray-700">Live-Standort</span>
        <button data-action="close-live-location-panel" class="text-gray-400 hover:text-gray-600">
            <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
            </svg>
        </button>
    </div>
    <div id="live-location-leaflet" class="h-48 w-full"></div>
</div>

<!-- Start-Button (neben anderen Chat-Actions) -->
<button type="button" data-action="start-live-location" title="Live-Standort starten"
        class="p-2 text-gray-500 hover:text-green-600">
    <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
              d="M17.657 16.657L13.414 20.9a1.998 1.998 0 01-2.827 0l-4.244-4.243a8 8 0 1111.314 0z"/>
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 11a3 3 0 11-6 0 3 3 0 016 0z"/>
    </svg>
</button>
```

### Step 4: Socket-Handler und Karten-JS in `_chat_js.html`

```javascript
// Live-Location-Management
let liveLocationMap = null;
let liveLocationMarkers = {};

function initLiveLocationMap() {
    const container = document.getElementById('live-location-leaflet');
    if (!container || liveLocationMap) return;
    liveLocationMap = L.map(container).setView([51.1657, 10.4515], 13);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; OpenStreetMap'
    }).addTo(liveLocationMap);
}

function showLiveLocationPanel() {
    const panel = document.getElementById('live-location-map');
    panel.classList.remove('hidden');
    initLiveLocationMap();
    setTimeout(() => liveLocationMap.invalidateSize(), 100);
}

function updateMarker(userId, lat, lng, label) {
    if (!liveLocationMap) return;
    if (liveLocationMarkers[userId]) {
        liveLocationMarkers[userId].setLatLng([lat, lng]);
    } else {
        liveLocationMarkers[userId] = L.marker([lat, lng])
            .bindPopup(label || 'Teilnehmer')
            .addTo(liveLocationMap);
    }
    // Karte auf alle Marker zentrieren
    const bounds = L.latLngBounds(Object.values(liveLocationMarkers).map(m => m.getLatLng()));
    liveLocationMap.fitBounds(bounds, {padding: [30, 30]});
}

// Socket.IO-Events
socket.on('live_location_started', data => {
    showLiveLocationPanel();
});

socket.on('location_update', data => {
    updateMarker(data.user_id, data.lat, data.lng, data.display_name);
});

socket.on('live_location_ended', data => {
    document.getElementById('live-location-map').classList.add('hidden');
    liveLocationMarkers = {};
    if (liveLocationMap) { liveLocationMap.remove(); liveLocationMap = null; }
});

// Event-Delegation: Start Live-Location
document.addEventListener('click', e => {
    const btn = e.target.closest('[data-action="start-live-location"]');
    if (!btn) return;
    fetch(`/api/messages/${currentConvId}/live-location`, {
        method: 'POST', credentials: 'same-origin',
        headers: {'Content-Type': 'application/json', 'X-CSRFToken': csrfToken}
    }).then(r => { if (r.ok) showLiveLocationPanel(); });
});

// Geolocation-Watcher fuer Position-Updates
let geoWatchId = null;
function startSharingPosition() {
    if (!navigator.geolocation) return;
    geoWatchId = navigator.geolocation.watchPosition(pos => {
        socket.emit('share_position', {
            conversation_id: currentConvId,
            lat: pos.coords.latitude,
            lng: pos.coords.longitude
        });
    }, null, {enableHighAccuracy: true, maximumAge: 5000});
}
```

### Step 5: Tests ausfuehren, Pass verifizieren

```bash
pytest tests/test_live_location_ui.py -v
```

### Step 6: Commit

```bash
git add templates/messages/_chat_js.html templates/messages/_chat_panel.html tests/test_live_location_ui.py
git commit -m "feat: Live-Location-Karte im Chat mit Leaflet.js"
```

---

## Task 3: Payment-Methods-Verwaltungsseite

**Kontext:** Volle API unter `/api/admin/payment-methods/*` existiert, aber keine Web-Seite.

**API-Endpoints (existierend):**
- `GET /api/admin/payment-methods` — Liste aller Zahlungsmethoden
- `POST /api/admin/payment-methods` — SetupIntent erstellen (`client_secret`)
- `DELETE /api/admin/payment-methods/<id>` — Karte entfernen
- `PUT /api/admin/payment-methods/<id>/default` — Als Standard setzen

**Files:**
- Create: `templates/admin/payment_methods.html` — Neue Admin-Seite
- Modify: `blueprints/admin/settings.py` oder `blueprints/saas.py` — Web-Route
- Modify: `templates/admin/admin_base.html` — Navigations-Link
- Test: `tests/test_admin_payment_methods_page.py`

### Step 1: Failing Test schreiben

```python
# tests/test_admin_payment_methods_page.py
"""Tests fuer Payment-Methods-Verwaltungsseite."""
from tests.conftest import login_user


def test_payment_methods_page_loads(client, sample_school):
    """Seite muss fuer Admins erreichbar sein."""
    school, admin = sample_school
    login_user(client, admin)
    resp = client.get("/admin/payment-methods")
    assert resp.status_code == 200
    assert "Zahlungsmethoden" in resp.data.decode()


def test_payment_methods_page_403_for_customer(client, sample_school_with_customer):
    """Kunden duerfen die Seite nicht sehen."""
    school, admin, customer = sample_school_with_customer
    login_user(client, customer)
    resp = client.get("/admin/payment-methods")
    assert resp.status_code in (302, 403)


def test_payment_methods_page_has_add_button(client, sample_school):
    """Seite muss einen 'Karte hinzufuegen'-Button haben."""
    school, admin = sample_school
    login_user(client, admin)
    resp = client.get("/admin/payment-methods")
    html = resp.data.decode()
    assert "add-payment-method" in html
```

### Step 2: Test ausfuehren, Fail verifizieren

```bash
pytest tests/test_admin_payment_methods_page.py -v
```

### Step 3: Web-Route hinzufuegen

In `blueprints/saas.py` oder `blueprints/admin/settings.py`:

```python
@bp.route("/admin/payment-methods")
@login_required
def payment_methods():
    """Zahlungsmethoden-Verwaltungsseite."""
    from context_helpers import get_current_role
    if get_current_role() != "admin":
        abort(403)
    return render_template("admin/payment_methods.html")
```

### Step 4: Template erstellen

```html
{% extends "admin/admin_base.html" %}
{% set active_page = "payment_methods" %}

{% block title %}Zahlungsmethoden{% endblock %}

{% block content %}
<div class="max-w-2xl mx-auto">
    <div class="flex items-center justify-between mb-6">
        <h1 class="text-2xl font-bold">Zahlungsmethoden</h1>
        <button data-action="add-payment-method"
                class="bg-blue-600 text-white px-4 py-2 rounded-lg text-sm hover:bg-blue-700">
            + Karte hinzufuegen
        </button>
    </div>

    <div id="payment-methods-list" class="space-y-3">
        <p class="text-gray-500 text-sm">Wird geladen...</p>
    </div>

    <!-- Stripe Elements Container (hidden bis Add geklickt) -->
    <div id="stripe-card-form" class="hidden mt-6 p-4 border rounded-lg">
        <div id="card-element" class="mb-4"></div>
        <div id="card-errors" class="text-red-500 text-sm mb-2"></div>
        <div class="flex gap-2">
            <button data-action="confirm-add-card"
                    class="bg-green-600 text-white px-4 py-2 rounded text-sm">Speichern</button>
            <button data-action="cancel-add-card"
                    class="text-gray-500 px-4 py-2 rounded text-sm">Abbrechen</button>
        </div>
    </div>
</div>

<script nonce="{{ csp_nonce() }}">
(function() {
    const csrfToken = document.querySelector('meta[name=csrf-token]')?.content;
    let stripe, cardElement;

    function loadMethods() {
        fetch('/api/admin/payment-methods', {credentials: 'same-origin'})
            .then(r => r.json())
            .then(data => renderMethods(data));
    }

    function renderMethods(data) {
        const list = document.getElementById('payment-methods-list');
        if (!data.payment_methods?.length) {
            list.innerHTML = '<p class="text-gray-500">Keine Zahlungsmethoden hinterlegt.</p>';
            return;
        }
        list.innerHTML = data.payment_methods.map(pm => `
            <div class="flex items-center justify-between p-4 bg-white border rounded-lg">
                <div class="flex items-center gap-3">
                    <span class="font-mono text-sm">${pm.brand} •••• ${pm.last4}</span>
                    <span class="text-xs text-gray-500">${pm.exp_month}/${pm.exp_year}</span>
                    ${pm.is_default ? '<span class="bg-green-100 text-green-700 text-xs px-2 py-0.5 rounded">Standard</span>' : ''}
                </div>
                <div class="flex gap-2">
                    ${!pm.is_default ? `<button data-action="set-default-card" data-pm-id="${pm.id}"
                        class="text-sm text-blue-600 hover:underline">Als Standard</button>` : ''}
                    <button data-action="delete-card" data-pm-id="${pm.id}"
                        class="text-sm text-red-500 hover:underline">Entfernen</button>
                </div>
            </div>
        `).join('');
    }

    document.addEventListener('click', e => {
        const addBtn = e.target.closest('[data-action="add-payment-method"]');
        if (addBtn) {
            fetch('/api/admin/payment-methods', {
                method: 'POST', credentials: 'same-origin',
                headers: {'Content-Type': 'application/json', 'X-CSRFToken': csrfToken}
            }).then(r => r.json()).then(data => {
                if (!stripe) {
                    stripe = Stripe(data.publishable_key || '{{ stripe_pk }}');
                    const elements = stripe.elements();
                    cardElement = elements.create('card');
                    cardElement.mount('#card-element');
                }
                document.getElementById('stripe-card-form').classList.remove('hidden');
                window._clientSecret = data.client_secret;
            });
        }

        const confirmBtn = e.target.closest('[data-action="confirm-add-card"]');
        if (confirmBtn && stripe && window._clientSecret) {
            stripe.confirmCardSetup(window._clientSecret, {
                payment_method: {card: cardElement}
            }).then(result => {
                if (result.error) {
                    document.getElementById('card-errors').textContent = result.error.message;
                } else {
                    document.getElementById('stripe-card-form').classList.add('hidden');
                    loadMethods();
                }
            });
        }

        const cancelBtn = e.target.closest('[data-action="cancel-add-card"]');
        if (cancelBtn) document.getElementById('stripe-card-form').classList.add('hidden');

        const defaultBtn = e.target.closest('[data-action="set-default-card"]');
        if (defaultBtn) {
            fetch(`/api/admin/payment-methods/${defaultBtn.dataset.pmId}/default`, {
                method: 'PUT', credentials: 'same-origin',
                headers: {'X-CSRFToken': csrfToken}
            }).then(() => loadMethods());
        }

        const deleteBtn = e.target.closest('[data-action="delete-card"]');
        if (deleteBtn && confirm('Zahlungsmethode wirklich entfernen?')) {
            fetch(`/api/admin/payment-methods/${deleteBtn.dataset.pmId}`, {
                method: 'DELETE', credentials: 'same-origin',
                headers: {'X-CSRFToken': csrfToken}
            }).then(() => loadMethods());
        }
    });

    loadMethods();
})();
</script>
{% endblock %}
```

### Step 5: Navigation-Link in `admin_base.html`

Im Einstellungen-Bereich der Sidebar:

```html
<a href="{{ url_for('saas.payment_methods') }}"
   class="...">Zahlungsmethoden</a>
```

### Step 6: Tests ausfuehren, Pass verifizieren

```bash
pytest tests/test_admin_payment_methods_page.py -v
```

### Step 7: Commit

```bash
git add templates/admin/payment_methods.html blueprints/saas.py templates/admin/admin_base.html tests/test_admin_payment_methods_page.py
git commit -m "feat: Payment-Methods-Verwaltungsseite fuer Admins"
```

---

## Task 4: Kontaktvorschlaege (Typeahead) im Compose-Modal

**Kontext:** `/api/messages/contacts` liefert gruppierte, angereicherte Kontaktdaten (Rollen, Schulnamen, Gruppen). Das Compose-Modal nutzt stattdessen nur `/messages/api/users?q=` mit einfacher Textsuche.

**API-Endpoint (existierend):**
- `GET /api/messages/contacts` — Gruppierte Kontaktliste mit Rollen, Schulnamen

**Files:**
- Modify: `templates/messages/_chat_js.html` — Kontakte vorladen, client-seitig filtern
- Modify: `templates/messages/_modal_new_chat.html` — Ergebnisanzeige mit Rollen-Badges, Gruppen
- Test: `tests/test_contacts_typeahead.py`

### Step 1: Failing Test schreiben

```python
# tests/test_contacts_typeahead.py
"""Tests fuer Kontaktvorschlaege im Compose-Modal."""
from tests.conftest import login_user


def test_contacts_endpoint_returns_grouped_data(client, sample_school_with_messaging):
    """Contacts-Endpoint muss gruppierte Daten liefern."""
    school, admin, customer = sample_school_with_messaging
    login_user(client, admin)
    resp = client.get("/api/messages/contacts")
    assert resp.status_code == 200
    data = resp.get_json()
    assert "contacts" in data or "groups" in data


def test_compose_modal_references_contacts(client, sample_school_with_messaging):
    """Compose-Modal muss Contacts-API referenzieren."""
    school, admin, _ = sample_school_with_messaging
    login_user(client, admin)
    resp = client.get("/messages/")
    html = resp.data.decode()
    assert "/api/messages/contacts" in html or "loadContacts" in html
```

### Step 2: Test ausfuehren, Fail verifizieren

### Step 3: Contacts-Prefetch und Client-Side-Filter implementieren

In `_chat_js.html`:

```javascript
// Kontakte vorladen
let contactsCache = null;

function loadContacts() {
    return fetch('/api/messages/contacts', {credentials: 'same-origin'})
        .then(r => r.json())
        .then(data => { contactsCache = data; return data; });
}

function filterContacts(query) {
    if (!contactsCache) return [];
    const q = query.toLowerCase();
    const contacts = contactsCache.contacts || contactsCache;
    return (Array.isArray(contacts) ? contacts : []).filter(c =>
        (c.name || '').toLowerCase().includes(q) ||
        (c.email || '').toLowerCase().includes(q)
    );
}
```

In `_modal_new_chat.html` — Ergebnisse mit Rollen-Badges:

```javascript
// Bestehende searchUsers() erweitern/ersetzen
function searchUsers(query) {
    if (contactsCache) {
        const results = filterContacts(query);
        renderContactResults(results);
    } else {
        // Fallback auf alten Endpoint
        fetch(`/messages/api/users?q=${encodeURIComponent(query)}`, {credentials: 'same-origin'})
            .then(r => r.json()).then(renderContactResults);
    }
}

function renderContactResults(contacts) {
    const container = document.getElementById('user-search-results');
    container.innerHTML = contacts.map(c => `
        <button data-action="select-contact" data-user-id="${c.id}"
                class="w-full flex items-center gap-3 p-2 hover:bg-gray-50 rounded text-left">
            <span class="font-medium text-sm">${escapeHtml(c.name)}</span>
            ${c.role ? `<span class="text-xs bg-gray-100 text-gray-600 px-2 py-0.5 rounded">${c.role}</span>` : ''}
            ${c.school_name ? `<span class="text-xs text-gray-400">${escapeHtml(c.school_name)}</span>` : ''}
        </button>
    `).join('') || '<p class="text-sm text-gray-500 p-2">Keine Ergebnisse</p>';
}
```

### Step 4: Tests ausfuehren, Pass verifizieren

### Step 5: Commit

```bash
git add templates/messages/_chat_js.html templates/messages/_modal_new_chat.html tests/test_contacts_typeahead.py
git commit -m "feat: Kontaktvorschlaege mit Rollen-Badges im Compose-Modal"
```

---

## Implementierungs-Reihenfolge

1. **Task 3: Payment Methods** — Am isoliertesten, keine Abhaengigkeiten
2. **Task 4: Kontaktvorschlaege** — Kleiner Scope, verbessert Compose-Modal
3. **Task 1: Polls UI** — Mittlerer Scope, erweitert bestehenden Chat-JS
4. **Task 2: Live Location Map** — Am komplexesten, benoetigt Socket-Handler + Leaflet

## Sicherheitshinweise

- Alle neuen UIs MUESSEN `data-action` Event-Delegation verwenden (kein `onclick`)
- Alle `<script>`-Tags MUESSEN `nonce="{{ csp_nonce() }}"` verwenden
- Alle `fetch()`-Aufrufe MUESSEN `credentials: 'same-origin'` und CSRF-Token-Header enthalten
- Payment-Methods-Seite muss `js.stripe.com` in CSP erlauben
- Kontaktvorschlaege duerfen keine Cross-Tenant-Daten leaken (API erzwingt das)
- Poll-Responses muessen Option-IDs validieren (API erzwingt das)
