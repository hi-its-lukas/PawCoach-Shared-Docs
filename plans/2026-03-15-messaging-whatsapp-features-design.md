# Messaging WhatsApp-Features Design

**Datum:** 2026-03-15
**Status:** Genehmigt

## Ziel

PawCoach Messaging auf WhatsApp-Niveau bringen: besseres Bubble-Layout, Nachrichteninfo, Kontaktinfo-Seite, Hunde-Kontakte, Sichtbarkeitseinstellungen, Zitieren, Suche und Zuletzt-Online.

---

## 1. Bubble-Layout (Inline Time)

**Aktuell:** Uhrzeit + Häkchen stehen in separatem HStack unter der Bubble.

**Neu:** Uhrzeit + Häkchen werden inline unten rechts IN der Bubble dargestellt (wie WhatsApp). Der Text fließt um die Zeitanzeige herum.

```
┌─────────────────────────────┐
│ Nachrichtentext hier und    │
│ dann weiter           10:22 ✓✓│
└─────────────────────────────┘
```

- Eigene Nachrichten: Zeit + Delivery-Status-Icon
- Empfangene Nachrichten: Nur Zeit
- Bei Media-Nachrichten: Zeit als Overlay unten rechts mit leichtem Schatten

## 2. Nachrichteninfo (Message Info Sheet)

Erreichbar via Swipe-Left auf eigene Nachricht oder Context-Menu "Info".

**Layout (wie WhatsApp):**
- Oben: Die Nachricht als Bubble dargestellt
- Darunter schlichte Liste:
  - ✓✓ (blau) **Gelesen** — Datum + Uhrzeit (oder "—")
  - ✓✓ (grau) **Zugestellt** — Datum + Uhrzeit

**Hinweis:** Backend trackt aktuell keinen Gelesen/Zugestellt-Status. Zeiten basieren auf `created_at`. Sobald Backend Delivery-Tracking liefert, können echte Zeiten angezeigt werden.

## 3. Kontaktinfo-Seite

Erreichbar via Tap auf den Namen in der NavigationBar des Chats.

### Layout

```
┌──────────────────────────────┐
│        [Avatar/Initialen]     │
│         Lukas H.              │
│          Kunde                │
│     zuletzt online 14:32      │
│                               │
│  ┌──────────────────────────┐ │
│  │ 📷 Medien, Links, Doks  42│ │
│  └──────────────────────────┘ │
│                               │
│  🐕 Hunde                     │
│     Bello (Labrador)          │
│       Weitere Halter:         │
│       👤 Lisa Mustermann   >  │
│     Luna (Schäferhund)        │
│                               │
│  📚 Gemeinsame Kurse          │
│     Welpengruppe Mo 10:00     │
│     Agility Fr 16:00          │
│                               │
│  ┌──────────────────────────┐ │
│  │ 🔔 Benachrichtigungen    │ │
│  │ 🗑 Chat leeren           │ │
│  └──────────────────────────┘ │
│                               │
│  ┌──────────────────────────┐ │
│  │ 🚫 Blockieren       (rot)│ │
│  │ ⚠️  Melden           (rot)│ │
│  └──────────────────────────┘ │
└──────────────────────────────┘
```

### Datenquellen

- **Profildaten:** Aus Konversations-Members oder neuer Endpoint
- **Hunde + Halter:** `/api/admin/dogs` (Admin/Trainer) oder `/api/customer/dogs` (Kunde)
- **Gemeinsame Kurse:** Neuer Endpoint oder aus bestehenden Kurs-Daten ableiten
- **Medien-Zähler:** Lokales Zählen der Nachrichten mit `mediaUrl != nil`
- **Tap auf Halter:** Öffnet dessen Kontaktinfo oder startet neuen Chat

### Sichtbarkeit

- Trainer/Admin sehen immer alle Daten
- Kunden sehen nur was die Profil-Sichtbarkeitseinstellungen des anderen erlauben

## 4. Hunde in der Kontaktauswahl

### Darstellung in NewConversationView

```
👤 Personen
   Lukas Hengl                    Trainer
   Max Mustermann                 Kunde

🐕 Hunde
   Bello (Max Mustermann, Lisa M.)
   Luna (Anna Schmidt)
```

### Verhalten

- Tap auf Hund → Gruppen-Konversation mit allen Haltern erstellen
- Betreff automatisch auf Hundenamen setzen
- Datenquelle:
  - Admin/Trainer: `/api/admin/dogs` (alle Hunde)
  - Kunde: `/api/customer/dogs` (eigene Hunde)

## 5. Kontakt-Sichtbarkeit

### Wer sieht wen in der Kontaktliste

| Rolle | Sieht |
|-------|-------|
| Admin/Trainer | Alle Personen + alle Hunde |
| Kunde (immer) | Trainer/Admins + eigene Hunde-Halter + Kunden im selben Kurs |
| Kunde (opt-in) | + alle anderen Kunden mit `discoverable = true` |

### Discoverable-Setting

- Opt-in bei Ersteinrichtung: "Moechtest du von anderen Hundeschulmitgliedern gefunden und kontaktiert werden koennen?"
- Jederzeit aenderbar in Einstellungen
- Backend speichert `discoverable: Bool` am User-Profil
- Kontaktliste filtert entsprechend

## 6. Profil-Sichtbarkeitseinstellungen

### Einstellbare Felder

| Information | Standard | Optionen |
|-------------|----------|----------|
| Vorname | Immer sichtbar | Nicht aenderbar |
| Nachname | Voll sichtbar | Voll / Nur Initial ("Lukas H.") |
| Profilbild | Sichtbar | Alle / Nur Trainer / Niemand |
| Meine Hunde | Sichtbar fuer Trainer | Alle / Nur Trainer / Niemand |
| Weitere Halter meiner Hunde | Sichtbar fuer Trainer | Alle / Nur Trainer / Niemand |
| Telefonnummer | Versteckt | Alle / Nur Trainer / Niemand |

- Trainer/Admin sehen immer alles (benoetigen Daten fuer ihre Arbeit)
- Einstellungen werden im Backend gespeichert und bei Kontakt-Abfragen beruecksichtigt

## 7. Zuletzt Online

- Anzeige unter dem Namen im Chat-Header: "online" oder "zuletzt online um 14:32"
- Datenquelle: `last_seen` Feld vom Backend (ggf. neuer Endpoint)
- Sichtbarkeitseinstellung: Alle / Nur Trainer / Niemand

## 8. Nachricht Zitieren (Reply)

- Swipe nach rechts auf eine Nachricht → Reply-Modus
- Ueber dem Eingabefeld erscheint eine Vorschau der zitierten Nachricht
- Gesendete Nachricht zeigt die zitierte Nachricht als kompakten Block ueber dem eigenen Text:

```
┌─────────────────────────────┐
│ ┃ Max Mustermann             │
│ ┃ Wann ist die naechste...   │
│                              │
│ Morgen um 10 Uhr!    10:23 ✓│
└─────────────────────────────┘
```

- Backend muss `reply_to` Feld unterstuetzen (Message-ID der zitierten Nachricht)
- Falls Backend das noch nicht hat: Frontend vorbereiten, Zitat als Text-Prefix senden

## 9. Suche im Chat

- Lupe-Icon in der Toolbar des Chats
- Oeffnet Suchleiste oben
- Filtert lokal durch geladene Nachrichten
- Treffer werden hervorgehoben, Pfeile zum Navigieren zwischen Treffern
- Suche scrollt zum jeweiligen Treffer

## Backend-Anforderungen

Einige Features benoetigen neue oder erweiterte Backend-Endpoints:

| Feature | Backend-Aenderung |
|---------|-------------------|
| Discoverable-Setting | `PATCH /api/auth/profile` mit `discoverable` Feld |
| Profil-Sichtbarkeit | `PATCH /api/auth/profile` mit Sichtbarkeits-Feldern |
| Kontaktliste erweitert | `/api/messages/contacts` muss Kurs-Mitglieder + discoverable filtern |
| Hunde in Kontakten | Bestehende Endpoints genuegen |
| Blockieren | `POST /api/messages/block/<user_id>` (neu) |
| Melden | `POST /api/messages/report/<user_id>` (neu) |
| Zuletzt online | `last_seen` Feld in User-Response |
| Reply/Zitieren | `reply_to` Feld in Message-Model |
| Chat leeren | `DELETE /api/messages/<conversation_id>/clear` (neu) |
| Medien-Zaehler | Lokal zaehlen oder `/api/messages/<id>/media-count` |
| Gelesen/Zugestellt | Delivery-Tracking Endpoints (Zukunft) |

## Priorisierung

### Phase 1 — Rein Frontend (sofort umsetzbar)
1. Bubble-Layout inline
2. Nachrichteninfo-Sheet verbessern
3. Suche im Chat
4. Nachricht zitieren (Frontend-Teil)
5. Stumm schalten (lokal UserDefaults)

### Phase 2 — Neue Views + einfaches Backend
6. Kontaktinfo-Seite
7. Hunde in Kontaktauswahl
8. Chat leeren
9. Zuletzt online

### Phase 3 — Backend-Erweiterungen
10. Discoverable-Setting + Opt-in
11. Profil-Sichtbarkeitseinstellungen
12. Blockieren / Melden
13. Reply/Zitieren (Backend-Teil)
