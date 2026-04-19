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
  Memvid MCP           → recall at session start via memvid_find; store after decisions via memvid_put
  DECISION_LOG.md      → fallback if Memvid MCP unavailable

Step 3 — Live docs:
  Fetch MCP            → live changelog, migration guides, niche docs Context7 doesn't cover

Step 4 — Frontend-backend bridge (shared memory):
  Memvid MCP           → shared.mv2 with frontend agent for cross-agent project context
```

### When to use MCP vs skills vs training memory

| Need | Source |
|------|--------|
| Django built-in APIs (ORM basics, views, URLs) | Training memory — for stable, well-known patterns |
| External library method signatures | Context7 (always) |
| Django 6-specific settings names | Context7 (always — settings change between majors) |
| Past decisions about auth/cache/queue | Memvid MCP → DECISION_LOG.md |
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

## 2. Memvid MCP — Persistent Memory

**Purpose**: Persistent, cross-session memory for architectural decisions, team conventions,
and project preferences. Uses portable `.mv2` files — no databases, no infrastructure.
Backend and frontend agents share a `shared.mv2` file for cross-agent context awareness.

**When to load the `memvid-docs` skill:** Load `skills/memvid-docs/SKILL.md`
when troubleshooting Memvid, configuring memory files, or needing the full
40-tool reference beyond what this section covers.

### Resolution protocol:
```
1. Attempt memvid_find on shared.mv2 + backend.mv2 → if results found, use them.
2. memvid_find returns empty/no results → No relevant prior context
   exists for this query. Proceed with DECISION_LOG.md + APP_STATE.md
   as primary context, then build fresh.
3. Memvid MCP is unavailable (connection error, not installed) →
   Fall back to DECISION_LOG.md + APP_STATE.md. If cross-agent memory
   is needed and Memvid is down, note in HANDOFF.md:
   "Memvid was unavailable — cross-agent decisions not synced."
4. If neither DECISION_LOG.md nor any .mv2 files exist →
   Ask user: A) Setup Memvid  B) Create DECISION_LOG.md  C) Start fresh
5. Never skip the recall step. Even if you are confident in your
   training data, memvid_find may surface project-specific context.
```

### Core operations:
```
memvid_put   → after any architectural decision, convention, or contract established
memvid_find  → at session start, before any architectural decision (hybrid search)
memvid_ask   → for complex queries needing RAG synthesis across multiple memories
```

### Memory file topology:
```
backend.mv2   → Backend agent's PRIVATE memory (model conventions, Django patterns, service structure)
frontend.mv2  → Frontend agent's PRIVATE memory (component patterns, UI decisions)
shared.mv2    → SHARED cross-agent memory (API contracts, auth, error formats, integration points)
```

### Storing a memory (memvid_put):
```
memvid_put {
  "file": "shared.mv2",
  "input": "[agent:backend] [type:architectural-decision] [feature:auth]\nDecision: Chose django-knox for token authentication.\nWhy: Per-token revocation, Chrome extension compatibility, stateless.\nRejected: SimpleJWT (no per-token revocation), django-allauth (session-based).\nInvalidated by: Social OAuth required; server-rendered frontend adopted.",
  "embed": true
}
```

Always include:
- `"embed": true` — enables semantic search via nomic embeddings
- Tag prefix: `[agent:backend]` or `[agent:frontend]`
- Type tag: `[type:architectural-decision]`, `[type:api-contract]`, `[type:convention]`

### Recalling memories (memvid_find):
```
# Session start — recall relevant context
memvid_find { "file": "shared.mv2", "query": "django project architecture decisions", "mode": "hybrid", "limit": 5 }
memvid_find { "file": "shared.mv2", "query": "authentication strategy", "mode": "hybrid", "limit": 5 }
memvid_find { "file": "backend.mv2", "query": "infrastructure available (Redis, S3)", "mode": "hybrid", "limit": 5 }

# Cross-agent context (backend learning from frontend decisions)
memvid_find { "file": "shared.mv2", "query": "frontend API expectations and integration patterns", "mode": "hybrid", "limit": 5 }
```

### Conflict resolution for shared memory:

When memvid_find returns contradictory memories (e.g., frontend stored
"no rate limiting needed" but backend stored "rate limiting on all endpoints"):

1. **Surface the conflict explicitly** to the user:
   ```
   ⚠️ MEMORY CONFLICT: shared.mv2 contains two contradictory entries:
   - [agent:frontend, Date]: "[statement 1]"
   - [agent:backend, Date]: "[statement 2]"
   Which should I follow?
   ```
2. **Do not silently pick one.** Contradictions in shared memory are
   architecture-level issues that need human resolution.
3. After resolution, `memvid_put` the resolved decision and `memvid_delete`
   the contradicting entry so both agents converge.

### Cross-agent fallback (when Memvid is unavailable):

If Memvid is unavailable and the task requires cross-agent knowledge:
1. Check for a `SHARED_CONTRACTS.md` file in the project root.
2. If it doesn't exist, create it with the contract you're defining.
3. Both agents must read `SHARED_CONTRACTS.md` at session start.
4. This file is the manual fallback for Memvid put/find.
5. Explicitly state: "Memvid unavailable. Writing to SHARED_CONTRACTS.md
   so the other agent can pick this up."

### Memory hygiene:
- Use `memvid_ask` after multi-feature implementations to synthesize and
  consolidate memories. Remove superseded entries with `memvid_delete`.
- **Do not store implementation details** (exact code lines, file paths).
  Store architectural decisions and contracts only.
- **Stale memory detection:** If a recalled decision references a library
  version, file, or pattern that no longer exists in the codebase, treat
  it as stale. Note the staleness and proceed with current codebase state.
  After resolution, `memvid_update` the stale entry to replace it.
- Use `memvid_timeline` to review how decisions evolved over time.

### MCP tool names (exact — do not guess):
```
Lifecycle: memvid_create, memvid_open, memvid_stats, memvid_verify, memvid_doctor
Content:   memvid_put, memvid_put_many, memvid_view, memvid_update,
           memvid_delete, memvid_correct, memvid_api_fetch
Search:    memvid_find, memvid_vec_search, memvid_ask, memvid_timeline, memvid_when
Graph:     memvid_enrich, memvid_memories, memvid_state, memvid_facts,
           memvid_follow, memvid_who
Session:   memvid_session, memvid_binding, memvid_status, memvid_sketch, memvid_nudge
Analysis:  memvid_audit, memvid_debug_segment, memvid_export, memvid_tables,
           memvid_schema, memvid_models
Encrypt:   memvid_lock, memvid_unlock
Utility:   memvid_process_queue, memvid_verify_single_file, memvid_config, memvid_version
```

For full tool parameter schemas, load the `memvid-docs` skill.

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
| Memvid | Read DECISION_LOG.md + APP_STATE.md; if cross-agent: SHARED_CONTRACTS.md |
| Fetch | Note limitation, use training memory with explicit caveat |

Never silently fall back to training memory when MCP was required.
Never proceed if Context7 is unavailable and required for a version-sensitive library.
