# Architecture Notes

This directory will hold the backend's architecture documentation (request flow, app boundaries, API design principles).

**Phase 1 status (population deferred to later phases by design):**

- The project is a single Django `config` package (`settings.py`, `urls.py`, `asgi.py`, `wsgi.py`) initialized directly in the workspace root, with Django 6.1.3 and Django REST Framework 3.18.3 installed and verified.
- Only framework defaults exist: Django admin is wired in `config/urls.py`; DRF is registered in `INSTALLED_APPS` with explicit `AllowAny` permission defaults (to be tightened when authentication is introduced).
- No business apps, models, serializers, or endpoints exist yet.
- Planned modules (documented here when implemented): authentication, members, memberships, payments, attendance.
