---
trigger: always_on
---

# Skill Routing — Load the Right Skill Before Writing Code

> This file replaces `skills/index.md`, which is not loaded by Antigravity.
> Skills are discovered via semantic triggering on their SKILL.md `description` field.
> This routing table is in `rules/` so it is always in context.

---

## Task → Skill Map

Before writing any implementation code, identify the task category and load the matching skill(s).
Multiple skills may apply — load all that match.

| Task Type | Load Skill |
|---|---|
| Python types, async, decorators, dataclasses, patterns | `python-senior-core` |
| Models, ORM, managers, migrations, signals, admin | `django-core-conventions` |
| Architecture, service/selector pattern, app structure | `django-architecture` |
| REST API, DRF, serializers, viewsets, auth, pagination | `django-rest-api` |
| WebSocket, SSE, webhooks, real-time communication | `django-realtime-communication` |
| Security, CORS, headers, auth hardening, secrets | `django-security` |
| Caching, query optimisation, DB indexing, scaling | `django-performance-scalability` |
| Celery, background tasks, scheduling, job queues | `django-async-tasks` |
| Tests, factories, coverage, linting, type checking | `django-testing-quality` |
| Edge cases, failure scenarios, adversarial testing | `django-edge-case-testing` |
| Docker, CI/CD, Gunicorn, Nginx, production config | `django-production-deployment` |
| File uploads, S3/media/static storage, signed URLs | `django-storages-s3` |
| Social login, OAuth2, MFA, allauth | `django-allauth` |
| Full-text search, facets, autocomplete, Elasticsearch | `django-elasticsearch` |
| OpenAPI docs, @extend_schema, Swagger UI | `django-drf-spectacular` |

---

## Context7 Library IDs

| Library | Context7 ID |
|---|---|
| Django official docs | `/websites/djangoproject_en_6_0` |
| Django source | `/django/django` |
| Django REST Framework | `/websites/django-rest-framework` |
| SimpleJWT | `/jazzband/djangorestframework-simplejwt` |
| Celery | `/celery/celery` |
| Django Celery Beat | `/celery/django-celery-beat` |
| Django Allauth | `/pennersr/django-allauth` |
| Elasticsearch 8.x | `/websites/elastic_co_guide_en_elasticsearch_reference_8_19` |
| drf-spectacular (GitHub) | `/tfranzel/drf-spectacular` |
| drf-spectacular (docs) | `/websites/drf-spectacular_readthedocs_io_en` |

Query format: `"Use Context7 to look up [ID] [specific topic]"`

---

## Quick Decision Tables

### Communication method
```
One request → one response         → REST API
Bidirectional, persistent          → WebSocket (Django Channels)
Server pushes, client reads        → SSE (async view)
External system notifies you       → Webhook (inbound POST + HMAC)
You notify external system         → Webhook (outbound via Celery)
Background work                    → Celery task
Scheduled work                     → Celery Beat
```

### Architecture pattern
```
New feature                        → New Django app
Business logic                     → services.py
Query logic                        → selectors.py
HTTP handling                      → views.py (thin)
Input/output shaping               → serializers.py
Cross-app notification             → Django signals
Cross-app async work               → Celery tasks
```

### Search technology
```
< 100k docs, simple                → Django ORM (icontains)
< 100k docs, PostgreSQL            → SearchVector + SearchQuery
> 100k docs OR relevance scoring   → Elasticsearch
Autocomplete                       → Elasticsearch edge-n-gram
Faceted filtering                  → Elasticsearch aggregations
```

### Storage
```
Public assets                      → PublicS3Storage (no signing, CDN)
Private documents                  → PrivateS3Storage (signed URL, short TTL)
Large uploads (video, datasets)    → Pre-signed POST (browser → S3 direct)
Local dev / CI / tests             → InMemoryStorage
```

### Authentication
```
Email + password only              → allauth + custom user model
Social login                       → django-allauth socialaccount
MFA / TOTP / WebAuthn              → allauth.mfa
SPA / mobile (headless)            → allauth.headless + JWT bridge
Token API auth                     → SimpleJWT
Service-to-service                 → API key (custom HasAPIKey)
```
