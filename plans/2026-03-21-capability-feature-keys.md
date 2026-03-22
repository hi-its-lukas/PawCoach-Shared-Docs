# Backend-Arbeit: Capability Feature-Keys fuer Plan/Addon-Gating

Datum: 2026-03-21

Die iOS-App (und Web-App) nutzen `GET /api/auth/capabilities` um Features basierend auf Abo-Plan und gebuchten Addons ein-/auszublenden. Die `features`-Map im Response muss folgende Keys liefern.

---

## Bestehende Feature-Keys (bereits implementiert)

| Key | Beschreibung | Typ |
|-----|-------------|-----|
| `admin_dashboard` | Admin-Dashboard und Tagesansicht | Plan |
| `trainer_calendar` | Trainer-Wochen- und Monatskalender | Plan |
| `booking` | Kunden-Buchungssystem | Plan |
| `messaging` | Nachrichten-System | Plan |
| `community_forum` | Community-Forum | Plan/Addon |
| `forum_moderation` | Forum-Moderationsrechte | Rolle |
| `voucher_system` | Gutschein-System | Addon |
| `stripe_payments` | Stripe Connect Integration | Plan |
| `accounting_integration` | Buchhaltungs-Integration | Addon |
| `admin_accounting` | Buchhaltungs-Zugang (alternativ) | Addon |

## Neue Feature-Keys (Backend muss liefern)

Diese Keys werden ab sofort in der iOS-App geprueft. Das Backend muss sie im `features`-Dict des Capability-Responses setzen, basierend auf dem Abo-Plan und den aktiven Addons der Hundeschule.

| Key | Beschreibung | Empfohlener Typ | Anzeige wenn `true` |
|-----|-------------|-----------------|---------------------|
| `credit_system` | Guthabenkarten-System (5er/10er-Karten) | Plan | Admin: Guthabenkarten-Verwaltung. Kunde: Guthaben-Uebersicht |
| `invoicing` | Rechnungssystem | Plan | Admin: Rechnungs-Verwaltung. Kunde: Rechnungen einsehen |
| `shop` | Kunden-Shop (Karten/Pakete kaufen) | Plan | Kunde: Shop/Purchase-Bereich |
| `direct_booking` | Admin-Direktbuchung fuer Kunden | Plan/Addon | Admin: Direktbuchungs-Tool |
| `availability_management` | Verfuegbarkeits-Verwaltung | Plan | Admin: Verfuegbarkeiten. Trainer: Verfuegbarkeit verwalten |
| `booking_requests` | Buchungs-/Stornierungsanfragen | Plan | Admin: Anfragen-Verwaltung |
| `support_access` | PawCoach-Support-Zugriff | Plan | Admin: Support-Zugriff verwalten |
| `email_templates` | E-Mail-Vorlagen anpassen | Addon | Admin: E-Mail-Vorlagen bearbeiten |
| `survey_system` | Umfragen erstellen und auswerten | Addon | Admin: Umfrage-Dashboard + Umfragen |
| `review_system` | Bewertungen verwalten | Addon | Admin: Bewertungs-Verwaltung |
| `calendar_subscription` | Kalender-Abo (iCal/WebCal) | Plan | Trainer + Kunde: Kalender abonnieren |

## Implementierung im Backend

### Empfohlene Zuordnung zu Abo-Plaenen

```
Starter-Plan:
  booking: true
  credit_system: true
  invoicing: true
  shop: true
  booking_requests: true
  availability_management: true
  calendar_subscription: true
  messaging: true
  support_access: true

Professional-Plan (alles aus Starter +):
  admin_dashboard: true
  trainer_calendar: true
  direct_booking: true
  stripe_payments: true

Enterprise-Plan (alles aus Professional +):
  email_templates: true
  (alle Addons inklusive)
```

### Addon-basierte Keys

Wenn ein Addon aktiv ist (`addons[slug].active == true`), soll der entsprechende Feature-Key automatisch auf `true` gesetzt werden:

| Addon-Slug | Feature-Key |
|------------|-------------|
| `community_forum` | `community_forum` |
| `voucher_system` | `voucher_system` |
| `accounting` | `accounting_integration` |
| `email_templates` | `email_templates` |
| `surveys` | `survey_system` |
| `reviews` | `review_system` |

### Logik

```python
# Pseudocode fuer build_features()
features = {}

# Plan-Features setzen
for key in plan.included_features:
    features[key] = True

# Addon-Features hinzufuegen
for addon in school.active_addons:
    features[addon.feature_key] = True

# In Plan inkludierte Addons
for addon_key, info in plan.included_addons.items():
    features[addon_key] = True
```

## Wichtig

- Wenn ein Feature-Key NICHT im `features`-Dict vorhanden ist, wird er als `false` behandelt (Feature ausgeblendet)
- Die iOS-App zeigt **nur** Features an, deren Key `true` ist
- Core-Features die immer sichtbar sein sollen (Abo & Addons, Setup-Assistent, Schuleinstellungen, Konto) werden NICHT gegated und brauchen keinen Feature-Key
