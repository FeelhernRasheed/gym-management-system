# Database Notes

**Phase 1 status — deliberately database-free of change:**

- The scaffold contains Django's default SQLite placeholder in `config/settings.py` (comment-marked as temporary) solely so `manage.py check` and `manage.py showmigrations` can run without external infrastructure.
- **No databases were created, connected to, modified, or migrated in Phase 1.** No records exist anywhere.
- PostgreSQL configuration (plans, connection details via environment variables, `psycopg` driver, models, migrations) is deferred to a later phase.
- Environment observation for that later phase: the local machine runs a **PostgreSQL 18** Windows service (`postgresql-x64-18`, running); `psql` is not currently on PATH. PostgreSQL 17 (as named in the plan) is not detected as installed — the phase plan should account for this.
