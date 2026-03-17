# Unified Modernization Implementation Plan

**Datum:** 2026-03-17
**Ziel:** Backend, Web und iOS ohne Workarounds auf einen gemeinsamen, modernen
Produkt- und Architekturstand bringen.

## 1. Zielbild

Nach dem Login sollen Web und iOS dieselben fachlichen Funktionen anbieten, soweit sie
im jeweiligen Client sinnvoll darstellbar sind. Sichtbarkeit und Zugriff richten sich
nicht nur nach Rollen, sondern nach dem wirksamen Produktkontext:

- aktive Rolle
- aktive Schule
- gebuchter SaaS-Plan
- aktive Addons
- serverseitig berechnete Feature-Freigaben

Die Architektur folgt dabei diesen Regeln:

- Backend ist System of Record.
- Web und iOS sind gleichwertige Clients gegen denselben Contract.
- REST ist der Standard fuer Laden und Mutationen.
- Socket.IO ist der Standard fuer in-app Realtime.
- Push ist nur Offline-Benachrichtigung und Deep Link.
- Keine dauerhaften Workarounds wie Polling, Role-only-Gating oder Client-Sonderpfade.

## 2. Zielarchitektur

### 2.1 Contracts

Es gibt drei verbindliche Contracts:

1. `openapi.json` fuer REST-Pfade und Auth-Anforderungen
2. Capability-Contract fuer Navigation, Sichtbarkeit und Berechtigungen
3. Socket-Event-Contract fuer Realtime-Zustaende

### 2.2 Capability-Contract

Ein neuer kanonischer Endpoint wird eingefuehrt:

- `GET /api/features`
oder
- `GET /api/auth/capabilities`

Empfohlenes Response-Schema:

```json
{
  "context": {
    "user_id": 8,
    "school_id": 12,
    "role": "admin"
  },
  "subscription": {
    "subscribed_plan": "professional",
    "effective_plan": "professional",
    "status": "active"
  },
  "addons": {
    "messaging": { "active": true, "included": false, "source": "addon" },
    "community_forum": { "active": false, "included": false, "source": null },
    "voucher_system": { "active": true, "included": false, "source": "addon" }
  },
  "features": {
    "dashboard": true,
    "messaging": true,
    "forum": false,
    "trainer_calendar": true,
    "einzelstunden": true,
    "opengroup": true,
    "voucher_system": true,
    "pending_registrations": true,
    "admin_accounting": true
  },
  "limits": {
    "max_trainers": 5,
    "max_courses": null
  },
  "contract_version": 1
}
```

Regeln:

- `features` entscheidet die UI-Sichtbarkeit.
- `addons` erklaert, warum ein Feature aktiv oder inaktiv ist.
- `limits` steuert quantitative Beschraenkungen.
- `contract_version` erlaubt saubere Weiterentwicklung.

### 2.3 Realtime-Contract fuer Messaging

REST bleibt die autoritative Schreibe-Schnittstelle. Socket.IO liefert alle Live-Updates.

#### Event `new_message`

```json
{
  "id": 123,
  "client_message_id": "d9b437d1-4a4a-4f7b-b4dd-8db5e6f0f4fd",
  "conversation_id": 45,
  "sender_id": 8,
  "sender_name": "Anna",
  "body": "Hi",
  "message_type": "text",
  "created_at": "2026-03-17T12:00:00+00:00",
  "reply_to_id": null,
  "delivered_at": null,
  "read_at": null,
  "delivery_status": "sent"
}
```

#### Event `message_receipts_updated`

```json
{
  "conversation_id": 45,
  "updates": [
    {
      "message_id": 123,
      "delivered_at": "2026-03-17T12:00:04+00:00",
      "read_at": null,
      "delivery_status": "delivered"
    }
  ]
}
```

#### Event `presence_update`

```json
{
  "user_id": 8,
  "is_online": true,
  "last_seen": "2026-03-17T12:00:00+00:00",
  "updated_at": "2026-03-17T12:00:00+00:00"
}
```

## 3. Workstream A — Shared Docs and Contract Hygiene

### Ziel

Eine saubere gemeinsame Planungs- und Contract-Basis schaffen.

### Aufgaben

1. Alle cross-platform Plaene in `PawCoach-Shared-Docs/plans/` zusammenfuehren.
2. Duplikate aus den Produkt-Repositories entfernen.
3. `CROSS-PLATFORM-FEATURE-SPEC.md` auf Capability-Logik aktualisieren.
4. `CROSS-PLATFORM-SYNC-STRATEGY.md` auf Shared-Repo-Ownership aktualisieren.
5. `openapi.json` als kanonisches Artefakt im Shared-Repo pflegen.
6. Dokumentationsdrift regelmaessig ueber Delta-Reviews pruefen.

### Definition of Done

- Keine doppelten cross-platform Planungsdateien mehr in Produkt-Repositories
- Alle Teams arbeiten auf dieselben Dokumente
- OpenAPI-Pfad im Backend zeigt auf `docs/shared/openapi.json`

## 4. Workstream B — Backend Foundation and Entitlements

### Ziel

Das Backend liefert einen sauberen, normalisierten Produktkontext fuer alle Clients.

### Aufgaben

1. Plan-, Addon- und Rollenlogik in einem zentralen Service kapseln.
2. Capability-Endpoint bauen.
3. Shared Registry fuer Features aufbauen.
4. Bestehende Plan- und Addon-Pruefungen auf die Registry umstellen.
5. `GET /api/auth/me` optional um die wichtigsten Capability-Referenzen erweitern oder
   sauber auf den neuen Endpoint verweisen.
6. Drift zwischen SaaS-Landingpage, Admin-Subscription-API und Enforcement schliessen.

### Konkrete Backend-Arbeitspakete

#### B1. Feature Registry

Ein Modul definieren, z.B.:

- `feature_keys.py`
- `capability_service.py`
- `subscription_entitlements.py`

Es soll:

- Feature-Schluessel zentral definieren
- Plan-Includes und Addons abbilden
- rollenabhaengige Ausnahmen kapseln
- Limits berechnen

#### B2. Capability Endpoint

Neuer Endpoint mit:

- Kontextdaten
- Subscription-Status
- aktiven Addons
- effektiven Features
- Limits

Tests:

- Kunde in Basic-Plan
- Admin in Professional-Plan
- Trainer mit eingeschraenkten Features
- Addon aktiviert, Plan aber ohne Inklusivleistung

#### B3. Contract Cleanup

Folgende Drift explizit schliessen:

- SaaS-Landingpage vs. echte Addon-Namen
- Platform-Admin `set-plan` Request-Schema vs. Doku
- fehlende oder veraltete Subscription-Endpunkte in der Doku
- OpenAPI und Handdokumentation angleichen

### Definition of Done

- Ein Endpoint entscheidet alle UI-relevanten Freigaben
- Web und iOS muessen keine Planlogik lokal nachbauen
- Tests decken Rollen, Plan und Addons gemeinsam ab

## 5. Workstream C — Backend Messaging and Realtime

### Ziel

Messaging modernisieren, damit Web und iOS ohne Polling und ohne Sonderlogik arbeiten.

### Aufgaben

1. `POST /api/messages/{conv_id}` bleibt autoritativer Write-Path.
2. Optionales `client_message_id` akzeptieren und in Response und Event zurueckgeben.
3. Nach erfolgreichem Persistieren `new_message` emittieren.
4. Bei `/delivered` und `/read` die Receipt-Events emittieren.
5. Praesenz in Redis oder vergleichbarem Shared Store fuehren.
6. Room-Rejoin und Presence-Lifecycle definieren.
7. Push fuer Offline-Faelle sauber von Live-State trennen.

### Konkrete Backend-Arbeitspakete

#### C1. Message Send Contract

- Request erweitert um `client_message_id`
- Response liefert vollstaendiges Message-Objekt
- Socket-Event spiegelt dieselbe Message-Struktur

#### C2. Receipt Events

In `mark_delivered` und `mark_read`:

- betroffene Nachricht-IDs ermitteln
- autorisierten Sender-Room ansprechen
- `message_receipts_updated` emitten

#### C3. Presence

- `connect` setzt online
- `disconnect` setzt offline
- Heartbeat oder TTL-basiertes Refresh
- `presence_update` fuer betroffene Konversationsteilnehmer

#### C4. Reconnect Semantik

Dokumentieren:

- welche User-Rooms global betreten werden
- welche Konversations-Rooms explizit beigetreten werden
- was bei Token-Refresh passiert

### Definition of Done

- iOS- und Web-Clients koennen Polling entfernen
- Zustell- und Lese-Status kommen ausschliesslich ueber Events nach
- Presence ist nicht mehr an zufaellige HTTP-Aktivitaet gekoppelt

## 6. Workstream D — Web Frontend Modernization

### Ziel

Das Web-Frontend nutzt dieselben Capabilities wie iOS und verlaesst sich nicht auf
historisch gewachsene Template-Sonderregeln.

### Aufgaben

1. Einen zentralen Capability Loader fuer eingeloggte Sessions einfuehren.
2. Seitenmenues, CTA-Buttons und Deep Links gegen Capabilities rendern.
3. Admin- und Kundenoberflaechen auf denselben Feature-Schluesseln aufbauen.
4. Alte Template-Checks und verteilte Plan-Ifs entfernen.
5. CTA fuer geblockte Features sauber unterscheiden:
   - nicht erlaubt
   - nicht gebucht
   - im Plan nicht enthalten

### Konkrete Web-Arbeitspakete

#### D1. Capability Bootstrap

Nach Login oder Context Switch:

- Capabilities laden
- Session-State aktualisieren
- Navigation neu aufbauen

#### D2. Route Guards

Browserseitig:

- sichtbare Navigation nur fuer erlaubte Features
- Route-Guards fuer direkte URL-Aufrufe
- konsistente Upsell- oder Lock-State-Komponenten fuer Addons

#### D3. Functional Parity Audit

Alle Web-Funktionen nach Login in Cluster auflisten:

- Dashboard
- Booking
- Kalender
- Messaging
- Forum
- Verwaltung
- Abrechnung
- Einstellungen

Fuer jeden Cluster:

- hat iOS diese Funktion?
- ist die Funktion gleichwertig?
- wenn nein, ist sie absichtlich web-only oder fehlt sie?

### Definition of Done

- Sichtbarkeit und Zugriff basieren auf demselben Contract wie iOS
- keine verstreuten Template-Entscheidungen fuer Plan/Addons mehr
- Web dient als klare Referenz fuer fachliche Paritaet

## 7. Workstream E — iOS Capability and Navigation Refactor

### Ziel

Die iOS-App zeigt nur Funktionen an, die im aktuellen Nutzer- und Schulkontext erlaubt
sind, und erreicht funktionale Paritaet mit den Web-Funktionen nach Login.

### Aufgaben

1. `CapabilityStore` oder `EntitlementStore` einfuehren.
2. `UserInfo` um Plan-, Addon- und Feature-Daten erweitern.
3. TabBar, Menues und Deep Links capability-driven machen.
4. Role-only-Logik aus Views und ViewModels entfernen.
5. Gesperrte Features sauber ausblenden oder als locked darstellen.

### Konkrete iOS-Arbeitspakete

#### E1. App State

- Nach Login Capabilities laden
- Nach Context Switch Capabilities neu laden
- bei Refresh oder Reconnect Zustand konsistent halten

#### E2. Navigation

Folgende Bereiche auf Capability-Gating umstellen:

- `MainTabView`
- `MoreMenuView`
- Admin-Verwaltung
- Forum
- Messaging-Zusaetze
- Einstellungen und Zusatzmodule

#### E3. Feature Parity Matrix

Ein iOS-seitiger Abarbeitungsschnitt:

- alle Web-Funktionen nach Login inventarisieren
- pro Funktion Status markieren:
  - vorhanden
  - vorhanden, aber unvollstaendig
  - fehlt
  - bewusst web-only
- fehlende Funktionen nach fachlichem Wert priorisieren

#### E4. Remove Workarounds

Entfernen oder refaktorieren:

- Role-only Gating
- API-Probes nur zur Rechte-Erkennung
- lokale Sonderpfade fuer fehlende Contracts
- starre statische Menues ohne Capability-Kontext

### Definition of Done

- iOS-Navigation folgt dem Capability-Contract
- Nutzer sehen keine Features, die ihre Schule nicht gebucht hat
- Mehrrollen-User und Context Switches funktionieren konsistent

## 8. Workstream F — iOS Messaging Architecture

### Ziel

Messaging auf robuste, ereignisgetriebene Architektur umstellen.

### Aufgaben

1. `WebSocketManager` auf Transport reduzieren.
2. `ChatStore` oder `ChatService` als zentrale State-Instanz einfuehren.
3. Nachrichten pro Konversation verwalten.
4. Optimistische Sends mit `client_message_id` reconciliieren.
5. Rejoin-Logik und Reconnect-Faehigkeit sauber modellieren.
6. Push und Deep Links sauber integrieren.

### Konkrete iOS-Arbeitspakete

#### F1. Datenmodell

Message-Model erweitern um:

- `conversationId`
- `clientMessageId`
- `deliveryStatus`
- `deliveredAt`
- `readAt`

#### F2. ChatStore

Zentral verantwortlich fuer:

- Konversationsliste
- Nachrichten pro Konversation
- Pending-Outbox
- Room-Mitgliedschaften
- Receipt-Updates
- Presence-Updates

#### F3. Event-Routing

Socket-Events mappen auf:

- neue Nachricht
- Receipt-Update
- Konversationsupdate
- Presence-Update
- optional Typing

#### F4. Push Integration

- Push-Token nach Login registrieren
- Deeplink-Ziel im App-State halten
- bei App-Start oder Resume korrekt in Konversation navigieren

### Definition of Done

- kein 10-Sekunden-Polling fuer aktive Chats mehr
- optimistic UI wird sauber bestaetigt oder zurueckgesetzt
- Reconnect fuehrt zum gleichen Chat-State wie vor Verbindungsverlust

## 9. Workstream G — Testing, Rollout and Quality Gates

### Ziel

Die Modernisierung sicher ausrollen und Contract-Drift dauerhaft verhindern.

### Aufgaben

1. Contract-Tests fuer Capability-Endpoint
2. Contract-Tests fuer Messaging-REST und Socket-Payloads
3. Web-UI-Tests fuer Sichtbarkeit pro Plan/Addon/Rolle
4. iOS-UI-Tests fuer Navigation und Feature-Gating
5. End-to-end Testfaelle fuer Kernprozesse

### Minimale E2E-Faelle

1. Admin mit Messaging-Addon sieht Messaging in Web und iOS
2. Kunde ohne Forum-Addon sieht kein Forum in Web und iOS
3. Trainer mit Kontextwechsel behaelt nur erlaubte Bereiche
4. Neue Nachricht erscheint live in zweitem Client
5. Delivered/Read-Status synchronisieren ohne Polling
6. Nach Upgrade des Plans werden neue Features in beiden Clients sichtbar

### Rollout

#### Phase 1

- Shared docs konsolidieren
- Capability-Endpoint implementieren
- OpenAPI aktualisieren

#### Phase 2

- Web capability-driven umstellen
- iOS capability-driven umstellen

#### Phase 3

- Messaging-Contracts und Realtime modernisieren
- Polling entfernen

#### Phase 4

- Restliche Paritaetsluecken schliessen
- technische Schulden entfernen

## 10. Definition of Done fuer das Gesamtprojekt

Das Projekt ist abgeschlossen, wenn:

- gemeinsame Specs nur noch im Shared-Repo gepflegt werden
- Web und iOS denselben Capability-Contract konsumieren
- alle nach Login relevanten Web-Funktionen entweder in iOS vorhanden oder bewusst
  als web-only dokumentiert sind
- Addons und Plansichtbarkeit serverseitig berechnet und clientseitig konsistent dargestellt werden
- Messaging ohne Polling, Rechte-Probes oder andere Workarounds funktioniert
- OpenAPI, Shared-Spec und Implementierung denselben Stand haben
