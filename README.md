# PawCoach Shared Docs

Geteilte Cross-Platform-Dokumentation fuer PawCoach.

Dieses Repo wird als **Git Submodule** in beiden Projekt-Repos eingebunden:
- Backend: `Dog-School-Manager/docs/shared/`
- iOS: `PawCoach-iOS/docs/shared/`

## Inhalt

| Datei | Beschreibung |
|-------|-------------|
| `CROSS-PLATFORM-FEATURE-SPEC.md` | Zentrale Feature-Spezifikation (Backend + Web + iOS) |
| `CROSS-PLATFORM-SYNC-STRATEGY.md` | Synchronisations-Strategie zwischen Plattformen |
| `plans/` | Plattformuebergreifende Implementierungsplaene |

## Workflow

1. Aenderungen an geteilten Docs **immer hier** vornehmen
2. In beiden Projekt-Repos das Submodule updaten:
   ```bash
   git submodule update --remote docs/shared
   git add docs/shared
   git commit -m "docs: shared docs aktualisiert"
   ```
3. CI prueft automatisch, ob das Submodule aktuell ist
