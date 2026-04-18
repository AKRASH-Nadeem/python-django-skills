---
trigger: always_on
---

> TECH STACK MANDATE: Never write production code for a new project or introduce a new library
> without an approved tech stack. The stack is proposed → reasoned → approved → locked.
> Deviations require explicit user confirmation and a recorded rationale.

---

# Tech Stack Decision & Library Reasoning Standards

---

## TS0. When This Rule Activates

1. **New project** — any task starting from scratch, including scaffold or "start a new API"
2. **New library introduction** — any time a library not already in `requirements/*.txt` would solve a problem

In both cases: **stop, propose, get approval, then proceed.** Never install first and explain later.

---

## TS1. New Project — Stack Proposal Protocol

### Step 1 — Gather requirements (ask only for what's missing)

```
1. App type          → REST API / WebSocket / Admin-only / Hybrid?
2. Frontend consumer → React SPA / Mobile / Server-rendered / None?
3. Auth type         → Email+password / Social OAuth / API keys / JWT / Session?
4. Data patterns     → Simple CRUD / Complex ORM / Real-time / Event-driven?
5. Async workloads?  → Background jobs needed? Scheduled tasks?
6. Scale target      → Prototype / Small team (<5) / Mid-scale / High-traffic?
7. Infrastructure    → Docker? Redis available? S3 available? Elasticsearch?
8. Deployment        → Heroku / DigitalOcean / AWS / VPS / Unknown?
```

### Step 2 — Propose with reasoning

```
## Proposed Stack — [Project name]

### Core
| Layer            | Choice                | Why this / why not alternatives |
|------------------|-----------------------|----------------------------------|
| Language         | Python 3.13           | ... |
| Framework        | Django 6.x            | ... |
| WSGI/ASGI        | Gunicorn / Uvicorn    | ... |
| Database         | PostgreSQL 17         | ... |
| ORM              | Django ORM (built-in) | ... |

### API
| Layer            | Choice                | Why this / why not alternatives |
|------------------|-----------------------|----------------------------------|
| REST Framework   | DRF 3.15              | ... |
| API Docs         | drf-spectacular       | ... |
| Auth             | SimpleJWT / Allauth   | ... |

### Infrastructure
| Layer            | Choice                | Condition                         |
|------------------|-----------------------|-----------------------------------|
| Cache            | Redis (django-redis)  | If caching needed                 |
| Task Queue       | Celery + Redis        | If background work needed         |
| Storage          | django-storages + S3  | If file uploads needed            |
| Search           | Elasticsearch         | If >100k docs or complex search   |
| Real-time        | Django Channels       | If WebSocket/SSE needed           |

### What I deliberately excluded and why
- [Library X] — [reason considered and rejected]

Confirm this stack or request changes before I begin.
```

### Step 3 — Lock the approved stack

Once approved, state:
```
Stack approved. I will:
- Never install a library outside this stack without asking first
- Explain tradeoffs before adding anything new
- Flag if a requirement arises the stack cannot serve
- Create AGENTS.md and APP_STATE.md now with the approved structure
```

Create `APP_STATE.md` and `AGENTS.md` immediately after approval.

---

## TS2. Library Selection Reasoning Matrix

### TS2.1 Library addition checklist

Before proposing any new library:

1. **Does the existing stack already solve this?** Check Django ORM, DRF built-ins, Python stdlib first.
2. **Is this a one-time need?** Write it in-house if < 50 lines and no edge cases.
3. **Is it maintained?** Last PyPI release date, open issues, download count. Flag if last release > 12 months.
4. **Security?** Run `pip-audit` after install. Check PyPI safety advisories.
5. **Does it have Django 6 compatibility?** Check setup.py/pyproject.toml `python_requires` and `install_requires`.
6. **Reversibility?** How deeply does it integrate? Can we remove it later without a full rewrite?

### TS2.2 Standard tradeoff tables

**Authentication:**

| Situation | Solution | Reason |
|---|---|---|
| SPA/mobile (JWT) | SimpleJWT | Stateless, standard, well-maintained |
| Social OAuth needed | django-allauth | Social + MFA + headless mode |
| SPA + Social | allauth.headless + SimpleJWT bridge | Best of both |
| Session (server-rendered) | Django built-in sessions | No extra package |
| Service-to-service | Custom `HasAPIKey` permission | No overhead |

**Task queues:**

| Situation | Solution | Reason |
|---|---|---|
| < 100ms operations, no retry needed | Synchronous (no queue) | Overhead not justified |
| Background work, retry needed | Celery + Redis | Production standard |
| Scheduled jobs | Celery Beat + django-celery-beat | ORM-backed, production safe |
| Simple periodic (no Redis) | APScheduler or management command + cron | Simpler, no broker |

**Caching:**

| Situation | Solution | Reason |
|---|---|---|
| Development / tests | LocMemCache (built-in) | Zero infrastructure |
| Production, single server | Redis (django-redis) | Fast, persistent |
| Session + cache | Redis for both | Share one connection pool |
| Per-user personalised cache | Redis + user_id in key | Never share cache across users |

**Search:**

| Situation | Solution | Reason |
|---|---|---|
| Simple contains search, <100k rows | `icontains` ORM | Zero infrastructure |
| PostgreSQL FTS, <100k rows | `SearchVector + SearchQuery` | Built-in, no extra infra |
| >100k docs, relevance scoring | Elasticsearch | Purpose-built |
| Autocomplete | Elasticsearch edge-n-gram | Best accuracy |

**File storage:**

| Situation | Solution |
|---|---|
| Dev/test | `InMemoryStorage` or local filesystem |
| Production (public files) | S3 + CloudFront CDN via django-storages |
| Production (private files) | S3 + signed URL (short TTL) via django-storages |
| Large direct uploads | Pre-signed POST (browser → S3 direct) |

---

## TS3. Stack Profiles

### Profile A — Minimal API (prototype / internal tool)
```
Django 6 + DRF + PostgreSQL + SimpleJWT + drf-spectacular
Configuration: django-environ
Testing: pytest-django + factory-boy
```

### Profile B — Standard Product (SaaS)
```
Profile A + Celery + Redis + django-allauth + django-storages + Sentry
Configuration: django-environ + split settings
```

### Profile C — High-Throughput / Event-Driven
```
Profile B + Elasticsearch + Channels + pgBouncer
Real-time: Django Channels + Redis channel layer
```

### Profile D — Real-time Collaboration
```
Profile B + Django Channels + WebSocket consumers + Redis pub/sub
```

---

## TS4. Library Veto Rules

| Banned | Reason | Alternative |
|---|---|---|
| `django-rest-auth` | Unmaintained, superseded | `dj-rest-auth` or `allauth.headless` |
| `django-tastypie` | Superseded by DRF | DRF |
| `south` | Replaced by Django migrations | Django built-in migrations |
| `Fabric` (v1) | Outdated, py2-era | Fabric3 or Ansible |
| `bleach` | Unmaintained as of 2023 | `nh3` (Rust-based, maintained) |
| `PyJWT` direct usage | Bypasses DRF auth layer | SimpleJWT via DRF |
| `requests` in views | Synchronous blocking | `httpx` async or Celery task |

**Pushback script:**
> *"[Library] is not recommended — [reason]. The replacement is [alternative]. Should I proceed with that?"*

---

## TS5. Dependency Addition Protocol (mid-project)

1. **Stop.** State: *"I need [library] for [feature]. Let me verify this is justified."*
2. Run TS2.1 checklist.
3. Propose: *"Maintained: yes (last release [N], [M] downloads/month). Alternative [Y] rejected because [reason]. Approve?"*
4. Install only after approval.
5. Document in `LIBRARY_LEDGER.md` — see `library-ledger.md`.
6. Update `APP_STATE.md` if it changes the Tech Stack section.

---

## Summary

| Situation | Action |
|---|---|
| New project | TS1 proposal → approval → create AGENTS.md + APP_STATE.md → begin |
| New library | TS2.1 checklist → propose → wait for approval → TS5 |
| Banned library requested | Decline → explain → propose alternative |
| Choosing between two libraries | TS2.2 tables → state choice + why not the other |
| Existing stack covers the need | State explicitly — no new library |
