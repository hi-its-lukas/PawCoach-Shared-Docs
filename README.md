# PawCoach Shared Docs

Zentrale, plattformuebergreifende Dokumentation fuer PawCoach.

Dieses Repository ist die einzige Quelle fuer gemeinsame Spezifikationen, API-Contracts
und Umsetzungsplaene. Es wird als Git-Submodule in die Projekt-Repositories eingehaengt:

- Backend + Web: `Dog-School-Manager/docs/shared/`
- iOS: `PawCoach-iOS/docs/shared/`

## Inhalt

| Datei | Beschreibung |
|-------|-------------|
| `CROSS-PLATFORM-FEATURE-SPEC.md` | Gemeinsame Produkt- und Feature-Spezifikation fuer Backend, Web und iOS |
| `CROSS-PLATFORM-SYNC-STRATEGY.md` | Prozess, Contracts und Release-Regeln fuer synchronisierte Entwicklung |
| `IOS-DEVELOPER-GUIDE.md` | Arbeitsanleitung fuer iOS-Entwicklung gegen die geteilten Contracts |
| `openapi.json` | Kanonische, vom Backend generierte API-Spezifikation |
| `plans/` | Geteilte Design-, Audit- und Implementierungsplaene |

## Arbeitsregel

Gemeinsame Spezifikationen, OpenAPI und cross-platform Plaene werden nur hier gepflegt.
Die Projekt-Repositories enthalten dafuer nur das Submodule und optionale Zeiger-Dokumente.

## Workflow

1. Aenderungen in diesem Repository vornehmen.
2. Aenderungen hier committen.
3. Anschliessend in den Projekt-Repositories das Submodule auf den neuen Stand ziehen:

```bash
git submodule update --remote docs/shared
git add docs/shared
git commit -m "docs: update shared docs submodule"
```

## Zielbild

- Eine Spezifikation
- Ein API-Contract
- Ein gemeinsamer Umsetzungsplan fuer Backend, Web und iOS
- Keine parallelen Kopien mehr in mehreren Repositories
