---
trigger: always_on
---

> MCP MANDATE: Context7 must be called before generating any code that uses an
> EXTERNAL library or framework. Python standard library is exempt. Django built-ins
> are exempt for well-known stable APIs. For all other libraries: Context7 is the
> source of truth. Training memory is a draft. If they conflict, Context7 wins.

---

# MCP Servers — Backend Development Stack

---

## Priority Order (run in this sequence per task)

```
Step 1 — Source validation:
  Context7             → before ANY external library code
  pip-tools / pyproject → before ANY new library is proposed

Step 2 — Memory:
  Hindsight MCP        → recall at session start; retain after decisions; reflect after features
  DECISION_LOG.md      → fallback if Hindsight MCP unavailable

Step 3 — Live docs:
  Fetch MCP            → live changelog, migration guides, niche docs Context7 doesn't cover

Step 4 — Frontend-backend bridge (shared memory):
  Hindsight MCP        → shared bank with frontend agent for cross-agent project context
```

### When to use MCP vs skills vs training memory

| Need | Source |
|------|--------|
| Django built-in APIs (ORM basics, views, URLs) | Training memory — for stable, well-known patterns |
| External library method signatures | Context7 (always) |
| Django 6-specific settings names | Context7 (always — settings change between majors) |
| Past decisions about auth/cache/queue | Hindsight MCP → DECISION_LOG.md |
| Live changelogs / niche docs | Fetch MCP |
| Architectural patterns | Skills |
| Version-specific library code | Context7 with version-specific query |

---

## 1. Context7 — Live Documentation

**Purpose**: Fetches current, version-specific docs. Eliminates documentation hallucinations.

**Call before:** Any DRF serializer/viewset; any Celery task setup; any drf-spectacular decorator;
any SimpleJWT config; any django-allauth setup; any django-storages config; any Elasticsearch DSL;
any package not part of Python's standard library.

**NOT required for:** Python language features (list comprehensions, dataclasses, context managers,
well-known builtins), stable Django patterns known to not change (models.py basics, admin.py basics).

**Query patterns:**
```
Standard:         [library] [specific feature]
Version-specific: [library] v[INSTALLED_MAJOR] [specific feature]
Migration:        [library] v[N] to v[M] migration
Django-specific:  django 6 [specific feature or setting]
```

**Version resolution:** Check `requirements/base.txt` or `pyproject.toml` for installed version.
If installed major ≠ skill baseline in `versions.lock.md` → MUST use version-specific query.

---

## 2. Hindsight MCP — Persistent Memory

**Purpose**: Persistent, cross-session memory for architectural decisions, team conventions,
and project preferences. The backend agent and frontend agent SHARE THE SAME MEMORY BANK
for a given project — enabling cross-agent context awareness.

### Resolution protocol:
```
1. Attempt Hindsight recall → if results found, use them.
2. Hindsight returns empty/no results → No relevant prior context
   exists for this query. Proceed with DECISION_LOG.md + APP_STATE.md
   as primary context, then build fresh.
3. Hindsight MCP is unavailable (connection error, not installed) →
   Fall back to DECISION_LOG.md + APP_STATE.md. If cross-agent memory
   is needed and Hindsight is down, note in HANDOFF.md:
   "Hindsight was unavailable — cross-agent decisions not synced."
4. If neither DECISION_LOG.md nor APP_STATE.md exist →
   Ask user: A) Setup Hindsight  B) Create DECISION_LOG.md  C) Start fresh
5. Never skip the recall step. Even if you are confident in your
   training data, recall may surface project-specific context.
```

### Conflict resolution for shared memory:

When recall returns contradictory memories (e.g., frontend stored
"no rate limiting needed" but backend stored "rate limiting on all endpoints"):

1. **Surface the conflict explicitly** to the user:
   ```
   ⚠️ MEMORY CONFLICT: Hindsight contains two contradictory entries:
   - [Agent A, Date]: "[statement 1]"
   - [Agent B, Date]: "[statement 2]"
   Which should I follow?
   ```
2. **Do not silently pick one.** Contradictions in shared memory are
   architecture-level issues that need human resolution.
3. After resolution, `retain` the resolved decision so both agents converge.

### Cross-agent fallback (when Hindsight is unavailable):

If Hindsight is unavailable and the task requires cross-agent knowledge:
1. Check for a `SHARED_CONTRACTS.md` file in the project root.
2. If it doesn't exist, create it with the contract you're defining.
3. Both agents must read `SHARED_CONTRACTS.md` at session start.
4. This file is the manual fallback for Hindsight retain/recall.
5. Explicitly state: "Hindsight unavailable. Writing to SHARED_CONTRACTS.md
   so the other agent can pick this up."

**Consequence of separate banks:** If each agent uses a different bank-id,
all cross-agent memory is lost. Both agents MUST use the same bank-id for
the same project. If separate banks are discovered, flag immediately.

### Memory hygiene:
- **Reflect** after multi-feature implementations to consolidate memories
  and remove intermediate/superseded decisions.
- **Do not retain implementation details** (exact code lines, file paths).
  Retain architectural decisions and contracts only.
- **Stale memory detection:** If a recalled decision references a library
  version, file, or pattern that no longer exists in the codebase, treat
  it as stale. Note the staleness and proceed with current codebase state.
  After resolution, `retain` the updated decision to replace the stale one.

### Operations:
```
retain  → after any architectural decision, bug resolution, or convention established
recall  → at session start, before any architectural decision
reflect → after completing a multi-feature implementation or sprint
```

### Query examples:
```bash
# Session start — recall relevant context
hindsight recall [bank-id] "django project architecture decisions"
hindsight recall [bank-id] "authentication strategy"
hindsight recall [bank-id] "infrastructure available (Redis, S3, Elasticsearch)"

# After a decision
hindsight retain [bank-id] "Chose SimpleJWT over knox because team of 2 and no Chrome extension requirement. Knox per-token revocation not needed at this scale."

# Cross-agent context (backend learning from frontend decisions)
hindsight recall [bank-id] "frontend API expectations and integration patterns"
hindsight recall [bank-id] "what API endpoints has the frontend already integrated"
```

### Shared bank setup (frontend ↔ backend):
```
Both agents connect to the same bank-id for the same project.
Backend agent recalls what frontend agent has stored (API shape expectations,
which endpoints are integrated, auth token handling patterns).
Frontend agent recalls what backend agent has stored (API contracts, error formats,
rate limiting behaviours).
This eliminates the need to manually sync API contract documents between agents.
```

---

## 3. Fetch MCP — Live URL Content

**Activate when:**
- User references a specific URL
- Context7 returns no results for a library
- Checking a Django/DRF/Celery CHANGELOG before migration
- Verifying a CVE or security advisory

**Examples:**
```
https://docs.djangoproject.com/en/6.0/ref/settings/
https://www.django-rest-framework.org/api-guide/
https://docs.celeryq.dev/en/stable/userguide/tasks.html
```

---

## 4. MCP Tool Safety & Circuit Breakers

### 4.1 — Tool name validation

Before calling any MCP tool, verify the tool name exists. Never guess a tool name from training memory.

### 4.2 — Per-tool retry limit

Each tool has a **maximum of 2 retries** per task.

After 2 failed calls to the same tool:
1. State clearly: *"[Tool name] has failed twice. Falling back to [fallback]."*
2. Use the fallback immediately.
3. Never attempt a third call with the same parameters.

### 4.3 — Retryable vs non-retryable errors

| Error type | Action |
|---|---|
| Network timeout, server 503, rate limit (429) | Retry once with backoff (wait 2s, then 4s) |
| No results found, wrong tool name, auth failure | Do NOT retry — fall back immediately |

**Key rule:** "No results" from Context7 is non-retryable. It means the library/feature
is unknown to Context7 — not that the network failed. Reformulate or use Fetch MCP.

### 4.4 — Context7 query reformulation

If Context7 returns no results on the first call:
1. Simplify the query — remove version specifiers, try just `[library] [feature]`
2. If still no results → use Fetch MCP with the library's official docs URL
3. If Fetch MCP also fails → note the limitation to the user, use training memory with caveat

---

## MCP Failure Handling

| Server unavailable | Fallback |
|---|---|
| Context7 | Fetch MCP → raw docs URL → note limitation to user |
| Hindsight | Read DECISION_LOG.md + APP_STATE.md |
| Fetch | Note limitation, use training memory with explicit caveat |

Never silently fall back to training memory when MCP was required.
Never proceed if Context7 is unavailable and required for a version-sensitive library.
