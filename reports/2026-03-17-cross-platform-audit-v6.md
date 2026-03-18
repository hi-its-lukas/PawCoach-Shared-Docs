# Cross-Platform Audit v6

**Datum:** 2026-03-17
**Scope:** Vor Login `Frontend + Backend`, nach Login `Backend + Web + iOS`
**Quellen:** Shared-Docs, aktueller Code in `Dog-School-Manager` und `PawCoach-iOS`

## Kurzfazit

Vor dem Login ist PawCoach fachlich weitgehend konsistent. Die groesste Luecke liegt dort nicht in Auth oder Registrierung, sondern im Pricing-Katalog: Preise, Paketinhalte und Upsell-Logik sind noch an mehreren Stellen hart codiert.

Nach dem Login ist die technische Grundlage deutlich besser als in den vorherigen Audits: der Capability-Endpoint existiert, die Realtime-Basis fuer Messaging ist vorhanden, und iOS hat die zentrale Infrastruktur fuer Capabilities und Chat-State. Die eigentliche Paritaetsluecke liegt jetzt vor allem im letzten Schritt der Konsolidierung: Web und iOS konsumieren den Capability-Contract noch nicht konsequent genug.

## Positiv verifiziert

- Der kanonische Capability-Contract ist im Backend implementiert: [blueprints/api_auth.py](/Users/luhengl/Documents/GitHub/Dog-School-Manager/blueprints/api_auth.py#L958), [capability_service.py](/Users/luhengl/Documents/GitHub/Dog-School-Manager/capability_service.py).
- iOS laedt Capabilities nach Login, Reconnect und Kontextwechsel: [AppState.swift](/Users/luhengl/Documents/GitHub/PawCoach-iOS/PawCoach/App/AppState.swift#L110), [CapabilityStore.swift](/Users/luhengl/Documents/GitHub/PawCoach-iOS/PawCoach/Core/Capabilities/CapabilityStore.swift).
- Messaging-Realtime ist architektonisch sauberer als im v5-Audit: REST als Write-Path, Socket.IO fuer Live-Events, ChatStore/WebSocketManager-Trennung.
- Die fruehere iOS-Luecke bei `POST /api/customer/einzelstunde/anfrage/{id}/cancel` ist inzwischen geschlossen: [Endpoint+Customer.swift](/Users/luhengl/Documents/GitHub/PawCoach-iOS/PawCoach/Core/Network/Endpoint+Customer.swift).

## Findings

### [P1] Voucher-Entitlement verwendet im Capability-Contract den falschen Addon-Slug

Der Capability-Contract registriert das Voucher-Feature mit `addon_slug = "voucher_system"`, waehrend die produktive Codebasis und das Billing durchgaengig `gutscheinsystem` verwenden. Dadurch kann der Contract fuer Voucher/Gutscheine vom echten Backend-Verhalten abweichen.

- Contract: [capability_service.py](/Users/luhengl/Documents/GitHub/Dog-School-Manager/capability_service.py#L47)
- Produktive Gates: [templates/base.html](/Users/luhengl/Documents/GitHub/Dog-School-Manager/templates/base.html#L127), [templates/customer/customer_base.html](/Users/luhengl/Documents/GitHub/Dog-School-Manager/templates/customer/customer_base.html#L136), [blueprints/api_admin/vouchers.py](/Users/luhengl/Documents/GitHub/Dog-School-Manager/blueprints/api_admin/vouchers.py)
- Seed/Stripe/Addons: [cli/seeding.py](/Users/luhengl/Documents/GitHub/Dog-School-Manager/cli/seeding.py#L494), [stripe_setup.py](/Users/luhengl/Documents/GitHub/Dog-School-Manager/stripe_setup.py#L275)

**Auswirkung:** Web und Backend koennen Gutscheine als aktiv behandeln, waehrend der Capability-Contract sie als inaktiv liefert. Genau das untergraebt die zentrale Entitlement-Logik.

### [P1] Web-Frontend ist nach dem Login noch nicht capability-driven

Die wichtigste UI nach dem Login wird im Web immer noch direkt ueber `ctx_role`, `plan_allows(...)` und `addon_active(...)` gesteuert, statt den kanonischen Capability-Contract zu konsumieren.

- Hauptnavigation: [templates/base.html](/Users/luhengl/Documents/GitHub/Dog-School-Manager/templates/base.html#L111)
- Customer-Sidebar: [templates/customer/customer_base.html](/Users/luhengl/Documents/GitHub/Dog-School-Manager/templates/customer/customer_base.html#L177)
- Weitere alte Gates per Suche: `addon_active(...)`, `plan_allows(...)` in Templates und Blueprints

**Auswirkung:** Der Capability-Endpoint existiert, ist aber fuer das Web noch nicht die bindende Sichtbarkeitsquelle. Damit bleibt die Logik doppelt und driftanfaellig.

### [P1] iOS ist nach dem Login nur teilweise capability-driven

iOS nutzt Capabilities aktuell im Wesentlichen fuer die Haupttabs. Sekundaere Navigation und Teilbereiche bleiben aber rollen- oder screen-basiert.

- Tabs capability-driven: [MainTabView.swift](/Users/luhengl/Documents/GitHub/PawCoach-iOS/PawCoach/App/MainTabView.swift#L48)
- `More`-Menue noch rollenbasiert, inklusive Forum und Gutscheine ohne Capability-Checks: [MoreMenuView.swift](/Users/luhengl/Documents/GitHub/PawCoach-iOS/PawCoach/Features/Customer/MoreMenuView.swift#L15), [MoreMenuView.swift](/Users/luhengl/Documents/GitHub/PawCoach-iOS/PawCoach/Features/Customer/MoreMenuView.swift#L147), [MoreMenuView.swift](/Users/luhengl/Documents/GitHub/PawCoach-iOS/PawCoach/Features/Customer/MoreMenuView.swift#L221)
- Admin-Hub zeigt plan-/addon-sensitive Bereiche ungefiltert: [AdminVerwaltungView.swift](/Users/luhengl/Documents/GitHub/PawCoach-iOS/PawCoach/Features/Admin/AdminVerwaltungView.swift#L40)
- Forum-Moderation basiert weiter auf lokalem Rollenstring aus der Keychain statt Capability-Contract: [ForumViewModel.swift](/Users/luhengl/Documents/GitHub/PawCoach-iOS/PawCoach/Features/Forum/ForumViewModel.swift#L17)

**Auswirkung:** iOS kann weiterhin Screens oder Aktionen zeigen, die fuer den aktuellen Schulkontext zwar technisch existieren, fachlich aber nicht freigeschaltet sind.

### [P2] Pricing-Katalog ist vor dem Login an mehreren Stellen dupliziert und nicht aus einer Quelle abgeleitet

Die Paketdefinitionen fuer Marketing, Registrierung und Abo-Aktivierung leben in einer statischen Python-Liste `PLANS`. Gleichzeitig existieren getrennte Preisverwaltung, Stripe-Price-IDs und Addon-Preisobjekte.

- Statische Plandefinition: [saas.py](/Users/luhengl/Documents/GitHub/Dog-School-Manager/blueprints/saas.py#L90)
- Landingpage rendert direkt aus `PLANS`: [saas.py](/Users/luhengl/Documents/GitHub/Dog-School-Manager/blueprints/saas.py#L171), [landing.html](/Users/luhengl/Documents/GitHub/Dog-School-Manager/templates/saas/landing.html#L660)
- Registrierung zieht ebenfalls `PLANS` direkt: [auth.py](/Users/luhengl/Documents/GitHub/Dog-School-Manager/blueprints/auth.py#L386), [register.html](/Users/luhengl/Documents/GitHub/Dog-School-Manager/templates/auth/register.html#L20)

**Auswirkung:** Plattform-Admin kann Preise und Addons aendern, aber die oeffentliche Preis- und Registrierungsdarstellung bleibt trotzdem potenziell veraltet. Vor-Login-Frontend und Billing-Backend koennen damit auseinanderlaufen.

### [P2] CapabilityStore faellt fail-open zurueck

Wenn das Laden der Capabilities fehlschlaegt, behandelt iOS Features effektiv als erlaubt.

- Fail-open bei `isFeatureEnabled`: [CapabilityStore.swift](/Users/luhengl/Documents/GitHub/PawCoach-iOS/PawCoach/Core/Capabilities/CapabilityStore.swift#L21)
- Fehlerpfad markiert nur `isLoaded = true`, aber setzt keine restriktive Ersatzlogik: [CapabilityStore.swift](/Users/luhengl/Documents/GitHub/PawCoach-iOS/PawCoach/Core/Capabilities/CapabilityStore.swift#L50)

**Auswirkung:** Bei Backend- oder Netzwerkfehlern kann iOS zu viele Funktionen anzeigen und erst spaeter ueber API-Fehler korrigiert werden.

### [P3] Shared-Docs enthalten noch veraltete Statusaussagen

Mindestens die Sync-Strategie enthaelt noch einen Statusblock, der Capability-Endpoint und Realtime-Backend-Arbeit als offen beschreibt, obwohl diese Basis inzwischen implementiert ist.

- Veralteter Statusblock: [CROSS-PLATFORM-SYNC-STRATEGY.md](/Users/luhengl/Documents/GitHub/PawCoach-Shared-Docs/CROSS-PLATFORM-SYNC-STRATEGY.md)

**Auswirkung:** Teams koennen bei Planung und Review auf ueberholte Annahmen schauen.

## Vor-Login Audit

### Fachlich konsistent

- Login, Passwort-Reset, E-Mail-Bestaetigung, Registrierung und Consent-Flows sind im Web mit dem Backend grundsaetzlich konsistent.
- Oeffentliche Schulsuche und Join-Code-Flows sind fachlich sauber getrennt und passen zum Backend.

### Hauptluecke

- Der oeffentliche Pricing- und Registrierungsbereich arbeitet noch nicht gegen einen kanonischen Pricing-Katalog.

## Nach-Login Audit

### Backend

- Gute Grundlage: Capability-Contract, Realtime-Events, zentrale Plan-/Addon-Services.
- Offene Schwachstelle: Slug- und Namensdrift bei Entitlements.

### Web

- Fachlich breit abgedeckt.
- Hauptproblem ist keine fehlende Funktion, sondern alte Sichtbarkeitslogik ausserhalb des Capability-Contracts.

### iOS

- Gute Fortschritte bei Realtime, Navigation und zentralem State.
- Hauptproblem ist unvollstaendige Capability-Nutzung in Untermenues, Admin-Hub und Forum.

## Empfohlene Reihenfolge

1. Voucher-/Addon-Slug-Kanonisierung entscheiden und im Contract, Billing und Clients vereinheitlichen.
2. Einen zentralen Pricing-Katalog einziehen, der Landingpage, Registrierung, Checkout und Capability-Logik speist.
3. Web-Navigation und Web-Sidebars capability-driven umbauen.
4. iOS `MoreMenuView`, `AdminVerwaltungView`, Forum und weitere Sekundaermenues capability-driven machen.
5. Danach einen automatisierten Delta-Check einfuehren: Pricing-Katalog, Capability-Contract und UI-Sichtbarkeit.

## Verifikation

- Source-Audit gegen aktuellen Code in `Dog-School-Manager`, `PawCoach-iOS` und `PawCoach-Shared-Docs`
- Kein automatisierter Backend-Testlauf in dieser Runde moeglich, weil lokal zwar Python `3.14.3` vorhanden ist, aber `pytest` nicht installiert ist
