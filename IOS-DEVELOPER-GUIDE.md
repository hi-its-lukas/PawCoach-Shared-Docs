# iOS-Entwickler Anleitung — Shared Docs & API Contract

**Stand:** 2026-03-16
**Betrifft:** Alle die am PawCoach-iOS Repo arbeiten

---

## Was hat sich geaendert?

Die Cross-Platform-Dokumentation (Feature-Spec, Sync-Strategy, OpenAPI-Spec) liegt jetzt in einem **eigenen Repo** (`PawCoach-Shared-Docs`) und wird per **Git Submodule** in beide Projekt-Repos eingebunden. So gibt es nur noch **eine einzige Quelle** fuer geteilte Docs — kein manuelles Kopieren mehr.

**Repo-Struktur jetzt:**
```
PawCoach-iOS/
├── docs/
│   ├── shared/                          ← Git Submodule (PawCoach-Shared-Docs)
│   │   ├── CROSS-PLATFORM-FEATURE-SPEC.md
│   │   ├── CROSS-PLATFORM-SYNC-STRATEGY.md
│   │   ├── openapi.json                 ← Auto-generierte API-Spec (249 Pfade)
│   │   └── plans/
│   │       └── 2026-03-16-ios-backend-sync-fixes.md
│   ├── CROSS-PLATFORM-FEATURE-SPEC.md   ← Symlink → shared/...
│   ├── backend-api.md
│   └── ...
```

---

## Einmaliges Setup (nach git pull)

Beim ersten Pull nach der Umstellung:

```bash
cd PawCoach-iOS
git pull origin main
git submodule init
git submodule update
```

Danach ist `docs/shared/` mit den aktuellen Docs befuellt.

**Alternativ** (wenn du das Repo frisch klonst):

```bash
git clone --recurse-submodules git@github.com:hi-its-lukas/PawCoach-iOS.git
```

---

## Taeglicher Workflow

### Shared Docs aktualisieren (regelmaessig machen!)

```bash
cd PawCoach-iOS
git submodule update --remote docs/shared
git add docs/shared
git commit -m "docs: shared docs aktualisiert"
git push
```

**Wann:** Immer wenn du weisst, dass Backend-Aenderungen an der Spec gemacht wurden, oder mindestens **vor jedem Feature-Start**.

### Geteilte Docs aendern

Aenderungen an `CROSS-PLATFORM-FEATURE-SPEC.md` oder anderen geteilten Docs werden **nicht** direkt im iOS-Repo gemacht, sondern:

```bash
cd docs/shared
# ... Datei bearbeiten ...
git add -A
git commit -m "docs: Beschreibung der Aenderung"
git push origin main
cd ../..
git add docs/shared
git commit -m "docs: shared docs aktualisiert"
git push
```

Danach muss das **Backend-Repo** das Submodule ebenfalls updaten (Backend-Entwickler informieren).

### iOS-eigene Docs aendern

Dateien die **nicht** in `docs/shared/` liegen (z.B. `docs/backend-api.md`, Audit-Reports) werden ganz normal im iOS-Repo bearbeitet — da aendert sich nichts.

---

## OpenAPI Spec nutzen

Die Datei `docs/shared/openapi.json` enthaelt die **komplette Backend-API** im OpenAPI 3.0 Format (249 Pfade, 308 Operationen). Du kannst sie nutzen fuer:

1. **Endpoint-Pfade pruefen:** Stimmt mein iOS-Pfad mit dem Backend ueberein?
2. **Request/Response-Struktur:** Welche Felder erwartet der Endpoint?
3. **Code-Generierung:** Tools wie `swift-openapi-generator` koennen Stubs generieren

**Schnell einen Endpoint nachschlagen:**
```bash
cat docs/shared/openapi.json | python3 -m json.tool | grep -A 5 "/api/messages/info"
```

---

## DRINGENDE FIXES (vor naechstem Release!)

In `docs/shared/plans/2026-03-16-ios-backend-sync-fixes.md` sind **6 kritische Mismatches** dokumentiert, die aktuell zu 404/400-Fehlern fuehren:

| # | Was | Wo in Xcode | Fix |
|---|-----|-------------|-----|
| 1 | `markDelivered` Pfad | `Endpoint+Messaging.swift` | `/mark-delivered` → `/delivered` |
| 2 | `messageInfo` Pfad | `Endpoint+Messaging.swift` | `/{id}/info` → `/info/{id}` |
| 3 | TOTP Verify | `Endpoint+Auth.swift` | `setup_token` Parameter hinzufuegen |
| 4 | Forum Thread | `Endpoint+Messaging.swift` | `content` → `body` |
| 5 | Poll Erstellung | `Endpoint+Messaging.swift` | `{question, options}` → `{title, questions: [{question_text, question_type, options}]}` |
| 6 | Registrierung | `Endpoint+Auth.swift` | `gdpr_consent` + `agb_consent` hinzufuegen |

**Details mit Code-Beispielen:** Siehe `docs/shared/plans/2026-03-16-ios-backend-sync-fixes.md`

---

## Checkliste

- [ ] `git submodule init && git submodule update` ausgefuehrt
- [ ] `docs/shared/` Ordner ist befuellt (nicht leer)
- [ ] Symlink `docs/CROSS-PLATFORM-FEATURE-SPEC.md` funktioniert (`cat` zeigt Inhalt)
- [ ] Fix 1-6 aus Sync-Fixes-Doku umgesetzt
- [ ] Jeden gefixten Endpoint gegen Backend getestet

---

## Bei Problemen

**Submodule ist leer / fehlt:**
```bash
git submodule update --init --recursive
```

**Symlink funktioniert nicht (Windows):**
Git Symlinks muessen unter Windows aktiviert sein:
```bash
git config --global core.symlinks true
```
Dann Repository neu klonen.

**Merge-Konflikt im Submodule:**
```bash
cd docs/shared
git checkout main
git pull origin main
cd ../..
git add docs/shared
git commit -m "docs: shared submodule conflict resolved"
```
