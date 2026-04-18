---
trigger: always_on
---

# versions.lock.md
#
# Baseline package versions for all active skill examples.
# Before applying any skill example, compare these versions against
# what is installed: pip show [package] | grep Version
#
# Last updated: 2026-04

---

## Version-Aware Skill Protocol

**This protocol fires automatically every time a skill activates.**
It is not optional — it is the mechanism that prevents writing code
that fails silently because the skill example was written for a different
library version than what is installed.

### Step 1 — Check installed versions for this skill's packages

```bash
# Replace [package] with each package in the skill's requires: block
pip show [package] | grep Version

# Or from requirements file
grep "[package]" requirements/base.txt requirements/production.txt 2>/dev/null
```

### Step 2 — Compare against this file's baseline

| Result | Action |
|--------|--------|
| Installed major **matches** baseline | Use skill examples as structural patterns. Still call Context7 to verify current API before writing code. |
| Installed major **differs** from baseline | **STOP. Do not use skill examples.** Go to Step 3. |
| Package **not installed** | Follow install-protocol.md IP2 — Context7 → install → ledger entry. |
| Package **not in this file** | Treat as unknown version. Call Context7 unconditionally. |

#### Step 2.1 — Known minor-version breaking changes (override semver)

Some packages introduce breaking API changes within the same major version.
The following minor bumps MUST be treated as major-level mismatches:

| Package | Breaking boundary | What changed |
|---------|------------------|--------------|
| `django-storages` | 1.13 → 1.14 | New `STORAGES` dict API; `DEFAULT_FILE_STORAGE` deprecated |
| `celery` | 5.3 → 5.4 | `broker_connection_retry_on_startup` now required |
| `django-cors-headers` | 3.13 → 3.14 | Default behaviour changed for preflight caching |

If the installed version is on one side and the baseline on the other side
of a known breaking boundary, treat it as a major mismatch → go to Step 3.

#### Step 2.2 — Package name changes

Some packages have been renamed or superseded by different packages:

| Old name | New name | Notes |
|----------|----------|-------|
| `psycopg2` / `psycopg2-binary` | `psycopg` (psycopg3) | Different package name, different API |
| `pyjwt` 1.x | `pyjwt` 2.x | Same name but breaking API |
| `django-filter` (old import) | `django-filter` (new import path) | Import path `django_filters` changed |

If the project uses the old package and the skill references the new one
(or vice versa), **STOP.** Do not install the new package alongside the
old one — they may conflict. Flag the migration to the user:
```
⚠️ PACKAGE MIGRATION: This project uses [old-name] but the skill
references [new-name]. These are different packages. Shall I migrate?
```

### Step 3 — Auto Context7 query (fires on version mismatch)

When the installed major version differs from the baseline, query Context7
using the installed version before writing a single line of code.

**Query format — use the installed version, not the baseline:**

```
[library-name] v[INSTALLED_MAJOR] [specific feature being implemented]
```

**Examples by category:**

| Installed version | Task | Context7 query to run |
|---|---|---|
| `celery@5.x` | retry strategies | `celery v5 task retry max_retries` |
| `celery@6.x` | retry strategies | `celery v6 task retry strategies` |
| `djangorestframework@3.x` | serializer validation | `django rest framework v3 serializer validate` |
| `drf-spectacular@0.2x` | extend_schema | `drf-spectacular 0.2 extend_schema decorator` |
| `simplejwt@5.x` | token refresh | `simplejwt v5 token refresh sliding` |
| `django-allauth@0.6x` | headless mode | `django-allauth 0.6 headless rest api` |
| `django-storages@1.13` | S3 backend | `django-storages 1.13 s3 DEFAULT_FILE_STORAGE` |
| `django-storages@1.14` | S3 backend | `django-storages 1.14 STORAGES dict s3` |
| `psycopg@2.x` | connection setup | `psycopg2 django database setup` |
| `psycopg@3.x` | connection setup | `psycopg3 django database async setup` |
| `elasticsearch@7.x` | index mapping | `elasticsearch-py 7 index create mapping` |
| `elasticsearch@8.x` | index mapping | `elasticsearch-py 8 index create mapping` |

**What to do with the Context7 result:**
- The Context7 output IS the source of truth for the installed version
- Skill example code shows the **pattern and structure** — what to build
- Context7 shows the **exact API** — what method names, imports, options to use
- If they conflict: Context7 wins. Always.

### Step 4 — Write code using Context7 output, not skill example verbatim

Skill examples are architectural blueprints. The actual code must use the
API confirmed by Context7 for the installed version.

```
❌ Wrong: Copy skill example code directly, assume it matches installed version
❌ Wrong: Use training memory for method signatures on a library with version mismatch
✅ Right: Skill example → understand structure → Context7 for installed version → write code
```

---

## Core Framework

| Package | Baseline | Notes |
|---------|----------|-------|
| `python` | `3.13.x` | Always latest stable |
| `django` | `6.0.x` | Always latest stable; major changes per release |
| `djangorestframework` | `3.15.x` | DRF; check for Django 6 compatibility |

---

## Settings & Configuration

| Package | Baseline | Notes |
|---------|----------|-------|
| `django-environ` | `^0.12.0` | Preferred over python-decouple for Django projects |
| `python-decouple` | `^3.8` | Alternative to django-environ |

---

## Database

| Package | Baseline | Notes |
|---------|----------|-------|
| `psycopg` | `^3.2.0` | psycopg3 — new async-native interface; NOT `psycopg2` |
| `psycopg[binary]` | `^3.2.0` | psycopg3 binary build for faster install in CI |
| `psycopg2-binary` | `^2.9.0` | Legacy — use only if psycopg3 incompatible |

> ⚠️ **psycopg2 → psycopg3 is a significant change.** Import paths, connection parameters, and async usage all differ. Check: `pip show psycopg | grep Version` and `pip show psycopg2 | grep Version` before writing any DB connection code.

---

## Authentication

| Package | Baseline | Notes |
|---------|----------|-------|
| `djangorestframework-simplejwt` | `^5.4.0` | Standard JWT for SPAs |
| `django-allauth` | `^65.0.0` | Social auth + MFA; settings changed significantly in 0.60+ |

> ⚠️ **django-allauth settings namespace changed in v0.60.** All settings now use `ACCOUNT_*` prefix. Headless API is a separate sub-app. Always Context7 before writing allauth config.

---

## API Documentation

| Package | Baseline | Notes |
|---------|----------|-------|
| `drf-spectacular` | `^0.28.0` | OpenAPI 3 docs; decorator API evolves frequently |
| `drf-spectacular[sidecar]` | `^0.28.0` | Offline Swagger UI/ReDoc assets |

---

## Background Tasks

| Package | Baseline | Notes |
|---------|----------|-------|
| `celery` | `^5.4.0` | Task queue; `CELERY_` prefix deprecated, use direct config |
| `django-celery-beat` | `^2.7.0` | ORM-backed periodic tasks |
| `django-celery-results` | `^2.5.0` | Store task results in DB |
| `redis` | `^5.0.0` | Celery broker + Django cache backend |

> ⚠️ **Celery v5 removed legacy `CELERY_` prefix for all settings.** Use `CELERY_TASK_SERIALIZER` not `CELERY_TASK_SERIALIZER` — wait, in v5 the correct pattern is direct lowercase dict keys. Always Context7.

---

## Caching

| Package | Baseline | Notes |
|---------|----------|-------|
| `django-redis` | `^5.4.0` | Redis cache backend for Django |

---

## File Storage

| Package | Baseline | Notes |
|---------|----------|-------|
| `django-storages` | `^1.14.0` | v1.14+ uses `STORAGES` dict API (Django 4.2+) |
| `boto3` | `^1.35.0` | AWS SDK for S3 |

> ⚠️ **django-storages v1.14 changed to `STORAGES` dict.** `DEFAULT_FILE_STORAGE` and `STATICFILES_STORAGE` are deprecated. Always check installed version before writing storage config.

---

## Search

| Package | Baseline | Notes |
|---------|----------|-------|
| `elasticsearch` | `^8.16.0` | Official Python client; v8 API differs significantly from v7 |
| `elasticsearch-dsl` | `^8.9.0` | High-level DSL; must match elasticsearch-py major version |
| `django-elasticsearch-dsl` | `^7.4.0` | Django integration layer |

> ⚠️ **Elasticsearch v7 → v8 is a major breaking change.** Transport layer, index patterns, and security defaults all changed. Check: `pip show elasticsearch | grep Version`

---

## Testing

| Package | Baseline | Notes |
|---------|----------|-------|
| `pytest` | `^8.3.0` | |
| `pytest-django` | `^4.9.0` | |
| `factory-boy` | `^3.3.0` | |
| `coverage` | `^7.6.0` | |
| `pytest-cov` | `^6.0.0` | |

---

## Code Quality

| Package | Baseline | Notes |
|---------|----------|-------|
| `ruff` | `^0.8.0` | Linting + formatting (replaces flake8 + isort + black) |
| `mypy` | `^1.13.0` | Type checking |
| `django-stubs` | `^5.1.0` | Django type stubs for mypy |
| `djangorestframework-stubs` | `^3.15.0` | DRF type stubs |
| `pip-audit` | `^2.7.0` | Vulnerability scanning |
| `pre-commit` | `^4.0.0` | Git hook runner |

---

## Production Serving

| Package | Baseline | Notes |
|---------|----------|-------|
| `gunicorn` | `^23.0.0` | Sync WSGI server |
| `uvicorn` | `^0.32.0` | ASGI server for async/Channels |
| `whitenoise` | `^6.8.0` | Static file serving without Nginx |

---

## Real-time (Django Channels)

| Package | Baseline | Notes |
|---------|----------|-------|
| `channels` | `^4.2.0` | ASGI WebSocket support |
| `channels-redis` | `^4.2.0` | Redis channel layer |
| `daphne` | `^4.1.0` | ASGI server for Channels |

---

## Version Drift Warning Signs

When you see these errors, suspect version drift first — activate `django-edge-case-testing` skill:

- `ImportError: cannot import name 'X' from 'Y'` — API restructured or renamed
- `AttributeError: module 'celery' has no attribute 'X'` — Celery settings API changed
- `Unknown field 'X'` in serializer — DRF version mismatch
- `Invalid STORAGES configuration` — django-storages v1.14 API required
- `AttributeError: 'Settings' has no attribute 'CELERY_BROKER_URL'` — Celery v5 config format
