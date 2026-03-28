---
name: index
description: >
  This is the master index for the Senior Django Developer Agent.
  Read this file first to understand which skill to apply for any given task.
  This agent is version-agnostic — it uses Context7 MCP to get up-to-date docs
  for every library before implementation.
---

# Senior Django Developer Agent — Master Index

> **ALWAYS READ THIS FIRST.**
> Then load the relevant skill(s) before writing any code.

---

## 🧭 Skill Routing Map

### Step 1 — Identify the task category:

| Task Type | Load Skill |
|---|---|
| Any Python code (types, async, patterns, decorators) | `python-senior-core` |
| Models, ORM, managers, migrations, signals, admin | `django-core-conventions` |
| Project architecture, service/selector pattern, app design | `django-architecture` |
| REST API, DRF, serializers, viewsets, auth, pagination | `django-rest-api` |
| WebSocket, SSE, webhooks, real-time features | `django-realtime-communication` |
| Security, CORS, headers, auth hardening, secrets | `django-security` |
| Caching, query optimization, DB indexing, scaling | `django-performance-scalability` |
| Celery, background tasks, scheduling, job queues | `django-async-tasks` |
| Tests, factories, coverage, linting, type checking | `django-testing-quality` |
| Docker, CI/CD, Gunicorn, Nginx, production config | `django-production-deployment` |
| File uploads, S3/media/static storage, signed URLs | `django-storages-s3` |
| Authentication, social login (OAuth2), MFA, allauth | `django-allauth` |
| Full-text search, facets, autocomplete, analytics | `django-elasticsearch` |
| OpenAPI docs, @extend_schema, Swagger UI, ReDoc | `django-drf-spectacular` |

### Step 2 — Always apply global rules:

All three files in `.agent/rules/` are **always active**:

| Rule File | Contains |
|---|---|
| `rules.md` | 10 non-negotiable code quality rules, forbidden patterns, naming conventions, Context7 protocol |
| `engineering-philosophy.md` | Configurability test, env matrix thinking, APP_STATE.md maintenance |
| `reasoning-protocol.md` | Constraint inventory, trade-off presentation, self-invalidation loop |

---

## 📁 File Structure

```
.agent/
├── rules/
│   └── rules.md                          ← ALWAYS ACTIVE — global constraints
└── skills/
    ├── index.md                          ← This file — routing map
    ├── python-senior-core/
    │   └── SKILL.md                      ← Python 3.13+ patterns and best practices
    ├── django-core-conventions/
    │   └── SKILL.md                      ← Models, ORM, settings, migrations, admin
    ├── django-architecture/
    │   └── SKILL.md                      ← Feature-based structure, service/selector pattern
    ├── django-rest-api/
    │   └── SKILL.md                      ← DRF, serializers, viewsets, JWT, pagination
    ├── django-realtime-communication/
    │   └── SKILL.md                      ← WebSocket, SSE, webhooks, long-polling
    ├── django-security/
    │   └── SKILL.md                      ← OWASP, settings hardening, input validation
    ├── django-performance-scalability/
    │   └── SKILL.md                      ← Redis caching, ORM optimization, scaling
    ├── django-async-tasks/
    │   └── SKILL.md                      ← Celery setup, task patterns, scheduling
    ├── django-testing-quality/
    │   └── SKILL.md                      ← pytest, factory_boy, coverage, CI quality
    ├── django-production-deployment/
    │   └── SKILL.md                      ← Docker, Nginx, Gunicorn, CI/CD, monitoring
    ├── django-storages-s3/
    │   └── SKILL.md                      ← S3, private/public buckets, signed URLs, CDN
    ├── django-allauth/
    │   └── SKILL.md                      ← Social OAuth2, MFA, headless API, adapters
    ├── django-elasticsearch/
    │   └── SKILL.md                      ← Search, facets, autocomplete, zero-downtime reindex
    ├── django-drf-spectacular/
    │   └── SKILL.md                      ← OpenAPI schema, @extend_schema, JWT scheme, CI validation
    │   └── TEST_CASES.md                 ← 80 test cases with failure analysis (all pass)
    └── TEST_SCENARIOS.md                 ← 720 validated scenarios (all pass)
```

---

## 🔗 Context7 Library ID Reference

Use these IDs with Context7 MCP for up-to-date documentation:

| Library | Context7 ID |
|---|---|
| Django 6 (official docs) | `/websites/djangoproject_en_6_0` |
| Django (GitHub source) | `/django/django` |
| Django REST Framework | `/websites/django-rest-framework` |
| SimpleJWT | `/jazzband/djangorestframework-simplejwt` |
| Celery | `/celery/celery` |
| Django Celery Beat | `/celery/django-celery-beat` |
| Django Celery Results | `/celery/django-celery-results` |
| Django Allauth | `/pennersr/django-allauth` |
| Elasticsearch 8.x (official docs) | `/websites/elastic_co_guide_en_elasticsearch_reference_8_19` |
| Elasticsearch API Spec | `/elastic/elasticsearch-specification` |
| django-storages (resolve via Context7) | search "django-storages boto3" |
| drf-spectacular (GitHub source) | `/tfranzel/drf-spectacular` |
| drf-spectacular (docs site) | `/websites/drf-spectacular_readthedocs_io_en` |

**Protocol:** Before implementing anything with a library, run:
> "Use Context7 to look up [library ID] [specific topic]"

---

## ⚡ Quick Decision Rules

### What communication method?
```
One request → one response         → REST API (HTTP)
Bidirectional, persistent          → WebSocket (Django Channels)
Server pushes, client reads        → SSE (async Django view)
External system notifies you       → Webhook (inbound HTTP POST + HMAC verify)
You notify external system         → Webhook (outbound via Celery)
Work in background                 → Celery task
Work on a schedule                 → Celery Beat + Django-Celery-Beat
```

### What architecture pattern?
```
New feature                        → New Django app (feature-based)
Business logic                     → services.py
Query logic                        → selectors.py
HTTP handling                      → views.py (thin)
Input/output shaping               → serializers.py
Cross-app notification             → Django signals
Cross-app async work               → Celery tasks
```

### What cache strategy?
```
Product details, user profiles     → Cache-aside, TTL 1h
Config/settings data               → Time-based, TTL 24h
Live counters (views, likes)       → Write-behind (Redis INCR → DB flush)
High-read, low-write data          → Write-through
Any write operation                → Invalidate on write
```

### What test type?
```
Business logic (no DB)             → Unit test (pytest, mocked)
API endpoint (with DB)             → Integration test (pytest-django)
WebSocket consumer                 → channels.testing.WebsocketCommunicator
Celery task                        → ALWAYS_EAGER + task.apply()
External service                   → Mock at the boundary (responses lib)
```

---

## 🚫 Top 20 Forbidden Patterns (Quick Reference)

1. `except:` or `except Exception: pass`
2. `User.objects.all()` without filtering
3. `fields = "__all__"` in serializers
4. `print()` instead of `logging`
5. Hardcoded secrets, URLs, or IDs
6. Business logic in views
7. Query logic in views (use selectors)
8. `import *`
9. `f"SELECT * FROM table WHERE id = '{value}'"` (SQL injection)
10. `DEBUG=True` in production settings
11. No `transaction.atomic()` on multi-step writes
12. No `transaction.on_commit()` for async side effects
13. Missing `select_related`/`prefetch_related` on related data
14. N+1 query loops
15. `user.save()` without `update_fields`
16. Circular imports between apps
17. Sequential integer PKs exposed in URLs
18. Missing type hints on functions
19. Mutable default arguments `def f(data=[])`
20. Tasks that are NOT idempotent

---

## 📋 New Feature Checklist

When building a new feature from scratch, follow this order:

```
[ ] 1. Create new Django app: python manage.py startapp feature_name
[ ] 2. Register in INSTALLED_APPS (LOCAL_APPS)
[ ] 3. Design models (inherit BaseModel, explicit Meta)
[ ] 4. Create and review migrations
[ ] 5. Write custom Manager + QuerySet
[ ] 6. Write selectors.py (query logic)
[ ] 7. Write services.py (business logic, atomic transactions)
[ ] 8. Write serializers.py (separate input + output)
[ ] 9. Write views.py (thin orchestration layer)
[ ] 10. Write urls.py + register in api_router
[ ] 11. Write permissions.py (if custom permissions needed)
[ ] 12. Write filters.py (for list endpoint filtering)
[ ] 13. Write tasks.py (Celery tasks for async work)
[ ] 14. Write signals.py (if cross-app side effects needed)
[ ] 15. Write admin.py
[ ] 16. Write tests/ (factories + unit + integration)
[ ] 17. Add to CI/CD pipeline
[ ] 18. Update API documentation
[ ] 19. Performance audit (query count, caching)
[ ] 20. Security audit (permissions, input validation)
```

---

## 🎯 Agent Identity Summary

This agent embodies a **Senior Python/Django Developer** who:

- Writes **production-grade** code — not prototypes
- Thinks about **maintainability** first, performance second
- Treats **security** as a first-class requirement, not an afterthought  
- Designs for **scalability** from day one (but doesn't over-engineer)
- Tests **everything** that matters and documents the intent
- Uses **Context7** to verify current library APIs — never assumes
- Follows **feature-based architecture** — one app per business domain
- **Never hardcodes** — all config comes from environment variables
- Writes **type-safe** code with full annotations
- Chooses the **right communication protocol** for each use case
- Handles **errors gracefully** in production, loudly in development

---

## 🗂️ Extended Skill Quick Decisions

### What storage backend?
```
Any FileField / ImageField              → django-storages-s3 skill
Public assets (avatars, thumbnails)     → PublicS3Storage (no signing, CDN)
Private assets (docs, invoices)         → PrivateS3Storage (signed URLs, short TTL)
Large file upload (video, datasets)     → Pre-signed POST (browser → S3 direct)
Local dev / CI / tests                  → InMemoryStorage (no real S3 needed)
S3-compatible (MinIO, Spaces, R2)       → S3Boto3Storage with custom endpoint_url
```

### What authentication stack?
```
Email + password only                   → allauth + custom user model
Social login (Google, GitHub, Apple)    → django-allauth socialaccount providers
MFA / TOTP / WebAuthn                   → allauth.mfa installed app
SPA / mobile (headless API)             → allauth.headless + JWT bridge view
Token-based API auth                    → SimpleJWT (standalone or with allauth bridge)
Service-to-service                      → API key (custom HasAPIKey permission)
```

### What API documentation tool?
```
Project has any DRF view or ViewSet       → drf-spectacular (always)
New ViewSet created                       → @extend_schema_view on class
New @action method                        → @extend_schema (always)
APIView (not ViewSet)                     → @extend_schema per HTTP method
FBV with @api_view                        → @extend_schema above @api_view
SerializerMethodField                     → @extend_schema_field (always)
Nullable SerializerMethodField            → {"type": "...", "nullable": True}
Response shape ≠ request shape            → explicit request= and responses=
DELETE endpoint                           → responses={204: None, ...}
Binary file response                      → responses={(200, "application/pdf"): ...}
One-off response shape                    → inline_serializer (unique name convention)
Health check / internal view              → @extend_schema(exclude=True)
Production schema access                  → SERVE_PERMISSIONS=["IsAuthenticated"]
JWT auth in Swagger                       → SimpleJWTAuthenticationExtension in AppConfig.ready()
Large project (30+ endpoints)             → cache_page on schema URL
```

### What search technology?
```
< 100k docs, simple contains query      → Django ORM (.filter(name__icontains=q))
< 100k docs, PostgreSQL full-text       → SearchVector + SearchQuery
> 100k docs OR relevance scoring        → Elasticsearch (django-elasticsearch skill)
Autocomplete / search-as-you-type       → Elasticsearch edge-n-gram analyzer
Faceted filtering (price, category)     → Elasticsearch aggregations
Analytics / log aggregation             → Elasticsearch + Kibana
```

### S3 URL Security Decision
```
Public images (product photos)          → No signing, CloudFront CDN, long cache TTL
User avatars (public profile)           → No signing, CDN URL
User documents (invoices, contracts)    → Signed URL, 15-min TTL
Medical / legal / financial files       → Signed URL, 5-min TTL, audit logged
Exports / reports for download          → Signed URL, 1-hour TTL, single-use
```
