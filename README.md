# Gym Management System — Backend

Django + Django REST Framework backend for the Gym Management System, initialized with industry-standard practices: environment-aware settings, no committed secrets, and a pinned dependency manifest.

**Status:** Phase 1 — Environment setup and Django project initialization ✅ (no business modules implemented yet; see [docs/progress](docs/progress/phase-1-initialization.md)).

---

## Project Overview

This repository contains the backend service for a gym management platform. At this stage (Phase 1) it is a clean, verified Django project scaffold with Django REST Framework installed and configured. Business modules (authentication, members, memberships, payments, attendance) will be added incrementally in later phases and tracked in `docs/progress/`.

## Technology Stack

| Layer | Technology | Version (installed) |
|---|---|---|
| Language | Python | 3.14.4 |
| Framework | Django | 6.1.3 |
| REST API | Django REST Framework | 3.18.3 |
| Database (planned) | PostgreSQL | 17/18 (configured in a later phase; not connected yet) |
| ORM | Django ORM (bundled) | — |
| Version control | Git | 2.54.0 |

Compatibility of Python 3.14, Django 6.1 and DRF 3.18 was verified against official PyPI metadata before installation (both packages declare Python 3.14 and Django 6.1 support).

## Prerequisites

- **Python 3.14** available as `python` on PATH
- **pip** (bundled with Python)
- **Git** (2.x)
- VS Code (recommended editor)
- PostgreSQL will only be required in the database phase; nothing is needed for this scaffold.

## Project Structure

```
backend/ (project root)
├── .venv/               # Python virtual environment (not committed)
├── config/              # Django project configuration package
│   ├── __init__.py
│   ├── settings.py      # Environment-aware settings (secrets via env vars)
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
├── docs/
│   ├── architecture/    # Architecture notes (populated in later phases)
│   ├── database/        # Database/schema notes (populated in later phases)
│   └── progress/        # Per-phase work logs
├── manage.py
├── requirements.txt     # Pinned, actually-installed versions
├── static/              # Project static assets
├── media/               # User-uploaded files (git-ignored)
└── README.md
```

Note: the project is initialized directly in the workspace root — there is no nested `backend/backend` directory. `manage.py` sits beside `config/`.

## Getting Started

### 1. Create the virtual environment

From the project root (PowerShell):

```powershell
python -m venv .venv
```

### 2. Activate the virtual environment

```powershell
.\.venv\Scripts\Activate.ps1
```

If PowerShell blocks the script, run once (with Administrator rights if required):

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

Verification: after activation the prompt shows `(.venv)` and:

```powershell
python --version
python -m pip --version   # should reference .venv\Lib\site-packages
```

### 3. Install dependencies

```powershell
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

### 4. Start the development server

```powershell
python manage.py migrate   # applies to the SQLite placeholder only; PostgreSQL replaces it in the database phase
python manage.py runserver
```

Then open:

- `http://127.0.0.1:8000/` — shows Django's default 404 page until project URLs are added (expected for a fresh scaffold)
- `http://127.0.0.1:8000/admin/` — Django admin (default scaffold)

Stop the server with `CTRL+C`.

## Django Verification Commands

Run these from the project root with the venv active:

```powershell
python --version                      # Python 3.14.4
python -m pip --version               # pip 26.0.1 inside .venv
python -m django --version            # 6.1.3
python -m pip show djangorestframework
python manage.py check                # no issues expected
python manage.py showmigrations       # migration state
```

## Configuration

| Setting | Source of truth | Default (development) |
|---|---|---|
| Secret key | `DJANGO_SECRET_KEY` env var; otherwise auto-generated into git-ignored `.secret_key` (dev only) | — |
| DEBUG | `DJANGO_DEBUG` env var | `True` |
| Allowed hosts | `DJANGO_ALLOWED_HOSTS` env var (comma-separated) | `localhost,127.0.0.1,[::1]` |
| Time zone | Fixed in code | `Africa/Nairobi` (USE_TZ=True) |
| Language | Fixed in code | `en-us` |
| Static files | `STATIC_URL=static/`, `STATICFILES_DIRS=static/`, `STATIC_ROOT=staticfiles/` | — |
| Media | `MEDIA_URL=media/`, `MEDIA_ROOT=media/` | — |

No secrets are hardcoded; the only generated key lives in `.secret_key`, which is git-ignored. Production hardening (SSL redirect, HSTS, secure cookies) is applied automatically when `DEBUG` is `False`.

## Database Notes

The scaffold ships with Django's default SQLite placeholder so the project runs without external infrastructure. **It is intentional that PostgreSQL is not yet connected.** The database layer will be switched to PostgreSQL (via environment variables, with a requirements addition such as `psycopg`) in a dedicated later phase; see [docs/database/README.md](docs/database/README.md).

## Development Workflow

- Docs of record per phase live in `docs/progress/`.
- Git: commits are made only with explicit approval at this stage; `.gitignore` protects `.venv/`, databases, media uploads, and secrets.
