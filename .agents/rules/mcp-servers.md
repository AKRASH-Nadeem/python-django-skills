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

**Full behaviour guide:** Load `skills/agent-memory/SKILL.md` — it governs when to write,
what to write, how to format, when NOT to write, CLI fallback, and shared memory authorship.
The commands below are the minimum session lifecycle. The skill is the authority.

### Memory file topology:
```
backend.mv2   → Backend agent's PRIMARY memory (decisions, patterns, constraints)
shared.mv2    → SHARED cross-agent memory (API contracts, auth flow, error format)
```

### Session Start — Mandatory Sequence

```
# 1. Verify files (create if missing)
memvid_stats { "file": "shared.mv2" }   → error? → memvid_create { "file": "shared.mv2" }
memvid_stats { "file": "backend.mv2" }  → error? → memvid_create { "file": "backend.mv2" }

# 2. Start session tracking
memvid_session { "file": "backend.mv2", "start": "be-[YYYYMMDD-HH]" }

# 3. Recall project context
memvid_find { "file": "shared.mv2", "query": "project architecture api contracts auth stack", "mode": "hybrid", "limit": 5 }
memvid_find { "file": "backend.mv2", "query": "recent decisions constraints conventions patterns", "mode": "hybrid", "limit": 5 }

# 4. First run only — migrate DECISION_LOG.md
memvid_put_many { "file": "backend.mv2", "input": "DECISION_LOG.md", "embed": true }
```

### Session End:
```
memvid_session { "file": "backend.mv2", "stop": true }
```

### Resolution Protocol:
```
1. memvid_find on shared.mv2 + backend.mv2 → results found → use them
2. Returns empty → no prior context. Build fresh, store decisions made.
3. Memvid unavailable → DECISION_LOG.md + APP_STATE.md fallback
   Cross-agent memory needed: write to SHARED_CONTRACTS.md
4. Neither exists → ask: A) Setup Memvid  B) Create DECISION_LOG.md  C) Start fresh
5. Never skip recall. Memory overrides training data.
```

### MCP Failure — CLI Fallback:
See `skills/agent-memory/SKILL.md §7` for full CLI commands.
```bash
memvid find backend.mv2 "[query]" --mode hybrid --limit 5
memvid put backend.mv2 "[agent:backend] [type:X] [feature:Y] [date:YYYY-MM-DD]\n..." --embed
```
If CLI also unavailable: `DECISION_LOG.md` for backend, `SHARED_CONTRACTS.md` for cross-agent.

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
No results on first call → simplify query (remove version specifiers) → if still none →
Fetch MCP official docs URL → if Fetch also fails → note limitation, use training memory with caveat.

---

## MCP Failure Handling

| Server unavailable | Fallback |
|---|---|
| Context7 | Fetch MCP → raw docs URL → note limitation |
| Memvid MCP | CLI fallback → DECISION_LOG.md + APP_STATE.md; cross-agent: SHARED_CONTRACTS.md |
| Fetch | Note limitation, use training memory with explicit caveat |

Never silently fall back to training memory when MCP was required.
