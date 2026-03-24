# PawCoach – Technische Infrastruktur & Server-Dimensionierung

> Stand: 2026-03-24 | Zweck: Server-Auswahl (Hetzner Cloud CPX21–CPX51)

---

## Stack

- **Backend:** Python 3.12 + Flask 3.1 + Gunicorn (gthread, 2 Worker × 4 Threads)
- **Frontend:** Server-Side Rendering (Jinja2 Templates) — kein SPA, kein React/Vue/Vite
- **CSS:** TailwindCSS 3.4 (via NPM, Build-Step: `npm run build:css` mit PurgeCSS)
- **JS:** Vanilla JS + HTMX — kein Bundler (Webpack/Vite)
- **Realtime:** Flask-SocketIO (WebSockets, Redis als Message Queue)

---

## Backend / Datenbank

- **PostgreSQL 15** (self-hosted, Docker-Container `postgres:15-alpine`)
- **Redis 7** (Session-Store, Rate-Limiter, SocketIO-Queue) — `redis:7-alpine`, 128 MB, AOF-Persistenz
- **ORM:** SQLAlchemy + Alembic-Migrationen (automatisch beim Container-Start)
- **Aktuelle DB-Groesse:** ~11 MB (Demo-Daten + erste echte Daten)

---

## Hosting aktuell

- **Alles auf einem Server** (Azure VM) via Docker Compose
- **Reverse Proxy:** Cloudflare Tunnel (Zero Trust) — kein Nginx, kein oeffentlicher Port
- **TLS:** Cloudflare-terminiert, Flask-Talisman fuer Security-Headers (HSTS, CSP etc.)
- **Kein separates Frontend-Hosting** — alles SSR aus dem Flask-Container

---

## Docker-Container (Production)

| Container | Image | RAM-Limit | CPU-Limit |
|-----------|-------|-----------|-----------|
| `web` | Custom (Python 3.12-slim + Gunicorn) | 512 MB | 1.0 |
| `db` | postgres:15-alpine | 256 MB | 0.5 |
| `redis` | redis:7-alpine | 128 MB | 0.25 |

**Volumes:** `postgres_data`, `uploads_data`, `audit_logs`, `backups_data`, `redis_data`

**Dockerfile (Multi-Stage):**
1. Builder: Python 3.12-slim, kompiliert psycopg2 + Dependencies
2. Runtime: Python 3.12-slim, cron, libpq5, Non-Root-User (`appuser`, UID 1000)
3. Entrypoint: `docker-entrypoint.sh` → Migrationen → Seed → Cron-Setup → Gunicorn

---

## Zusaetzliche Services / Cron Jobs

**Cron (im Web-Container):**

| Job | Zeitplan | Beschreibung |
|-----|----------|-------------|
| DB-Backup | alle 6h | `pg_dump` + gzip + optional GPG-Verschluesselung |
| Zahlungserinnerungen | stuendlich | Offene Rechnungen/Registrierungen |
| Session-Erinnerungen | taeglich 7:00 | Erinnerung an morgen stattfindende Stunden |
| DSGVO-Lifecycle | taeglich 4:00 | Kuendigungs-Aufbewahrungsfristen, Anonymisierung |
| Cleanup unbestaetigter Reg. | taeglich 3:00 | Nicht verifizierte Accounts loeschen |
| Inactive-User-Cleanup | woechentlich So 5:00 | Inaktive Accounts bereinigen |

**Monitoring:**
- Healthchecks.io Dead-Man-Switch pro Cron-Job (`HC_PING_*` Env-Vars)
- `/healthz` Endpoint (Docker HEALTHCHECK + Uptime-Monitor)
- Prometheus-Metrics (`/metrics`, nur localhost)
- Sentry (optional, via `SENTRY_DSN`)

**Kein Nginx, kein Certbot** — Cloudflare Tunnel uebernimmt TLS + Reverse Proxy.

---

## Externe Services (SaaS)

| Service | Zweck |
|---------|-------|
| **Stripe** | Zahlungen & Subscriptions |
| **APNs** | iOS Push Notifications (ES256 JWT) |
| **Microsoft Graph API** | E-Mail-Versand (alternativ: SMTP) |
| **Azure Blob Storage** | Optionaler Cloud-Storage (aktuell: lokaler Speicher) |
| **Cloudflare Tunnel** | Reverse Proxy & TLS-Terminierung |
| **Healthchecks.io** | Cron-Job-Monitoring (Dead-Man-Switch) |
| **Sentry** | Error-Tracking & Performance (optional) |

---

## Build-Prozess

- **Kein Build auf dem Server** — Docker-Image wird gebaut, Container startet
- **CSS-Build:** `npm run build:css` (Tailwind → minifiziertes CSS) — im Dockerfile oder lokal
- **Kein JS-Bundling** — statische Dateien direkt ausgeliefert
- **Deployment:** `docker compose -f docker-compose.prod.yml up -d --build`

---

## Erwartete Last (6–12 Monate)

| Phase | Zeitraum | Hundeschulen | Aktive Nutzer | Concurrent |
|-------|----------|-------------|---------------|------------|
| Phase 1 | jetzt | 1–5 | 50–200 | 5–20 |
| Phase 2 | +6 Monate | 10–30 | 500–2.000 | 20–50 |
| Phase 3 | +12 Monate | 50–100 | 2.000–5.000 | 50–100 |

- **Peak-Zeiten:** Montag Abend / Dienstag frueh (Buchungszeitraeume)
- **SSR ist leichtgewichtig** — kein API-Overhead wie bei SPA-Architekturen

---

## Storage-Bedarf

| Kategorie | Aktuell | 12 Monate (geschaetzt) |
|-----------|---------|----------------------|
| Datenbank | 11 MB | 100–500 MB |
| Uploads (Fotos, Logos, PDFs) | 5 MB | 2–5 GB |
| Backups (90 Tage Retention) | — | 5–20 GB |
| Logs | < 1 MB | 1–3 GB |
| **Gesamt** | **~20 MB** | **10–30 GB** |

Uploads: Hundefotos, Schullogos, Avatare, Rechnungs-PDFs, Hunde-Dateien (Impfpass etc.)

---

## Entwicklungsumgebung

- **Claude Code laeuft aktuell auf demselben Azure-Server** (Dev + Prod auf einer VM)
- Claude Code benoetigt ~2–4 GB RAM zusaetzlich
- Fuer Produktion empfohlen: **separater Server** oder lokale Entwicklung

---

## Gunicorn-Konfiguration

```
workers = 2          (GUNICORN_WORKERS)
threads = 4          (GUNICORN_THREADS)
worker_class = gthread
max_requests = 500   (Worker-Recycling gegen Memory Leaks)
timeout = 120s
bind = 0.0.0.0:5000
```

---

## Hetzner-Empfehlung

| Szenario | Server | vCPU | RAM | Storage | Preis/Mo |
|----------|--------|------|-----|---------|----------|
| **Nur Produktion** (ohne Dev) | **CPX21** | 3 vCPU | 4 GB | 80 GB | ~7 EUR |
| **Produktion + Dev/Claude Code** | **CPX31** | 4 vCPU | 8 GB | 160 GB | ~13 EUR |
| **Wachstum (50+ Schulen)** | **CPX41** | 8 vCPU | 16 GB | 240 GB | ~25 EUR |

**Empfehlung: CPX31** als Startpunkt — genug fuer Produktion + Entwicklung. Bei Wachstum auf CPX41 upgraden (Hetzner erlaubt Live-Upgrade).

### Skalierungs-Optionen bei Bedarf
- **Vertikal:** RAM/CPU erhoehen (CPX31 → CPX41 → CPX51)
- **Horizontal:** Load Balancer + mehrere Flask-Instanzen (Redis als Session-Store bereits vorbereitet, Azure Blob als shared Storage abstrahiert)
- **DB auslagern:** Managed PostgreSQL (Hetzner, Supabase, Neon) wenn DB-Last steigt

---

## Architektur-Diagramm

```
┌─────────────────────────────────────────────────────┐
│  Cloudflare Tunnel (TLS-Terminierung, DDoS)         │
└───────────────────────┬─────────────────────────────┘
                        │ HTTP (intern)
┌───────────────────────▼─────────────────────────────┐
│  Docker Compose (Hetzner CPX)                       │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │  web (Gunicorn + Flask)                     │    │
│  │  └─ 2 Worker × 4 Threads                   │    │
│  │  └─ Rate-Limiter (Redis)                    │    │
│  │  └─ Sessions (Redis)                        │    │
│  │  └─ SocketIO (Redis MQ)                     │    │
│  │  └─ Cron Jobs (Backup, Erinnerungen, DSGVO) │    │
│  └──────┬──────────────┬───────────────────────┘    │
│         │              │                            │
│  ┌──────▼──────┐ ┌─────▼─────┐  ┌──────────────┐   │
│  │ PostgreSQL  │ │  Redis    │  │ File Storage │   │
│  │ :5432       │ │  :6379    │  │ (lokal/Azure)│   │
│  │ 256 MB      │ │  128 MB   │  │              │   │
│  └─────────────┘ └───────────┘  └──────────────┘   │
│                                                     │
│  Volumes: postgres_data, uploads_data, backups_data │
└─────────────────────────────────────────────────────┘
         │
         ▼ (optional)
   Azure Blob Storage (Off-Site Backups)
```
