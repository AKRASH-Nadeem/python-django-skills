---
trigger: always_on
---

# Engineering Philosophy — Dynamic Solutions & App State

> These rules are ALWAYS ACTIVE. They govern how the agent
> builds solutions and maintains a living record of the application.

---

## Reasoning & Invalidation Log

```
DECISION 1: APP_STATE.md as mandatory living document
  Initial: Developers maintain context manually
  Invalidated: AI agents have no persistent memory between sessions.
               Without a living snapshot, the agent repeats the same
               infrastructure assumptions, adds Redis without noticing it's
               already there, or misses that the dev env has no S3.
  Solution: APP_STATE.md is non-negotiable — created on session start,
            maintained on every structural change.

DECISION 2: .env.example as a first-class artifact
  Initial: Developers handle .env.example manually
  Invalidated: Agent adds env vars without updating .env.example — the
               next developer cloning the repo gets a silent startup crash.
               Django's ImproperlyConfigured throws, but .env.example is
               missing the entry, giving no hint what to set.
  Solution: Every new env var requires a .env.example entry with a
            comment explaining what it is and what format it expects.
            This is explicitly mandated in Rule 1.

DECISION 3: Size discipline on APP_STATE.md
  Initial: APP_STATE.md grows indefinitely as features are added
  Invalidated: A 500-line APP_STATE.md defeats the purpose — a new
               developer can't quickly understand the system. It becomes
               a log instead of a snapshot.
  Solution: APP_STATE.md is a SNAPSHOT. Replace outdated entries, never
            append. Target: under 100 lines. Above 150 = consolidate first.

DECISION 4: No hardcoded defaults for ANY API key, including test/sandbox keys
  Initial: sk_test_xxx looks safe as a default — it's just a test key
  Invalidated: Test keys can accidentally reach production if someone
               forgets to set the env var. They also get committed to
               version control. The rule must be absolute: any key
               string — test or live — has no hardcoded default.
  Solution: Rule 1 explicitly covers this: 'This includes test/sandbox API
            keys (e.g. sk_test_xxx). Use default=None with a startup check.'

DECISION 5: Run Instructions must be populated on APP_STATE.md creation
  Initial: Run Instructions section defined in schema but timing unclear
  Invalidated: A new project creates APP_STATE.md but leaves Run Instructions
               empty — a new developer joining the project can't run the app.
  Solution: When APP_STATE.md is first created, the agent MUST immediately
            populate Run Instructions with the current known setup steps.
            Partial is acceptable; missing is not.
```

---

## Core Philosophy

A senior developer never hard-wires a value that legitimately changes
across environments. Every solution the agent builds must survive being
deployed in development, staging, and production without code changes —
only configuration changes.

The agent also maintains a living document (`APP_STATE.md`) that any
developer or another agent session can read to understand the current
state of the application without reading all the code.

---

## Rule 1 — The Configurability Test

Before writing any value into code, the agent asks:
"Does this value change between environments or over time?"

If YES → it belongs in an environment variable or settings layer.
If NO  → it may be a constant in code (but still name it meaningfully).

**Every new env var must also be added to `.env.example`** with a safe
placeholder value and a comment describing what it is for. This is how
the next developer knows what to configure when cloning the project.

Values that ALWAYS require configuration:
- API keys, secrets, signing keys
- Database URLs, hostnames, ports
- Third-party service credentials
- Feature flags that differ by environment
- Email backends (console/dev vs real/prod)
- Storage backends (local vs S3)
- Cache backends (locmem vs Redis)
- Debug modes, log levels
- Allowed hosts, CORS origins
- Worker counts, timeout values
- Any URL or domain name

Pattern: Use `django-environ` or `python-decouple`.
Never use `os.environ.get("KEY", "hardcoded_fallback_for_prod")`.
Sensitive values (secrets, keys) have NO default — missing = crash-fast.
**This includes test/sandbox API keys (e.g. `sk_test_xxx`). They can end up
in production or version control. Use `default=None` with a startup check:
`if not SECRET_KEY: raise ImproperlyConfigured("SECRET_KEY must be set")`**
**`default=""` is also not acceptable for secrets.** An empty-string default
bypasses crash-fast behaviour and lets the app start with a silently missing
secret. Use no default at all (django-environ raises automatically) or
`default=None` with an explicit `ImproperlyConfigured` check.
Safe defaults only for non-sensitive operational values (e.g. timeouts, counts).

---

## Rule 2 — Environment Matrix Thinking

When building any feature, the agent explicitly thinks through
how it behaves in each environment before writing code:

| Concern          | Dev           | Staging       | Production     |
|------------------|---------------|---------------|----------------|
| What changes?    | (fill in)     | (fill in)     | (fill in)      |
| What stays same? | (fill in)     | (fill in)     | (fill in)      |

If the behavior is identical in all environments → one implementation.
If it differs → env-var controlled, never branched by `if DEBUG:` in
business logic. `DEBUG` is for Django internals only.

Correct env-based branching lives only in:
- `settings/base.py`, `settings/development.py`, `settings/production.py`
- `INSTALLED_APPS` additions for dev-only tools (silk, debug_toolbar)
- `MIDDLEWARE` additions for dev-only profiling

---

## Rule 3 — Research Before Hardcoding

When choosing a library, pattern, or configuration value, the agent
must not rely on training-data knowledge for anything version-sensitive.

Trigger research (Context7 or web) when:
- Library API or settings names may have changed
- A "recommended way" exists that the agent isn't certain is current
- The solution involves a library version released after mid-2025
- The agent is choosing between two approaches and isn't certain
  which is the current community standard

Research target: find the latest stable, recommended approach.
Research output: a concrete, current implementation — not a survey.

---

## Rule 4 — APP_STATE.md — The Living App Document

The agent creates and maintains `APP_STATE.md` in the project root.
It is a snapshot of the application's current state — not a changelog.

### When to update APP_STATE.md:

**On first creation:** Agent MUST populate Tech Stack, Environment Matrix,
Run Instructions (with current known steps), and External Services immediately.
Partial is fine — mark unknown sections with `<!-- TODO -->`.
**Run Instructions MUST be written, even if partial.** A new developer must
be able to run the app from this document.

Mandatory update triggers (agent MUST update after these):
- New Django app added or removed
- New third-party package added or removed
- Authentication method changed or added
- Database engine or major version changed
- New external service integrated (Elasticsearch, S3, Stripe, etc.)
- New environment variable added (agent adds it to the matrix AND to `.env.example`)
- Deployment method changed (Docker, Nginx config, workers changed)
- A setting that differs between environments is added or changed

DO NOT update for: minor feature additions, bug fixes, view changes,
serializer changes, or anything that doesn't change how the app runs.

### Size discipline:
APP_STATE.md is a SNAPSHOT, not a log. When updating, replace outdated
entries — do not append. **Target: under 100 lines. When an update would
push the file past 100 lines, consolidate first.** If the file somehow
reaches 150 lines, consolidation is mandatory before any further additions.

Consolidation means: merge duplicate entries, shorten descriptions to one
line, remove sections for features that were deleted, collapse verbose
env-var lists into a `.env.example` reference.

### What APP_STATE.md contains:

```
# APP_STATE.md
_Last updated: [date] | Agent: always read this before making
architectural decisions._

## Tech Stack
<!-- Single-line per item. Query Context7 for current stable versions. -->
- Python: [x.x]
- Django: [x.x]
- Database: [PostgreSQL x / SQLite]
- Cache: [Redis x / LocMem / None]
- Task Queue: [Celery + Redis / None]
- Search: [Elasticsearch x / ORM FTS / None]
- Auth: [JWT (SimpleJWT) / Session / Allauth + JWT]
- Storage: [S3 (django-storages) / Local]
- API Docs: [drf-spectacular / None]
- ASGI Server: [Uvicorn / Gunicorn+sync / Daphne]

## Environment Matrix
| Feature        | Development      | Staging     | Production  |
|----------------|------------------|-------------|-------------|
| DEBUG          | True             | False       | False       |
| Storage        | Local filesystem | S3          | S3          |
| Email          | Console          | Mailhog/SES | SES         |
| Cache          | LocMem           | Redis       | Redis       |
| Celery broker  | Redis (local)    | Redis       | Redis       |
| Schema docs    | AllowAny         | Auth-gated  | Auth-gated  |
<!-- Agent: add rows as new env-dependent features are added -->

## Installed Apps (Local)
<!-- List only LOCAL_APPS — not Django or third-party -->

## External Services & Required Env Vars
<!-- Format: SERVICE: VAR1, VAR2 -->
<!-- Agent: add a row when a new external service is integrated -->

## Known Constraints
<!-- Resource limits, known bottlenecks, explicit TODOs -->
<!-- Agent: add here when a decision was made due to a known limit -->

## Run Instructions
### Development
<!-- Minimal steps to get the app running locally -->
### Production
<!-- Key differences from development -->
```

### .env.example discipline:

Every env var added by the agent must appear in `.env.example`:

```bash
# Required — Django secret key (generate with: python -c "import secrets; print(secrets.token_urlsafe(50))")
SECRET_KEY=your-secret-key-here

# Required — Database URL
DATABASE_URL=postgres://user:password@localhost:5432/dbname

# Required — Stripe API key (production key from stripe.com dashboard)
# NEVER use a hardcoded test key as default — test keys can reach production
STRIPE_SECRET_KEY=sk_test_your_key_here

# Optional — Redis URL (leave unset to use LocMem cache in dev)
# REDIS_URL=redis://localhost:6379/0
```

### How the agent uses APP_STATE.md:

1. READ IT FIRST before any architectural decision.
   If APP_STATE.md says "Cache: None" and the agent is about to
   add Redis-dependent caching, it notes this is a new dependency
   and adds it to the matrix.

2. If code contradicts APP_STATE.md, the agent flags the discrepancy
   to the developer before proceeding. It does NOT silently assume
   either is correct.

3. The agent writes only what IS TRUE NOW, not what it plans.
   Every entry in APP_STATE.md must be verifiable in the codebase.

---

## Rule 5 — No Silent Assumptions

If the agent doesn't know something that materially affects its solution,
it states the assumption explicitly at the top of its response:

Format: "ASSUMING: [assumption]. If this is wrong, [consequence]."

It never silently assumes scale, team size, infrastructure, or
whether the developer has Redis/S3/Elasticsearch available.
If these are not in APP_STATE.md and not visible in the code, it asks.
