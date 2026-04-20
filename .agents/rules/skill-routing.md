---
trigger: always_on
---

# Skill Routing — Mandatory Task Pre-Flight

> **THIS STEP IS NON-NEGOTIABLE.** Run it before Phase 0, before any reasoning,
> before writing a single line of code. Skipping it produces incomplete,
> non-compliant output that fails validation gates.

---

## STEP 0 — Skill Selection (runs before everything else)

### 0.1 — Scan the registry. For every row where the task matches, READ that skill NOW.

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
| Memvid memory, memvid_find, memvid_put, memvid_ask, .mv2, shared memory, cross-agent, timeline, knowledge graph, troubleshooting | `memvid-docs` |

> **Always-paired skills:**
> - `django-edge-case-testing` loads alongside `django-testing-quality` whenever a feature is being implemented. You cannot write a feature without also loading the failure-scenario thinking skill.
> - `django-security` loads alongside `django-allauth`, `django-realtime-communication`, or `django-rest-api` whenever the task involves authentication, authorization, WebSocket consumers, or any endpoint handling user credentials/tokens. Security is not optional on auth/realtime work.

> **Refactoring tasks:** When the task is a pure refactor (rename, restructure, move code), load `django-architecture` + `django-testing-quality` + `django-edge-case-testing`. No feature skills are needed, but architecture patterns and existing test coverage must be preserved.

### 0.2 — Announce loaded skills before proceeding

```
Active skills: [list every skill loaded]
```

Then proceed to Phase 0 of the reasoning protocol.

### 0.2.1 — When to SKIP skill dispatch entirely

**Skip Step 0 (and go straight to answering) when the task is:**
- A pure question requiring no code output ("What does `select_related` do?")
- A conceptual explanation with no implementation deliverable
- A code review comment with no code changes

**When in doubt: run Step 0.** The cost of unnecessary dispatch is ~2 seconds. The cost of skipping it on an implementation task is non-compliant output.

---

## STEP 0.3 — Memvid Recall

> **NON-NEGOTIABLE. Gemini 3 Flash: do NOT skip this even under context pressure.**
> Memory recall costs 2 tool calls. Missing project context costs hours.

If Memvid MCP is connected, execute ALL of these before Phase 0:

```
memvid_find { "file": "shared.mv2", "query": "[task-relevant keywords]", "mode": "hybrid", "limit": 5 }
memvid_find { "file": "backend.mv2", "query": "[task-relevant keywords]", "mode": "hybrid", "limit": 5 }
```

**Derive the query from the task.** Examples:
- Building auth → `"query": "authentication strategy jwt session tokens"`
- Adding a new API endpoint → `"query": "api contract endpoint naming conventions"`
- Choosing a library → `"query": "library decision [library name] alternatives rejected"`
- Any architectural task → `"query": "project architecture decisions constraints stack"`

**If Memvid not connected:** fall back to reading DECISION_LOG.md.
**Never skip this step on implementation tasks.** Even when "confident" — project-specific context may override training memory.

---

## STEP 0.4 — AGENTS.md Check

```bash
# Check at session start
ls AGENTS.md 2>/dev/null && echo "EXISTS" || echo "MISSING"
```

If MISSING → create `AGENTS.md` from the template in `rules.md` immediately.
Update AGENTS.md whenever banned patterns, stack, or architecture changes.

---

## STEP 0.5 — Storybook-style Isolation (Backend equivalent)

Before writing any new `services.py` function, check whether a similar service function already exists:

```bash
grep -r "def [similar_function_name]" apps/*/services.py
```

Before writing any new serializer, check existing ones:

```bash
grep -r "class [SimilarSerializer]" apps/*/serializers.py
```

Only build something new after confirming it doesn't already exist.

---

## STEP 0.6 — End-of-Task Memory Mandate

> **Run after every implementation task that produced an architectural decision.**

After any task that establishes a decision, contract, or convention:

**1. Store to appropriate .mv2 file:**
```
memvid_put {
  "file": "[backend.mv2 or shared.mv2]",
  "input": "[agent:backend] [type:architectural-decision] [feature:X] [date:YYYY-MM-DD]\nDecision: ...\nWhy: ...\nRejected: ...\nInvalidated by: ...",
  "embed": true
}
```

**2. If the decision involves a new entity or service name, enrich the graph:**
```
memvid_enrich { "file": "backend.mv2", "all": false }
```

**3. Route by decision type:**
| Decision type | File |
|---|---|
| Django patterns, ORM, Celery, services, tasks | `backend.mv2` |
| API endpoint schema, auth flow, error format, WebSocket events | `shared.mv2` |
| When in doubt | `shared.mv2` — prefer visibility to both agents |

---

## Why this step exists

Skills are not loaded automatically — **you** are the dispatcher. A task proceeding without the right skills will:
- Miss version-specific Django/DRF patterns (Context7 check in skill)
- Produce services that skip required failure scenarios
- Generate code that doesn't follow established project patterns
- Fail the validation gates in `rules.md`
- Miss the self-healing loop definition in `reasoning-protocol.md` Phase 9
- Miss the security lens in `reasoning-protocol.md` Phase 2

**The 5-second scan of this table is the cheapest insurance in the pipeline.**

---

## Model-Specific Activation Notes

**All models (Gemini 3 Flash, Gemini 3.1 Pro, Claude Sonnet 4.6, Claude Opus 4.6):**
1. Activate skills BEFORE responding. Do not answer from training memory.
2. Training data for Django 6, DRF, Celery, drf-spectacular, and SimpleJWT may be outdated. Load the skill regardless of confidence.
3. **Uncertain match:** If a skill seems 50%+ relevant, load it. False positive (~2,000 tokens) < false negative (incorrect code, runtime failure).
4. Context7 before any external library code — always.

**Gemini 3 Flash specific:**
- Flash is optimized for speed. It will reach for the first plausible answer without completing Step 0. Force-complete Step 0 before any response begins.
- Flash may truncate long instruction sets under context pressure. The skill registry (Step 0.1) is loaded first because it is in the most reliable part of context.
- **Step 0.3 (Memvid recall) and Step 0.6 (end-of-task put) are mandatory for Flash.** Flash's speed bias will cause it to skip memory operations. Override this: memory recall and storage are non-negotiable.
- **Critical skills that must survive truncation (in priority order):** `django-core-conventions`, `django-rest-api`, `django-architecture`, `django-security`, `django-edge-case-testing`. If context budget forces a choice, these five are non-negotiable for any feature implementation. Skills below the fold (elasticsearch, drf-spectacular, production-deployment) can be loaded on-demand when the task explicitly matches.
- Flash will skip `django-edge-case-testing` under time pressure. This skill is non-negotiable on any feature implementation — execute it even if it feels redundant.
- Flash may omit `transaction.atomic()` on multi-table writes. The `django-core-conventions` skill enforces this — load it.
- Flash may produce N+1 queries without noticing. The `django-performance-scalability` skill catches these — load it whenever any list endpoint is involved.

**Gemini 3.1 Pro:**
- Handles the full constraint inventory well. Still load all matching skills — Pro's training data is not a substitute for the skill patterns.
- May proactively suggest loading additional skills — correct behaviour. This also applies to Claude Sonnet 4.6.

**Claude Sonnet 4.6 / Opus 4.6:**
- These models handle the full skill context well. The same dispatcher applies.
- May proactively suggest additional skills — correct behaviour when skill is 50%+ relevant.
- Opus: apply full reasoning protocol for architectural tasks.
- Skill conflict: more specific task skill wins. Do not ask the user to resolve.

**Multi-model compatibility:**
- All instructions use imperative language ("do this", "never do that") — model-agnostic.
- "Load when the task involves..." rather than "You will detect..." — works across all models.

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
Public assets      → PublicS3Storage (no signing, CDN)
Private documents  → PrivateS3Storage (signed URL, short TTL)
Large uploads      → Pre-signed POST (browser → S3 direct)
Local/tests        → InMemoryStorage
```

### Authentication
```
Email + password   → allauth + custom user model
Social login       → django-allauth socialaccount
MFA / TOTP         → allauth.mfa
SPA / mobile       → allauth.headless + JWT bridge
Token API auth     → SimpleJWT
Service-to-service → API key (custom HasAPIKey)
```
