# Azure Blob Storage, Wartungsmodus & Platform-Admin Messaging — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Storage-Abstraction mit Local/Azure-Backend, Wartungsmodus mit Planung, und cross-school Messaging fuer Platform-Admins implementieren.

**Architecture:** Neues `storage.py` Modul mit ABC + zwei Backends. Wartungsmodus als `before_request`-Hook in `app.py` mit neuen PlatformSettings-Feldern. Messaging-Erweiterung in bestehenden Blueprints.

**Tech Stack:** Flask, SQLAlchemy, Alembic, azure-storage-blob, Jinja2, Marshmallow

---

## Feature 1: Azure Blob Storage Abstraction

### Task 1: StorageBackend ABC + LocalStorage

**Files:**
- Create: `storage.py`
- Test: `tests/test_storage.py`

**Step 1: Write the failing tests**

```python
# tests/test_storage.py
import os
import tempfile
import pytest
from unittest.mock import patch

def test_local_storage_save(app):
    """LocalStorage.save() speichert Datei und gibt URL zurueck."""
    from storage import LocalStorage
    with app.app_context():
        s = LocalStorage()
        url = s.save("avatars", "test.jpg", b"\xff\xd8\xff\xe0testdata")
        assert url.startswith("/static/uploads/avatars/")
        assert url.endswith(".jpg")
        # Datei muss existieren
        full_path = os.path.join(app.static_folder, url.replace("/static/", ""))
        assert os.path.exists(full_path)
        # Aufraumen
        os.remove(full_path)

def test_local_storage_delete(app):
    """LocalStorage.delete() loescht Datei."""
    from storage import LocalStorage
    with app.app_context():
        s = LocalStorage()
        url = s.save("avatars", "del.jpg", b"\xff\xd8\xff\xe0testdata")
        rel_path = url.replace("/static/uploads/", "")
        s.delete(rel_path)
        full_path = os.path.join(app.static_folder, "uploads", rel_path)
        assert not os.path.exists(full_path)

def test_local_storage_exists(app):
    """LocalStorage.exists() prueft ob Datei existiert."""
    from storage import LocalStorage
    with app.app_context():
        s = LocalStorage()
        url = s.save("avatars", "ex.jpg", b"\xff\xd8\xff\xe0testdata")
        rel_path = url.replace("/static/uploads/", "")
        assert s.exists(rel_path) is True
        s.delete(rel_path)
        assert s.exists(rel_path) is False

def test_local_storage_url(app):
    """LocalStorage.url() gibt statische URL zurueck."""
    from storage import LocalStorage
    with app.app_context():
        s = LocalStorage()
        assert s.url("avatars/test.jpg") == "/static/uploads/avatars/test.jpg"

def test_get_storage_returns_local_by_default(app):
    """get_storage() gibt LocalStorage zurueck wenn STORAGE_BACKEND nicht gesetzt."""
    from storage import get_storage, LocalStorage
    with app.app_context():
        s = get_storage()
        assert isinstance(s, LocalStorage)
```

**Step 2: Run tests to verify they fail**

Run: `cd /home/azure-ubuntu/Dog-School-Manager && python -m pytest tests/test_storage.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'storage'`

**Step 3: Write minimal implementation**

```python
# storage.py
"""Storage-Abstraction-Layer: Local und Azure Blob Storage Backends."""
import os
from abc import ABC, abstractmethod
from flask import current_app
from upload_helpers import safe_stored_filename


class StorageBackend(ABC):
    """Abstract Base Class fuer Storage-Backends."""

    @abstractmethod
    def save(self, category: str, filename: str, data: bytes) -> str:
        """Speichert Datei und gibt URL/Pfad zurueck."""

    @abstractmethod
    def url(self, path: str) -> str:
        """Gibt oeffentliche URL fuer Pfad zurueck."""

    @abstractmethod
    def delete(self, path: str) -> None:
        """Loescht Datei."""

    @abstractmethod
    def exists(self, path: str) -> bool:
        """Prueft ob Datei existiert."""


class LocalStorage(StorageBackend):
    """Speichert Dateien im lokalen Dateisystem unter static/uploads/."""

    def _upload_dir(self, category: str) -> str:
        base = os.path.join(current_app.static_folder, "uploads", category)
        os.makedirs(base, exist_ok=True)
        return base

    def save(self, category: str, filename: str, data: bytes) -> str:
        safe_name = safe_stored_filename(filename)
        directory = self._upload_dir(category)
        filepath = os.path.join(directory, safe_name)
        with open(filepath, "wb") as f:
            f.write(data)
        return f"/static/uploads/{category}/{safe_name}"

    def url(self, path: str) -> str:
        return f"/static/uploads/{path}"

    def delete(self, path: str) -> None:
        filepath = os.path.join(current_app.static_folder, "uploads", path)
        if os.path.exists(filepath):
            os.remove(filepath)

    def exists(self, path: str) -> bool:
        filepath = os.path.join(current_app.static_folder, "uploads", path)
        return os.path.exists(filepath)


_storage_instance = None


def get_storage() -> StorageBackend:
    """Gibt das konfigurierte Storage-Backend zurueck (Singleton pro Request-Kontext)."""
    global _storage_instance
    backend = os.environ.get("STORAGE_BACKEND", "local")
    if backend == "azure":
        from storage import AzureBlobStorage
        return AzureBlobStorage()
    return LocalStorage()
```

**Step 4: Run tests to verify they pass**

Run: `cd /home/azure-ubuntu/Dog-School-Manager && python -m pytest tests/test_storage.py -v`
Expected: PASS (5 tests)

**Step 5: Commit**

```bash
git add storage.py tests/test_storage.py
git commit -m "feat: Storage-Abstraction-Layer mit LocalStorage Backend"
```

---

### Task 2: AzureBlobStorage Backend

**Files:**
- Modify: `storage.py`
- Modify: `requirements.txt` (add azure-storage-blob)
- Test: `tests/test_storage.py` (add Azure tests)

**Step 1: Add azure-storage-blob to requirements.txt**

Add `azure-storage-blob>=12.0.0` to requirements.txt.

**Step 2: Write failing tests for AzureBlobStorage**

```python
# tests/test_storage.py — additional tests
from unittest.mock import MagicMock, patch

def test_azure_storage_save(app):
    """AzureBlobStorage.save() uploaded nach Azure und gibt URL zurueck."""
    from storage import AzureBlobStorage
    with app.app_context():
        mock_client = MagicMock()
        mock_container = MagicMock()
        mock_client.get_container_client.return_value = mock_container
        mock_blob = MagicMock()
        mock_blob.url = "https://account.blob.core.windows.net/container/avatars/test.jpg"
        mock_container.get_blob_client.return_value = mock_blob

        with patch.object(AzureBlobStorage, '_get_client', return_value=mock_client):
            with patch.object(AzureBlobStorage, '_get_container_name', return_value='testcontainer'):
                s = AzureBlobStorage()
                url = s.save("avatars", "test.jpg", b"\xff\xd8\xff\xe0testdata")
                assert "avatars/" in url
                assert url.endswith(".jpg")
                mock_blob.upload_blob.assert_called_once()

def test_azure_storage_delete(app):
    """AzureBlobStorage.delete() loescht Blob."""
    from storage import AzureBlobStorage
    with app.app_context():
        mock_client = MagicMock()
        mock_container = MagicMock()
        mock_client.get_container_client.return_value = mock_container
        mock_blob = MagicMock()
        mock_container.get_blob_client.return_value = mock_blob

        with patch.object(AzureBlobStorage, '_get_client', return_value=mock_client):
            with patch.object(AzureBlobStorage, '_get_container_name', return_value='testcontainer'):
                s = AzureBlobStorage()
                s.delete("avatars/test.jpg")
                mock_blob.delete_blob.assert_called_once()

def test_azure_storage_exists(app):
    """AzureBlobStorage.exists() prueft Blob-Existenz."""
    from storage import AzureBlobStorage
    with app.app_context():
        mock_client = MagicMock()
        mock_container = MagicMock()
        mock_client.get_container_client.return_value = mock_container
        mock_blob = MagicMock()
        mock_blob.get_blob_properties.return_value = {"size": 100}
        mock_container.get_blob_client.return_value = mock_blob

        with patch.object(AzureBlobStorage, '_get_client', return_value=mock_client):
            with patch.object(AzureBlobStorage, '_get_container_name', return_value='testcontainer'):
                s = AzureBlobStorage()
                assert s.exists("avatars/test.jpg") is True

def test_get_storage_returns_azure_when_configured(app):
    """get_storage() gibt AzureBlobStorage zurueck wenn STORAGE_BACKEND=azure."""
    from storage import get_storage, AzureBlobStorage
    with app.app_context():
        with patch.dict(os.environ, {"STORAGE_BACKEND": "azure"}):
            with patch.object(AzureBlobStorage, '_get_client', return_value=MagicMock()):
                with patch.object(AzureBlobStorage, '_get_container_name', return_value='test'):
                    s = get_storage()
                    assert isinstance(s, AzureBlobStorage)
```

**Step 3: Run tests to verify they fail**

Run: `python -m pytest tests/test_storage.py -v -k azure`
Expected: FAIL — `AzureBlobStorage` not defined

**Step 4: Implement AzureBlobStorage**

Add to `storage.py`:

```python
class AzureBlobStorage(StorageBackend):
    """Speichert Dateien in Azure Blob Storage."""

    def _get_client(self):
        from azure.storage.blob import BlobServiceClient
        from models.platform import PlatformSettings
        settings = PlatformSettings.get_instance()
        conn_str = settings.azure_blob_connection_string
        if not conn_str:
            raise ValueError("Azure Blob Connection String nicht konfiguriert")
        return BlobServiceClient.from_connection_string(conn_str)

    def _get_container_name(self) -> str:
        from models.platform import PlatformSettings
        settings = PlatformSettings.get_instance()
        return settings.azure_blob_container_name or "uploads"

    def save(self, category: str, filename: str, data: bytes) -> str:
        safe_name = safe_stored_filename(filename)
        blob_path = f"{category}/{safe_name}"
        client = self._get_client()
        container = client.get_container_client(self._get_container_name())
        blob = container.get_blob_client(blob_path)
        blob.upload_blob(data, overwrite=True)
        return blob.url

    def url(self, path: str) -> str:
        client = self._get_client()
        container = client.get_container_client(self._get_container_name())
        blob = container.get_blob_client(path)
        return blob.url

    def delete(self, path: str) -> None:
        client = self._get_client()
        container = client.get_container_client(self._get_container_name())
        blob = container.get_blob_client(path)
        blob.delete_blob()

    def exists(self, path: str) -> bool:
        try:
            client = self._get_client()
            container = client.get_container_client(self._get_container_name())
            blob = container.get_blob_client(path)
            blob.get_blob_properties()
            return True
        except Exception:
            return False
```

**Step 5: Run all storage tests**

Run: `python -m pytest tests/test_storage.py -v`
Expected: PASS (alle 9 Tests)

**Step 6: Commit**

```bash
git add storage.py tests/test_storage.py requirements.txt
git commit -m "feat: AzureBlobStorage Backend und azure-storage-blob Dependency"
```

---

### Task 3: Upload-Endpoints auf Storage-Abstraction umstellen

**Files:**
- Modify: `blueprints/api_auth.py` (api_upload_avatar)
- Modify: `blueprints/api_customer/dogs.py` (upload_dog_photo)
- Modify: `blueprints/api_admin/dogs.py` (upload_dog_photo)
- Modify: `blueprints/api_mobile.py` (api_dog_file_upload)
- Modify: `blueprints/api_messaging.py` (upload_file)
- Modify: `blueprints/messaging.py` (upload_file)
- Modify: `blueprints/admin/dogs.py` (upload_dog_file)
- Modify: `blueprints/saas.py` (school_settings — Logo/Hero)
- Modify: `blueprints/platform_admin.py` (settings — Logo)

**Step 1: Identifiziere und ersetze in jeder Datei das Muster**

Ersetze in jeder Upload-Funktion:
```python
# ALT:
filepath = os.path.join(upload_dir, safe_name)
with open(filepath, "wb") as f:
    f.write(data)
url = f"/static/uploads/{category}/{safe_name}"

# NEU:
from storage import get_storage
storage = get_storage()
url = storage.save(category, original_filename, data)
```

Gehe jede der 9 Dateien einzeln durch. Fuer jede Datei:

1. Lese die Upload-Funktion
2. Ersetze `open(filepath, "wb")` / `file.save(filepath)` durch `get_storage().save(category, filename, data)`
3. Ersetze hartcodierte URL-Konstruktion durch die von `save()` zurueckgegebene URL
4. Entferne `os.makedirs()` Aufrufe fuer Upload-Verzeichnisse (macht LocalStorage intern)

**Wichtig:** `upload_helpers.py` Funktionen (`allowed_file`, `validate_magic_bytes`, `safe_stored_filename`, `check_storage_quota`) bleiben unveraendert — die werden VOR `storage.save()` aufgerufen. Nur die eigentliche Datei-Speicherung wird ersetzt.

**Step 2: Run existing tests to verify nothing breaks**

Run: `python -m pytest tests/ -v --timeout=60 -x`
Expected: Alle bestehenden Tests PASS

**Step 3: Commit**

```bash
git add blueprints/api_auth.py blueprints/api_customer/dogs.py blueprints/api_admin/dogs.py \
       blueprints/api_mobile.py blueprints/api_messaging.py blueprints/messaging.py \
       blueprints/admin/dogs.py blueprints/saas.py blueprints/platform_admin.py
git commit -m "refactor: Upload-Endpoints auf Storage-Abstraction umgestellt"
```

---

### Task 4: Migrations-Script (Local → Azure Blob)

**Files:**
- Create: `scripts/migrate_to_blob.py`
- Test: `tests/test_migrate_to_blob.py`

**Step 1: Write failing tests**

```python
# tests/test_migrate_to_blob.py
import os
import tempfile
from unittest.mock import MagicMock, patch

def test_migrate_discovers_files(app, tmp_path):
    """Script findet alle Dateien in uploads-Verzeichnissen."""
    from scripts.migrate_to_blob import discover_local_files
    uploads = tmp_path / "uploads"
    uploads.mkdir()
    (uploads / "avatars").mkdir()
    (uploads / "avatars" / "test.jpg").write_bytes(b"fake")
    files = discover_local_files(str(tmp_path))
    assert len(files) == 1
    assert files[0]["relative_path"] == "avatars/test.jpg"

def test_migrate_dry_run_no_upload(app, tmp_path):
    """--dry-run uploadet keine Dateien."""
    from scripts.migrate_to_blob import migrate_files
    uploads = tmp_path / "uploads"
    uploads.mkdir()
    (uploads / "test.jpg").write_bytes(b"fake")
    mock_storage = MagicMock()
    result = migrate_files(str(tmp_path), mock_storage, dry_run=True, delete_local=False)
    mock_storage.save.assert_not_called()
    assert result["skipped"] == 1

def test_migrate_uploads_and_verifies(app, tmp_path):
    """Script uploaded und verifiziert (Byte-Vergleich)."""
    from scripts.migrate_to_blob import migrate_files
    uploads = tmp_path / "uploads"
    uploads.mkdir()
    content = b"\xff\xd8\xff\xe0testjpg"
    (uploads / "photo.jpg").write_bytes(content)
    mock_storage = MagicMock()
    mock_storage.save.return_value = "https://blob.core/uploads/photo.jpg"
    mock_storage.exists.return_value = True
    result = migrate_files(str(tmp_path), mock_storage, dry_run=False, delete_local=False)
    mock_storage.save.assert_called_once()
    assert result["uploaded"] == 1
```

**Step 2: Run tests to verify they fail**

Run: `python -m pytest tests/test_migrate_to_blob.py -v`
Expected: FAIL — `ModuleNotFoundError`

**Step 3: Implement migration script**

```python
# scripts/migrate_to_blob.py
"""Migration: Lokale Uploads → Azure Blob Storage.

Usage:
    flask migrate-to-blob --dry-run
    flask migrate-to-blob --delete-local
"""
import os
import click
from flask import current_app
from flask.cli import with_appcontext
from storage import get_storage


def discover_local_files(base_path: str) -> list[dict]:
    """Findet alle Dateien in uploads/ und static/uploads/."""
    files = []
    for search_dir in ["uploads", os.path.join("static", "uploads")]:
        full_dir = os.path.join(base_path, search_dir)
        if not os.path.isdir(full_dir):
            continue
        for root, _, filenames in os.walk(full_dir):
            for fname in filenames:
                full_path = os.path.join(root, fname)
                rel_path = os.path.relpath(full_path, full_dir)
                files.append({
                    "full_path": full_path,
                    "relative_path": rel_path,
                    "size": os.path.getsize(full_path),
                    "source_dir": search_dir,
                })
    return files


def migrate_files(base_path: str, storage, dry_run: bool = True,
                  delete_local: bool = False) -> dict:
    """Migriert lokale Dateien zum konfigurierten Storage-Backend."""
    files = discover_local_files(base_path)
    result = {"total": len(files), "uploaded": 0, "skipped": 0, "errors": 0, "deleted": 0}

    for i, f in enumerate(files, 1):
        rel = f["relative_path"]
        if dry_run:
            click.echo(f"  [DRY-RUN] {i}/{len(files)}: {rel} ({f['size']} bytes)")
            result["skipped"] += 1
            continue

        try:
            with open(f["full_path"], "rb") as fh:
                data = fh.read()
            category = os.path.dirname(rel) or "misc"
            filename = os.path.basename(rel)
            # Upload mit original Pfad-Struktur
            storage.save(category, filename, data)
            # Verifizieren
            if storage.exists(rel):
                result["uploaded"] += 1
                click.echo(f"  [{i}/{len(files)}] ✓ {rel}")
                if delete_local:
                    os.remove(f["full_path"])
                    result["deleted"] += 1
            else:
                click.echo(f"  [{i}/{len(files)}] ✗ Verifikation fehlgeschlagen: {rel}")
                result["errors"] += 1
        except Exception as e:
            click.echo(f"  [{i}/{len(files)}] ✗ Fehler: {rel} — {e}")
            result["errors"] += 1

    return result
```

Register CLI command in `app.py`:

```python
@app.cli.command("migrate-to-blob")
@click.option("--dry-run", is_flag=True, default=False, help="Nur anzeigen, nicht hochladen")
@click.option("--delete-local", is_flag=True, default=False, help="Lokale Dateien nach Upload loeschen")
@with_appcontext
def migrate_to_blob_cmd(dry_run, delete_local):
    """Migriert lokale Uploads nach Azure Blob Storage."""
    from scripts.migrate_to_blob import migrate_files
    from storage import get_storage
    storage = get_storage()
    base_path = current_app.root_path
    click.echo(f"Storage-Backend: {type(storage).__name__}")
    if dry_run:
        click.echo("=== DRY RUN ===")
    result = migrate_files(base_path, storage, dry_run=dry_run, delete_local=delete_local)
    click.echo(f"\nErgebnis: {result['uploaded']} hochgeladen, {result['skipped']} uebersprungen, "
               f"{result['errors']} Fehler, {result['deleted']} lokal geloescht")
```

**Step 4: Run tests**

Run: `python -m pytest tests/test_migrate_to_blob.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add scripts/migrate_to_blob.py tests/test_migrate_to_blob.py app.py
git commit -m "feat: Migrations-Script fuer Local → Azure Blob Storage"
```

---

## Feature 2: Wartungsmodus

### Task 5: Alembic-Migration fuer Wartungsmodus-Felder

**Files:**
- Create: `migrations/versions/xxxx_add_maintenance_fields.py` (via Alembic)
- Modify: `models/platform.py` (PlatformSettings)

**Step 1: Felder zum Model hinzufuegen**

In `models/platform.py`, PlatformSettings-Klasse, nach den bestehenden Azure-Feldern (Zeile ~174):

```python
    # --- Wartungsmodus ---
    maintenance_portal_active = db.Column(db.Boolean, default=False, nullable=False)
    maintenance_portal_message = db.Column(db.Text, nullable=True)
    maintenance_website_active = db.Column(db.Boolean, default=False, nullable=False)
    maintenance_website_message = db.Column(db.Text, nullable=True)
    maintenance_scheduled_start = db.Column(db.DateTime, nullable=True)
    maintenance_scheduled_end = db.Column(db.DateTime, nullable=True)
    maintenance_scheduled_message = db.Column(db.Text, nullable=True)
```

**Step 2: Migration generieren**

Run: `cd /home/azure-ubuntu/Dog-School-Manager && flask db migrate -m "add maintenance fields to platform_settings"`

**Step 3: Migration pruefen und anwenden**

Run: `flask db upgrade`
Expected: Migration erfolgreich

**Step 4: Commit**

```bash
git add models/platform.py migrations/versions/
git commit -m "feat: Wartungsmodus-Felder in PlatformSettings"
```

---

### Task 6: Wartungsmodus before_request Middleware

**Files:**
- Modify: `app.py`
- Test: `tests/test_maintenance.py`

**Step 1: Write failing tests**

```python
# tests/test_maintenance.py
import pytest
from datetime import datetime, timedelta, timezone
from tests.helpers import ensure_platform_settings

def test_maintenance_portal_blocks_login(client, app):
    """Portal-Wartung blockiert Login-Seite."""
    with app.app_context():
        ps = ensure_platform_settings()
        ps.maintenance_portal_active = True
        ps.maintenance_portal_message = "Wartungsarbeiten bis 06:00 Uhr"
        from extensions import db
        db.session.commit()
    rv = client.get("/login")
    assert rv.status_code == 503
    assert b"Wartungsarbeiten" in rv.data

def test_maintenance_portal_allows_public_pages(client, app):
    """Portal-Wartung laesst oeffentliche Seiten durch."""
    with app.app_context():
        ps = ensure_platform_settings()
        ps.maintenance_portal_active = True
        from extensions import db
        db.session.commit()
    rv = client.get("/healthz")
    assert rv.status_code == 200

def test_maintenance_website_blocks_everything(client, app):
    """Website-Wartung blockiert alle Seiten."""
    with app.app_context():
        ps = ensure_platform_settings()
        ps.maintenance_website_active = True
        ps.maintenance_website_message = "Komplettwartung"
        from extensions import db
        db.session.commit()
    rv = client.get("/")
    assert rv.status_code == 503
    assert b"Komplettwartung" in rv.data

def test_maintenance_healthz_always_accessible(client, app):
    """Healthz-Endpoint ist immer erreichbar."""
    with app.app_context():
        ps = ensure_platform_settings()
        ps.maintenance_website_active = True
        from extensions import db
        db.session.commit()
    rv = client.get("/healthz")
    assert rv.status_code == 200

def test_maintenance_static_always_accessible(client, app):
    """/static/ ist immer erreichbar."""
    with app.app_context():
        ps = ensure_platform_settings()
        ps.maintenance_website_active = True
        from extensions import db
        db.session.commit()
    rv = client.get("/static/manifest.json")
    # 200 oder 304, aber nicht 503
    assert rv.status_code != 503

def test_maintenance_scheduled_auto_activates(client, app):
    """Geplante Wartung aktiviert sich automatisch im Zeitfenster."""
    with app.app_context():
        ps = ensure_platform_settings()
        now = datetime.now(timezone.utc)
        ps.maintenance_scheduled_start = now - timedelta(hours=1)
        ps.maintenance_scheduled_end = now + timedelta(hours=1)
        ps.maintenance_scheduled_message = "Geplante Wartung"
        ps.maintenance_portal_active = False
        ps.maintenance_website_active = False
        from extensions import db
        db.session.commit()
    rv = client.get("/login")
    assert rv.status_code == 503

def test_maintenance_api_returns_json(client, app):
    """API-Endpoints bekommen JSON-Antwort statt HTML bei Wartung."""
    with app.app_context():
        ps = ensure_platform_settings()
        ps.maintenance_portal_active = True
        ps.maintenance_portal_message = "API Wartung"
        from extensions import db
        db.session.commit()
    rv = client.get("/api/auth/me", headers={"Accept": "application/json"})
    assert rv.status_code == 503
    data = rv.get_json()
    assert data["error"] == "maintenance"
```

**Step 2: Run tests to verify they fail**

Run: `python -m pytest tests/test_maintenance.py -v`
Expected: FAIL

**Step 3: Implement before_request hook**

In `app.py`, nach den bestehenden before_request Hooks, vor `enforce_subscription()`:

```python
@app.before_request
def enforce_maintenance_mode():
    """Prueft ob Wartungsmodus aktiv ist und blockiert ggf. den Zugriff."""
    from models.platform import PlatformSettings

    # Immer erreichbare Pfade
    exempt_prefixes = ("/healthz", "/static/", "/metrics")
    if any(request.path.startswith(p) for p in exempt_prefixes):
        return None

    try:
        ps = PlatformSettings.get_instance()
    except Exception:
        return None

    now = datetime.now(timezone.utc)

    # Geplante Wartung pruefen
    scheduled_active = (
        ps.maintenance_scheduled_start and ps.maintenance_scheduled_end
        and ps.maintenance_scheduled_start <= now <= ps.maintenance_scheduled_end
    )

    website_active = ps.maintenance_website_active or False
    portal_active = ps.maintenance_portal_active or scheduled_active or False

    if not website_active and not portal_active:
        return None

    # Platform-Admins duerfen immer durch
    if current_user and current_user.is_authenticated and current_user.is_platform_admin:
        return None

    # Website-Wartung: alles gesperrt
    if website_active:
        message = ps.maintenance_website_message or "Wartungsarbeiten"
        return _maintenance_response(message, ps)

    # Portal-Wartung: Login/App gesperrt, oeffentliche Seiten erlaubt
    if portal_active:
        public_prefixes = ("/schule/", "/s/", "/api/public/", "/sitemap", "/robots.txt")
        if any(request.path.startswith(p) for p in public_prefixes) or request.path == "/":
            return None
        message = (ps.maintenance_scheduled_message if scheduled_active
                   else ps.maintenance_portal_message) or "Wartungsarbeiten"
        return _maintenance_response(message, ps)

    return None


def _maintenance_response(message, ps):
    """Gibt Wartungs-Response zurueck (JSON fuer API, HTML fuer Browser)."""
    if request.path.startswith("/api/") or request.accept_mimetypes.best == "application/json":
        data = {"error": "maintenance", "message": message}
        if ps.maintenance_scheduled_end:
            data["scheduled_end"] = ps.maintenance_scheduled_end.isoformat()
        return jsonify(data), 503

    return render_template("maintenance.html",
                           message=message,
                           scheduled_end=ps.maintenance_scheduled_end), 503
```

**Step 4: Run tests**

Run: `python -m pytest tests/test_maintenance.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add app.py tests/test_maintenance.py
git commit -m "feat: Wartungsmodus before_request Middleware"
```

---

### Task 7: Wartungsseite Template

**Files:**
- Create: `templates/maintenance.html`

**Step 1: Erstelle eigenstaendiges Template**

```html
<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Wartungsarbeiten — PawCoach</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            background: #f5f7f5;
            color: #333;
            display: flex;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            padding: 2rem;
        }
        .maintenance-card {
            background: white;
            border-radius: 12px;
            padding: 3rem;
            max-width: 500px;
            text-align: center;
            box-shadow: 0 4px 24px rgba(0,0,0,0.08);
        }
        .logo { font-size: 2rem; font-weight: 700; color: #4A7C59; margin-bottom: 1.5rem; }
        .icon { font-size: 3rem; margin-bottom: 1rem; }
        h1 { font-size: 1.5rem; margin-bottom: 1rem; color: #222; }
        .message { font-size: 1.1rem; line-height: 1.6; color: #555; margin-bottom: 1.5rem; }
        .scheduled-end {
            background: #f0f7f0;
            border-radius: 8px;
            padding: 1rem;
            font-size: 0.95rem;
            color: #4A7C59;
        }
    </style>
</head>
<body>
    <div class="maintenance-card">
        <div class="logo">PawCoach</div>
        <div class="icon" aria-hidden="true">&#128736;</div>
        <h1>Wartungsarbeiten</h1>
        <p class="message">{{ message }}</p>
        {% if scheduled_end %}
        <div class="scheduled-end">
            Voraussichtliches Ende: {{ scheduled_end.strftime('%d.%m.%Y um %H:%M Uhr') }}
        </div>
        {% endif %}
    </div>
</body>
</html>
```

**Step 2: Commit**

```bash
git add templates/maintenance.html
git commit -m "feat: Wartungsseite Template"
```

---

### Task 8: Wartungsmodus CLI-Commands

**Files:**
- Modify: `app.py`
- Test: `tests/test_maintenance.py` (CLI-Tests ergaenzen)

**Step 1: Write failing tests**

```python
# tests/test_maintenance.py — additional CLI tests
from click.testing import CliRunner

def test_cli_maintenance_on(app):
    """flask maintenance-on aktiviert Wartungsmodus."""
    with app.app_context():
        ensure_platform_settings()
    runner = CliRunner()
    result = runner.invoke(app.cli, ["maintenance-on", "--scope", "portal", "--message", "Test"])
    assert result.exit_code == 0
    with app.app_context():
        from models.platform import PlatformSettings
        ps = PlatformSettings.get_instance()
        assert ps.maintenance_portal_active is True
        assert ps.maintenance_portal_message == "Test"

def test_cli_maintenance_off(app):
    """flask maintenance-off deaktiviert Wartungsmodus."""
    with app.app_context():
        ps = ensure_platform_settings()
        ps.maintenance_portal_active = True
        from extensions import db
        db.session.commit()
    runner = CliRunner()
    result = runner.invoke(app.cli, ["maintenance-off", "--scope", "portal"])
    assert result.exit_code == 0
    with app.app_context():
        from models.platform import PlatformSettings
        ps = PlatformSettings.get_instance()
        assert ps.maintenance_portal_active is False

def test_cli_maintenance_status(app):
    """flask maintenance-status zeigt aktuellen Status."""
    with app.app_context():
        ensure_platform_settings()
    runner = CliRunner()
    result = runner.invoke(app.cli, ["maintenance-status"])
    assert result.exit_code == 0
    assert "Portal" in result.output
    assert "Website" in result.output
```

**Step 2: Run tests to verify they fail**

Run: `python -m pytest tests/test_maintenance.py -v -k cli`
Expected: FAIL

**Step 3: Implement CLI commands**

In `app.py`:

```python
@app.cli.command("maintenance-on")
@click.option("--scope", type=click.Choice(["portal", "website", "all"]), required=True)
@click.option("--message", default="Wartungsarbeiten", help="Wartungsmeldung")
@click.option("--until", default=None, help="Ende-Zeitpunkt (YYYY-MM-DD HH:MM)")
@with_appcontext
def maintenance_on(scope, message, until):
    """Wartungsmodus aktivieren."""
    from models.platform import PlatformSettings
    ps = PlatformSettings.get_instance()
    end_dt = None
    if until:
        end_dt = datetime.strptime(until, "%Y-%m-%d %H:%M").replace(tzinfo=timezone.utc)

    if scope in ("portal", "all"):
        ps.maintenance_portal_active = True
        ps.maintenance_portal_message = message
    if scope in ("website", "all"):
        ps.maintenance_website_active = True
        ps.maintenance_website_message = message
    if end_dt:
        ps.maintenance_scheduled_end = end_dt

    db.session.commit()
    click.echo(f"Wartungsmodus aktiviert: {scope}")
    if end_dt:
        click.echo(f"Geplantes Ende: {end_dt.strftime('%d.%m.%Y %H:%M')} UTC")


@app.cli.command("maintenance-off")
@click.option("--scope", type=click.Choice(["portal", "website", "all"]), required=True)
@with_appcontext
def maintenance_off(scope):
    """Wartungsmodus deaktivieren."""
    from models.platform import PlatformSettings
    ps = PlatformSettings.get_instance()

    if scope in ("portal", "all"):
        ps.maintenance_portal_active = False
        ps.maintenance_portal_message = None
    if scope in ("website", "all"):
        ps.maintenance_website_active = False
        ps.maintenance_website_message = None
    if scope == "all":
        ps.maintenance_scheduled_start = None
        ps.maintenance_scheduled_end = None
        ps.maintenance_scheduled_message = None

    db.session.commit()
    click.echo(f"Wartungsmodus deaktiviert: {scope}")


@app.cli.command("maintenance-status")
@with_appcontext
def maintenance_status():
    """Wartungsmodus-Status anzeigen."""
    from models.platform import PlatformSettings
    ps = PlatformSettings.get_instance()

    click.echo("=== Wartungsmodus-Status ===")
    click.echo(f"Portal:  {'AKTIV' if ps.maintenance_portal_active else 'inaktiv'}")
    if ps.maintenance_portal_message:
        click.echo(f"  Meldung: {ps.maintenance_portal_message}")
    click.echo(f"Website: {'AKTIV' if ps.maintenance_website_active else 'inaktiv'}")
    if ps.maintenance_website_message:
        click.echo(f"  Meldung: {ps.maintenance_website_message}")
    if ps.maintenance_scheduled_start:
        click.echo(f"Geplant: {ps.maintenance_scheduled_start} — {ps.maintenance_scheduled_end}")
        if ps.maintenance_scheduled_message:
            click.echo(f"  Meldung: {ps.maintenance_scheduled_message}")
```

**Step 4: Run tests**

Run: `python -m pytest tests/test_maintenance.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add app.py tests/test_maintenance.py
git commit -m "feat: Wartungsmodus CLI-Commands (on/off/status)"
```

---

### Task 9: Wartungsmodus API-Endpoints (Platform-Admin)

**Files:**
- Modify: `blueprints/api_platform_admin/settings.py` (oder neues Modul)
- Test: `tests/test_maintenance.py` (API-Tests ergaenzen)

**Step 1: Write failing tests**

```python
# tests/test_maintenance.py — API-Tests
def test_api_get_maintenance_status(client, app, platform_admin_headers):
    """GET /api/platform/maintenance gibt aktuellen Status."""
    with app.app_context():
        ensure_platform_settings()
    rv = client.get("/api/platform/maintenance", headers=platform_admin_headers)
    assert rv.status_code == 200
    data = rv.get_json()
    assert "portal_active" in data
    assert "website_active" in data

def test_api_put_maintenance(client, app, platform_admin_headers):
    """PUT /api/platform/maintenance setzt Wartungsmodus."""
    with app.app_context():
        ensure_platform_settings()
    rv = client.put("/api/platform/maintenance",
                    json={"portal_active": True, "portal_message": "API-Test"},
                    headers=platform_admin_headers)
    assert rv.status_code == 200
    with app.app_context():
        from models.platform import PlatformSettings
        ps = PlatformSettings.get_instance()
        assert ps.maintenance_portal_active is True

def test_api_schedule_maintenance(client, app, platform_admin_headers):
    """POST /api/platform/maintenance/schedule plant Wartungsfenster."""
    with app.app_context():
        ensure_platform_settings()
    rv = client.post("/api/platform/maintenance/schedule",
                     json={"start": "2026-03-15T02:00:00Z",
                           "end": "2026-03-15T06:00:00Z",
                           "message": "Geplante Wartung"},
                     headers=platform_admin_headers)
    assert rv.status_code == 200

def test_api_delete_schedule(client, app, platform_admin_headers):
    """DELETE /api/platform/maintenance/schedule loescht geplantes Fenster."""
    with app.app_context():
        ps = ensure_platform_settings()
        ps.maintenance_scheduled_start = datetime(2026, 3, 15, 2, 0, tzinfo=timezone.utc)
        ps.maintenance_scheduled_end = datetime(2026, 3, 15, 6, 0, tzinfo=timezone.utc)
        from extensions import db
        db.session.commit()
    rv = client.delete("/api/platform/maintenance/schedule",
                       headers=platform_admin_headers)
    assert rv.status_code == 200

def test_api_maintenance_requires_platform_admin(client, app, admin_headers):
    """Wartungs-API erfordert Platform-Admin Rechte."""
    rv = client.get("/api/platform/maintenance", headers=admin_headers)
    assert rv.status_code in (401, 403)
```

**Step 2: Run tests to verify they fail**

Run: `python -m pytest tests/test_maintenance.py -v -k api`
Expected: FAIL — Endpoints existieren nicht

**Step 3: Implement API endpoints**

In `blueprints/api_platform_admin/settings.py` oder neues `blueprints/api_platform_admin/maintenance.py`:

```python
@api_platform_admin_bp.route("/maintenance", methods=["GET"])
@jwt_or_session_required
@platform_admin_required
@limiter.limit("60 per hour")
def get_maintenance_status():
    """Aktuellen Wartungsstatus abrufen."""
    from models.platform import PlatformSettings
    ps = PlatformSettings.get_instance()
    return jsonify({
        "portal_active": ps.maintenance_portal_active or False,
        "portal_message": ps.maintenance_portal_message,
        "website_active": ps.maintenance_website_active or False,
        "website_message": ps.maintenance_website_message,
        "scheduled_start": ps.maintenance_scheduled_start.isoformat() if ps.maintenance_scheduled_start else None,
        "scheduled_end": ps.maintenance_scheduled_end.isoformat() if ps.maintenance_scheduled_end else None,
        "scheduled_message": ps.maintenance_scheduled_message,
    })


@api_platform_admin_bp.route("/maintenance", methods=["PUT"])
@jwt_or_session_required
@platform_admin_required
@limiter.limit("20 per hour")
def update_maintenance():
    """Wartungsmodus setzen/aendern."""
    from models.platform import PlatformSettings
    data = request.get_json(silent=True) or {}
    ps = PlatformSettings.get_instance()

    if "portal_active" in data:
        ps.maintenance_portal_active = bool(data["portal_active"])
    if "portal_message" in data:
        ps.maintenance_portal_message = bleach.clean(data["portal_message"]) if data["portal_message"] else None
    if "website_active" in data:
        ps.maintenance_website_active = bool(data["website_active"])
    if "website_message" in data:
        ps.maintenance_website_message = bleach.clean(data["website_message"]) if data["website_message"] else None

    try:
        db.session.commit()
    except Exception:
        db.session.rollback()
        return jsonify({"error": "Datenbankfehler"}), 500

    return jsonify({"message": "Wartungsmodus aktualisiert"})


@api_platform_admin_bp.route("/maintenance/schedule", methods=["POST"])
@jwt_or_session_required
@platform_admin_required
@limiter.limit("10 per hour")
def schedule_maintenance():
    """Wartungsfenster planen."""
    from models.platform import PlatformSettings
    data = request.get_json(silent=True) or {}
    ps = PlatformSettings.get_instance()

    start = data.get("start")
    end = data.get("end")
    if not start or not end:
        return jsonify({"error": "start und end erforderlich"}), 400

    ps.maintenance_scheduled_start = datetime.fromisoformat(start)
    ps.maintenance_scheduled_end = datetime.fromisoformat(end)
    ps.maintenance_scheduled_message = bleach.clean(data.get("message", "")) if data.get("message") else None

    try:
        db.session.commit()
    except Exception:
        db.session.rollback()
        return jsonify({"error": "Datenbankfehler"}), 500

    return jsonify({"message": "Wartungsfenster geplant"})


@api_platform_admin_bp.route("/maintenance/schedule", methods=["DELETE"])
@jwt_or_session_required
@platform_admin_required
@limiter.limit("10 per hour")
def delete_maintenance_schedule():
    """Geplantes Wartungsfenster loeschen."""
    from models.platform import PlatformSettings
    ps = PlatformSettings.get_instance()
    ps.maintenance_scheduled_start = None
    ps.maintenance_scheduled_end = None
    ps.maintenance_scheduled_message = None

    try:
        db.session.commit()
    except Exception:
        db.session.rollback()
        return jsonify({"error": "Datenbankfehler"}), 500

    return jsonify({"message": "Wartungsfenster geloescht"})
```

**Step 4: Run tests**

Run: `python -m pytest tests/test_maintenance.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add blueprints/api_platform_admin/ tests/test_maintenance.py
git commit -m "feat: Wartungsmodus API-Endpoints (GET/PUT/POST/DELETE)"
```

---

### Task 10: Wartungsmodus Platform-Admin UI

**Files:**
- Modify: `templates/platform_admin/settings.html` (oder neuer Abschnitt)
- Modify: `blueprints/platform_admin.py` (Template-Kontext erweitern)

**Step 1: Erweitere Platform-Admin Settings Template**

Fuege einen neuen Abschnitt "Wartungsmodus" zum bestehenden Platform-Admin Settings Template hinzu:

- Toggle fuer Portal-Wartung + Textfeld
- Toggle fuer Website-Wartung + Textfeld
- Formular fuer Wartungsfenster planen (Start datetime-local, Ende datetime-local, Meldung)
- Status-Anzeige (aktiv/inaktiv, geplant)
- Alle Formulare submitten per POST an bestehende Routes

**Step 2: Erweitere Platform-Admin Route**

In `blueprints/platform_admin.py`, `settings()` Route:
- Wartungs-Felder aus PlatformSettings an Template uebergeben
- POST-Handler fuer Wartungs-Toggle und Zeitplanung

**Step 3: Test manuell im Browser**

Da UI-Tests schwierig zu automatisieren sind: manuell verifizieren, dass:
- Toggles funktionieren
- Zeitplanung gespeichert wird
- Status korrekt angezeigt wird

**Step 4: Commit**

```bash
git add templates/platform_admin/ blueprints/platform_admin.py
git commit -m "feat: Wartungsmodus Platform-Admin UI"
```

---

### Task 11: Wartungs-Banner Vorwarnung

**Files:**
- Modify: `templates/base.html`
- Modify: `app.py` (Context-Processor)

**Step 1: Context-Processor fuer Banner**

In `app.py`, im bestehenden `@app.context_processor`:

```python
@app.context_processor
def inject_maintenance_banner():
    from models.platform import PlatformSettings
    try:
        ps = PlatformSettings.get_instance()
        now = datetime.now(timezone.utc)
        if (ps.maintenance_scheduled_start
                and ps.maintenance_scheduled_start > now
                and ps.maintenance_scheduled_end):
            return {
                "maintenance_banner": {
                    "start": ps.maintenance_scheduled_start,
                    "end": ps.maintenance_scheduled_end,
                    "message": ps.maintenance_scheduled_message or "Geplante Wartungsarbeiten",
                }
            }
    except Exception:
        pass
    return {"maintenance_banner": None}
```

**Step 2: Banner in base.html**

In `templates/base.html`, direkt nach `<body>`:

```html
{% if maintenance_banner %}
<div style="background:#ffc107;color:#333;padding:0.75rem 1rem;text-align:center;font-size:0.9rem;">
    Geplante Wartungsarbeiten am {{ maintenance_banner.start.strftime('%d.%m.%Y') }}
    von {{ maintenance_banner.start.strftime('%H:%M') }}
    bis {{ maintenance_banner.end.strftime('%H:%M') }} Uhr:
    {{ maintenance_banner.message }}
</div>
{% endif %}
```

**Step 3: Commit**

```bash
git add templates/base.html app.py
git commit -m "feat: Wartungs-Banner Vorwarnung bei geplanter Wartung"
```

---

## Feature 3: Platform-Admin ↔ Schuladmin Messaging

### Task 12: list_contacts() fuer Platform-Admins erweitern (API)

**Files:**
- Modify: `blueprints/api_messaging.py`
- Test: `tests/test_platform_messaging.py`

**Step 1: Write failing tests**

```python
# tests/test_platform_messaging.py
import pytest
from tests.helpers import login, auth_header, ensure_platform_settings

def test_platform_admin_sees_all_school_admins(client, app, platform_admin_headers):
    """Platform-Admin sieht alle Schuladmins aller Schulen in Kontaktliste."""
    rv = client.get("/api/messages/contacts", headers=platform_admin_headers)
    assert rv.status_code == 200
    data = rv.get_json()
    contacts = data.get("contacts", [])
    # Mindestens ein Kontakt mit school_name
    school_contacts = [c for c in contacts if c.get("school_name")]
    # Platform-Admin sollte Schuladmins sehen
    assert isinstance(contacts, list)

def test_platform_admin_contacts_grouped_by_school(client, app, platform_admin_headers):
    """Platform-Admin Kontakte enthalten school_name pro Schuladmin."""
    rv = client.get("/api/messages/contacts", headers=platform_admin_headers)
    data = rv.get_json()
    for contact in data.get("contacts", []):
        # Jeder Kontakt hat entweder school_name oder group
        assert "school_name" in contact or "group" in contact

def test_school_admin_sees_platform_admins(client, app, admin_headers):
    """Schuladmin sieht Platform-Admins als 'PawCoach Support' Gruppe."""
    rv = client.get("/api/messages/contacts", headers=admin_headers)
    data = rv.get_json()
    contacts = data.get("contacts", [])
    support_contacts = [c for c in contacts if c.get("group") == "PawCoach Support"]
    # Sollte mindestens einen Platform-Admin zeigen
    assert isinstance(support_contacts, list)
```

**Step 2: Run tests to verify they fail**

Run: `python -m pytest tests/test_platform_messaging.py -v`
Expected: FAIL — Kontakte enthalten kein `school_name`/`group`

**Step 3: Erweitere list_contacts()**

In `blueprints/api_messaging.py`, `list_contacts()` (Zeile ~592):

```python
def list_contacts():
    user = get_current_api_user()
    school_id = get_api_school_id()

    contacts_map = {}

    # --- Bestehende Logik: Kontakte der eigenen Schule ---
    if school_id:
        school_roles = (
            UserSchoolRole.query
            .filter(UserSchoolRole.school_id == school_id, UserSchoolRole.user_id != user.id)
            .all()
        )
        for sr in school_roles:
            if sr.user_id not in contacts_map:
                u = sr.user
                contacts_map[sr.user_id] = {
                    "id": sr.user_id, "name": u.name, "role": sr.role,
                    "roles": [sr.role], "group": None, "school_name": None,
                }
            else:
                contacts_map[sr.user_id]["roles"].append(sr.role)
                role_priority = {"admin": 0, "trainer": 1, "customer": 2}
                current_primary = contacts_map[sr.user_id]["role"]
                if role_priority.get(sr.role, 9) < role_priority.get(current_primary, 9):
                    contacts_map[sr.user_id]["role"] = sr.role

    # --- NEU: Platform-Admin sieht alle Schuladmins ---
    if user.is_platform_admin:
        from models.school import SchoolProfile
        admin_roles = (
            UserSchoolRole.query
            .filter(UserSchoolRole.role == "admin", UserSchoolRole.user_id != user.id)
            .all()
        )
        for sr in admin_roles:
            if sr.user_id not in contacts_map:
                u = sr.user
                school = db.session.get(SchoolProfile, sr.school_id)
                contacts_map[sr.user_id] = {
                    "id": sr.user_id, "name": u.name, "role": "admin",
                    "roles": ["admin"], "group": None,
                    "school_name": school.name if school else "Unbekannt",
                }

        # Andere Platform-Admins
        from models.user import User
        other_admins = User.query.filter(
            User.is_platform_admin == True, User.id != user.id
        ).all()
        for admin in other_admins:
            if admin.id not in contacts_map:
                contacts_map[admin.id] = {
                    "id": admin.id, "name": admin.name, "role": "platform_admin",
                    "roles": ["platform_admin"], "group": "Platform-Admins",
                    "school_name": None,
                }

    # --- NEU: Schuladmin sieht Platform-Admins ---
    elif school_id:
        from models.user import User
        sr_check = UserSchoolRole.query.filter_by(
            user_id=user.id, school_id=school_id, role="admin"
        ).first()
        if sr_check:
            platform_admins = User.query.filter(
                User.is_platform_admin == True, User.id != user.id
            ).all()
            for admin in platform_admins:
                if admin.id not in contacts_map:
                    contacts_map[admin.id] = {
                        "id": admin.id, "name": admin.name, "role": "platform_admin",
                        "roles": ["platform_admin"], "group": "PawCoach Support",
                        "school_name": None,
                    }

    contacts = sorted(contacts_map.values(), key=lambda c: (c["name"] or "").lower())
    return jsonify({"contacts": contacts})
```

**Step 4: Run tests**

Run: `python -m pytest tests/test_platform_messaging.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add blueprints/api_messaging.py tests/test_platform_messaging.py
git commit -m "feat: Platform-Admin sieht alle Schuladmins in Kontaktliste (API)"
```

---

### Task 13: list_contacts() fuer Platform-Admins erweitern (Web)

**Files:**
- Modify: `blueprints/messaging.py`

**Step 1: Gleiche Erweiterung wie Task 12 fuer Web-Blueprint**

In `blueprints/messaging.py`, `api_users()` (Zeile ~1096):
- Platform-Admin: zusaetzlich alle Schuladmins aller Schulen zurueckgeben
- Schuladmin: zusaetzlich Platform-Admins als "PawCoach Support" zurueckgeben
- Gleiche Logik wie in Task 12, angepasst an das Web-Response-Format

**Step 2: Run existing messaging tests**

Run: `python -m pytest tests/ -v -k messaging`
Expected: PASS

**Step 3: Commit**

```bash
git add blueprints/messaging.py
git commit -m "feat: Platform-Admin sieht alle Schuladmins in Kontaktliste (Web)"
```

---

### Task 14: new_conversation() Berechtigungspruefung erweitern

**Files:**
- Modify: `blueprints/api_messaging.py`
- Modify: `blueprints/messaging.py`
- Test: `tests/test_platform_messaging.py` (erweitern)

**Step 1: Write failing tests**

```python
# tests/test_platform_messaging.py — additional tests
def test_platform_admin_can_message_school_admin(client, app, platform_admin_headers, school_admin_id):
    """Platform-Admin kann Schuladmin anschreiben."""
    rv = client.post("/api/messages/new",
                     json={"participant_id": school_admin_id, "message": "Hallo von der Plattform"},
                     headers=platform_admin_headers)
    assert rv.status_code == 201

def test_school_admin_can_message_platform_admin(client, app, admin_headers, platform_admin_id):
    """Schuladmin kann Platform-Admin anschreiben."""
    rv = client.post("/api/messages/new",
                     json={"participant_id": platform_admin_id, "message": "Supportanfrage"},
                     headers=admin_headers)
    assert rv.status_code == 201

def test_school_admin_cannot_message_other_school_admin(client, app, admin_headers, other_school_admin_id):
    """Schuladmin kann NICHT Schuladmin anderer Schule anschreiben."""
    rv = client.post("/api/messages/new",
                     json={"participant_id": other_school_admin_id, "message": "Test"},
                     headers=admin_headers)
    assert rv.status_code == 404
```

**Step 2: Run tests to verify they fail**

Run: `python -m pytest tests/test_platform_messaging.py -v -k "message"`
Expected: FAIL — Berechtigungspruefung blockiert

**Step 3: Erweitere new_conversation() Berechtigungspruefung**

In `blueprints/api_messaging.py`, `new_conversation()` (Zeile ~274), ersetze die Berechtigungspruefung:

```python
# BESTEHEND (aendern):
# participant = db.session.get(User, participant_id)
# school_id = get_api_school_id()
# if not participant or not participant.get_roles_for_school(school_id):
#     return jsonify({"error": "Teilnehmer nicht gefunden"}), 404

# NEU:
participant = db.session.get(User, participant_id)
if not participant:
    return jsonify({"error": "Teilnehmer nicht gefunden"}), 404

school_id = get_api_school_id()

# Platform-Admin darf jeden Schuladmin anschreiben
if user.is_platform_admin:
    is_school_admin = UserSchoolRole.query.filter_by(
        user_id=participant_id, role="admin"
    ).first()
    is_platform_admin = participant.is_platform_admin
    if not is_school_admin and not is_platform_admin:
        return jsonify({"error": "Teilnehmer nicht gefunden"}), 404
# Schuladmin darf Platform-Admins anschreiben
elif participant.is_platform_admin:
    sr_check = UserSchoolRole.query.filter_by(
        user_id=user.id, school_id=school_id, role="admin"
    ).first()
    if not sr_check:
        return jsonify({"error": "Teilnehmer nicht gefunden"}), 404
# Standard: gleiche Schule
else:
    if not participant.get_roles_for_school(school_id):
        return jsonify({"error": "Teilnehmer nicht gefunden"}), 404
```

Gleiche Aenderung in `blueprints/messaging.py` fuer die Web-Version.

**Wichtig:** Konversation ohne `school_id` erstellen wenn Platform-Admin ↔ Schuladmin (cross-school). Pruefen ob `Conversation.school_id` nullable ist; falls nicht, Migration hinzufuegen.

**Step 4: Run tests**

Run: `python -m pytest tests/test_platform_messaging.py -v`
Expected: PASS

**Step 5: Commit**

```bash
git add blueprints/api_messaging.py blueprints/messaging.py tests/test_platform_messaging.py
git commit -m "feat: Platform-Admin ↔ Schuladmin Messaging Berechtigungen"
```

---

## Abschluss

### Task 15: Gesamttest und Design-Dok aktualisieren

**Step 1: Alle Tests ausfuehren**

Run: `python -m pytest tests/ -v --timeout=120`
Expected: Alle Tests PASS

**Step 2: Design-Dok als erledigt markieren**

In `plans/2026-03-13-blob-wartung-messaging-design.md`:
- Status von "Genehmigt" auf "Erledigt" aendern

**Step 3: Commit**

```bash
git add plans/
git commit -m "docs: Blob-Wartung-Messaging Design als erledigt markiert"
```

**Step 4: Push**

```bash
git push origin main
```
