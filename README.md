# PawCoach Shared Docs

Zentrale, plattformuebergreifende Dokumentation fuer PawCoach.

Dieses Repository ist die einzige kanonische Quelle fuer gemeinsame Spezifikationen,
API-Contracts und plattformuebergreifende Statusdokumente. Es wird als Git-Submodule in
die Produkt-Repositories eingebunden:

- Backend + Web: `Dog-School-Manager/docs/shared/`
- iOS: `PawCoach-iOS/docs/shared/`

## Kanonische Artefakte

| Datei | Beschreibung |
|-------|-------------|
| `CROSS-PLATFORM-FEATURE-SPEC.md` | Gemeinsame Produkt-, Feature- und Contract-Spezifikation fuer Backend, Web und iOS |
| `CROSS-PLATFORM-SYNC-STRATEGY.md` | Regeln fuer Ownership, Contract-first Entwicklung und Plan-Lifecycle |
| `BACKEND-DEVELOPER-GUIDE.md` | Backend-Leitfaden fuer Pricing, Entitlements, Standorte und Realtime |
| `WEB-DEVELOPER-GUIDE.md` | Web-Leitfaden fuer capability-driven Navigation und Pricing-Integration |
| `IOS-DEVELOPER-GUIDE.md` | iOS-Leitfaden fuer CapabilityStore, Realtime und Contract-Nutzung |
| `openapi.json` | Kanonische, vom Backend generierte REST-Spezifikation |
| `reports/` | Stabile Audits, Alignment- und Statusberichte |
| `plans/` | Temporare Design- und Umsetzungsplaene fuer noch offene Arbeit |

## Arbeitsregel

Gemeinsame Spezifikationen, OpenAPI, cross-platform Reports und aktive Plaene werden nur
hier gepflegt. Die Produkt-Repositories enthalten dafuer nur das Submodule und optionale
Repo-spezifische Guide-Dokumente.

## Lifecycle fuer Plaene

- `plans/` ist fuer aktive oder noch offene Arbeit gedacht.
- Sobald ein Plan fachlich umgesetzt und in stabile Doku ueberfuehrt wurde, wird er
  geloescht.
- Der dauerhafte Referenzstand liegt dann in `CROSS-PLATFORM-*.md`, `openapi.json`
  oder unter `reports/`.

## Workflow

1. Aenderungen in diesem Repository vornehmen.
2. Aenderungen hier committen.
3. Anschliessend in den Projekt-Repositories das Submodule aktualisieren:

```bash
git submodule update --remote docs/shared
git add docs/shared
git commit -m "docs: update shared docs submodule"
```

## Zielbild

- Eine Spezifikation
- Ein REST-Contract
- Ein Capability- und Realtime-Contract
- Ein gemeinsamer Dokumentationsstand fuer Backend, Web und iOS
- Keine veralteten Parallelkopien oder erledigten Plan-Dateien
