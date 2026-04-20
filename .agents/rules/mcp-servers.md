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
Step 1a — Source validation:
  Context7             → before ANY external library code
  pip-tools / pyproject → before ANY new library is proposed

Step 1b — Memory (run BEFORE implementation, AFTER source validation):
  Memvid MCP           → recall at session start; store after every decision
  DECISION_LOG.md      → fallback if Memvid MCP unavailable

Step 2 — Live docs:
  Fetch MCP            → live changelog, migration guides, niche docs

Step 3 — Frontend-backend bridge:
  Memvid MCP           → shared.mv2 cross-agent context
```

### When to use MCP vs skills vs training memory

| Need | Source |
|------|--------|
| Django built-in APIs (ORM basics, views, URLs) | Training memory — stable, well-known patterns |
| External library method signatures | Context7 (always) |
| Django 6-specific settings names | Context7 (always) |
| Past decisions about auth/cache/queue/architecture | Memvid MCP → DECISION_LOG.md |
| Live changelogs / niche docs | Fetch MCP |
| Architectural patterns | Skills |
| Version-specific library code | Context7 with version-specific query |

---

## 1. Context7 — Live Documentation

**Purpose**: Fetches current, version-specific docs. Eliminates documentation hallucinations.

**Call before:** Any DRF serializer/viewset; any Celery task setup; any drf-spectacular decorator;
any SimpleJWT config; any django-allauth setup; any django-storages config; any Elasticsearch DSL;
any package not part of Python's standard library.

**NOT required for:** Python language features, stable Django patterns known to not change.

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

> **Load `skills/memvid-docs/SKILL.md`** when troubleshooting Memvid, configuring files,
> or needing the full 40-tool reference. For day-to-day use, this section is sufficient.

### Memory file topology:
```
backend.mv2   → Backend agent's PRIVATE memory (model conventions, Django patterns, service structure)
frontend.mv2  → Frontend agent's PRIVATE memory (component patterns, UI decisions)
shared.mv2    → SHARED cross-agent memory (API contracts, auth, error formats, integration points)
```

### Session Start — Full Init Sequence

Run at the start of every session (also covered in `rules.md` Step 1.5):

```
# 1. Verify files exist (create if missing)
memvid_stats { "file": "shared.mv2" }   → error? → memvid_create { "file": "shared.mv2" }
memvid_stats { "file": "backend.mv2" }  → error? → memvid_create { "file": "backend.mv2" }

# 2. Start session
memvid_session { "file": "backend.mv2", "start": "be-[YYYYMMDD-HH]" }

# 3. Recall context
memvid_find { "file": "shared.mv2", "query": "project architecture api contracts auth stack", "mode": "hybrid", "limit": 5 }
memvid_find { "file": "backend.mv2", "query": "recent decisions constraints conventions patterns", "mode": "hybrid", "limit": 5 }

# 4. First run only — migrate DECISION_LOG.md
memvid_put_many { "file": "backend.mv2", "input": "DECISION_LOG.md", "embed": true }
```

### Storing a memory (memvid_put):

```
memvid_put {
  "file": "shared.mv2",
  "input": "[agent:backend] [type:architectural-decision] [feature:auth] [date:2026-04-19]\nDecision: Chose django-knox for token authentication.\nWhy: Per-token revocation, Chrome extension compatibility, stateless.\nRejected: SimpleJWT (no per-token revocation), django-allauth (session-based).\nInvalidated by: Social OAuth required; server-rendered frontend adopted.",
  "embed": true
}
```

**Always include:**
- `"embed": true` — enables semantic search via nomic embeddings
- Tag prefix: `[agent:backend]`
- Type tag: `[type:architectural-decision]`, `[type:api-contract]`, `[type:convention]`, `[type:constraint]`
- Date tag: `[date:YYYY-MM-DD]`

### Recalling memories (memvid_find):
```
# Before any architectural decision
memvid_find { "file": "shared.mv2", "query": "authentication strategy", "mode": "hybrid", "limit": 5 }
memvid_find { "file": "backend.mv2", "query": "project architecture decisions", "mode": "hybrid", "limit": 5 }

# Cross-agent context
memvid_find { "file": "shared.mv2", "query": "frontend API expectations and integration patterns", "mode": "hybrid", "limit": 5 }
```

### After storing — enrich knowledge graph:
```
memvid_enrich { "file": "backend.mv2", "all": false }
```
Run after any `memvid_put` that introduces a new service name, library, or architectural entity.

### Session end:
```
memvid_session { "file": "backend.mv2", "stop": true }
```

### Resolution protocol:
```
1. memvid_find on shared.mv2 + backend.mv2 → results found → use them
2. Returns empty → no prior context. Build fresh from DECISION_LOG.md + APP_STATE.md
3. Memvid unavailable → fall back to DECISION_LOG.md + APP_STATE.md
   If cross-agent memory needed: write to SHARED_CONTRACTS.md
4. Neither DECISION_LOG.md nor .mv2 files exist →
   Ask: A) Setup Memvid  B) Create DECISION_LOG.md  C) Start fresh
5. Never skip recall. Project context overrides training memory.
```

### Conflict resolution for shared memory:
When memvid_find returns contradictory memories:
```
⚠️ MEMORY CONFLICT: shared.mv2 contains two contradictory entries:
- [agent:frontend, Date]: "[statement 1]"
- [agent:backend, Date]: "[statement 2]"
Which should I follow?
```
Do not silently pick one. After resolution: `memvid_put` the winner, `memvid_delete` the loser.

### Cross-agent fallback (Memvid unavailable):
1. Check for `SHARED_CONTRACTS.md` in project root.
2. If missing, create it with the contract you're defining.
3. Both agents read `SHARED_CONTRACTS.md` at session start.
4. State: "Memvid unavailable. Writing to SHARED_CONTRACTS.md."

### Memory hygiene:
- Decision reversed → `memvid_update` the original frame. Never append a contradiction.
- Stale memory (references deleted library/file) → `memvid_update` to replace it.
- Periodic consolidation: `memvid_ask { "file": "backend.mv2", "question": "Summarize all active architectural decisions." }`
- View evolution: `memvid_timeline { "file": "backend.mv2" }`

### What to store vs skip:

| Store ✅ | Skip ❌ |
|---------|---------|
| Library choices + rejections with reasoning | Bug fixes following existing patterns |
| API contracts (endpoint schema, error format) | New model fields following conventions |
| Auth/cache/queue/storage decisions | Style/copy changes |
| Developer constraints ("no Redux", "max 2 workers") | Test additions |
| Architecture patterns (service layer, app structure) | Anything that doesn't change reasoning |

### Core tool reference:
```
Lifecycle: memvid_create, memvid_open, memvid_stats, memvid_verify, memvid_doctor
Content:   memvid_put, memvid_put_many, memvid_view, memvid_update, memvid_delete, memvid_correct, memvid_api_fetch
Search:    memvid_find, memvid_vec_search, memvid_ask, memvid_timeline, memvid_when
Graph:     memvid_enrich, memvid_memories, memvid_state, memvid_facts, memvid_follow, memvid_who
Session:   memvid_session, memvid_binding, memvid_status, memvid_sketch, memvid_nudge
Analysis:  memvid_audit, memvid_debug_segment, memvid_export, memvid_tables, memvid_schema, memvid_models
Encrypt:   memvid_lock, memvid_unlock
Utility:   memvid_process_queue, memvid_verify_single_file, memvid_config, memvid_version
```

---

## 3. Fetch MCP — Live URL Content

**Activate when:**
- User references a specific URL
- Context7 returns no results for a library
- Checking a Django/DRF/Celery CHANGELOG before migration
- Verifying a CVE or security advisory

---

## 4. MCP Tool Safety & Circuit Breakers

### 4.1 — Tool name validation
Before calling any MCP tool, verify the tool name exists. Never guess from training memory.

### 4.2 — Per-tool retry limit
Each tool: **maximum 2 retries** per task. After 2 failures:
1. State: *"[Tool] has failed twice. Falling back to [fallback]."*
2. Use fallback immediately. Never attempt a third call.

### 4.3 — Retryable vs non-retryable

| Error type | Action |
|---|---|
| Network timeout, 503, rate limit (429) | Retry once with backoff (2s, then 4s) |
| No results, wrong tool name, auth failure | Do NOT retry — fall back immediately |

**Key:** "No results" from Context7 is non-retryable. Reformulate or use Fetch MCP.

### 4.4 — Context7 query reformulation
No results on first call → simplify query (remove version specifiers) → if still none → Fetch MCP official docs URL → if Fetch also fails → note limitation, use training memory with caveat.

---

## MCP Failure Handling

| Server unavailable | Fallback |
|---|---|
| Context7 | Fetch MCP → raw docs URL → note limitation |
| Memvid | Read DECISION_LOG.md + APP_STATE.md; cross-agent: SHARED_CONTRACTS.md |
| Fetch | Note limitation, use training memory with explicit caveat |

Never silently fall back to training memory when MCP was required.
