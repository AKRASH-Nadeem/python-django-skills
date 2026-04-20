---
name: agent-memory
description: >
  LOAD THIS SKILL before any implementation task and at session start.
  Governs when to write memory, when NOT to write, what to write, how to
  format it, and how to recall it. Triggers: session start, any architectural
  decision, library chosen or rejected, API endpoint defined, auth/cache/queue/
  storage decided, developer states a constraint, convention established,
  previous decision reversed, API contract finalised, WebSocket event defined,
  error format established. Also load for: memory conflicts, CLI fallback,
  memvid MCP errors, cross-agent shared memory questions, troubleshooting
  memvid_* tool failures.
---

# Memvid — Backend Memory Behaviour Guide

> `backend.mv2` is your PRIMARY memory. Every decision that would change
> how you reason next session goes here. `shared.mv2` is the BRIDGE to
> the frontend agent — API contracts, auth flows, error formats only.
> Training memory is a DRAFT. Memory files are the source of truth.

---

## RULE 0 — Flash Override (Read First)

Gemini 3 Flash is speed-optimised and WILL skip memory operations to respond
faster. These rules exist to override that bias. They are non-negotiable:

1. **Write IMMEDIATELY** after a decision — not at end of session.
2. **Recall BEFORE forming an opinion** — not after.
3. **Every `memvid_put` MUST include `"embed": true`** — without it, semantic search returns nothing next session.
4. **Every shared.mv2 write MUST include `[agent:backend]`** — the frontend reads this to identify who owns the contract.
5. **Never compress the memory format** — Flash wants one-liners. Use the 4-field format anyway.
6. **If MCP is slow or erroring → use CLI fallback** (see §7). Do NOT skip the write.

---

## §1 — AUTO-DETECT: These ALWAYS Trigger an Immediate Write

When any of the following happens, write to memory NOW — no confirmation needed:

| What happened | Write to | Tag |
|---|---|---|
| Library chosen over alternatives | `backend.mv2` | `[type:architectural-decision]` |
| Library rejected with reason | `backend.mv2` | `[type:architectural-decision]` |
| Django app structure confirmed | `backend.mv2` | `[type:convention]` |
| Service / selector / task pattern established | `backend.mv2` | `[type:convention]` |
| Developer stated a constraint ("no Redis", "max N workers", "no X library") | `backend.mv2` | `[type:constraint]` |
| ORM, caching, or queue strategy decided | `backend.mv2` | `[type:architectural-decision]` |
| API endpoint schema finalised | **`shared.mv2`** | `[type:api-contract]` |
| Error response format established | **`shared.mv2`** | `[type:api-contract]` |
| Auth token flow decided | **`shared.mv2`** | `[type:api-contract]` |
| WebSocket event format defined | **`shared.mv2`** | `[type:api-contract]` |
| Any integration point with the frontend defined | **`shared.mv2`** | `[type:api-contract]` |
| A previous decision is reversed | Update existing frame — NEVER append a contradiction |

---

## §2 — WRITE vs SKIP

**WRITE — these change future reasoning:**
- Library choices + the alternatives that were rejected and why
- Architecture patterns the codebase is now committed to
- Developer/infrastructure constraints that limit future options
- API contracts the frontend depends on
- Auth, cache, queue, storage decisions

**SKIP — these do not change future reasoning:**
- Bug fixes that follow an already-stored pattern
- New model fields following an established convention
- Test additions, style changes, copy edits
- Any code change that fully follows a decision already in memory

**When uncertain → WRITE.** A redundant frame costs 50ms. Missing context costs a full session of wrong assumptions.

---

## §3 — WHERE TO WRITE

```
Django ORM / models / migrations?                     → backend.mv2
Celery tasks / scheduling / queue strategy?           → backend.mv2
Service layer / selector / Django app structure?      → backend.mv2
Python patterns / conventions for this project?       → backend.mv2

API endpoint schema / URL / HTTP method / payload?    → shared.mv2
Auth token format / flow / expiry / storage?          → shared.mv2
Error envelope format / HTTP status mapping?          → shared.mv2
WebSocket events / SSE event names / payload?         → shared.mv2
Any data shape the frontend must parse?               → shared.mv2

Decision could affect either agent?                   → shared.mv2
When in doubt?                                        → shared.mv2
```

Rule: **shared.mv2 is not an archive — it is a live contract.** Only write
to it when a frontend agent needs the information to build correctly.

---

## §4 — HOW TO WRITE — Required Format

```
memvid_put {
  "file": "backend.mv2",
  "input": "[agent:backend] [type:TYPE] [feature:FEATURE] [date:YYYY-MM-DD]\nDecision: One sentence — what was decided.\nWhy: The non-obvious reasoning. Why this over alternatives.\nRejected: Alt1 (reason). Alt2 (reason). Omit only if no alternatives existed.\nInvalidated by: The single condition that would make this decision wrong.",
  "embed": true
}
```

**Required tags — never omit:**
- `[agent:backend]` — authorship. Mandatory in shared.mv2. Required in backend.mv2 for consistency.
- `[type:TYPE]` → `architectural-decision` | `api-contract` | `convention` | `constraint` | `project-context`
- `[feature:FEATURE]` → the Django app / domain (e.g. `auth`, `orders`, `payments`, `core`)
- `[date:YYYY-MM-DD]` → today's date
- `"embed": true` → always

**Good memory (store this):**
```
[agent:backend] [type:api-contract] [feature:auth] [date:2026-04-20]
Decision: JWT access 15min, refresh 7 days, refresh in httpOnly cookie, access in memory.
Why: httpOnly blocks XSS on refresh. In-memory access token never hits disk.
Rejected: localStorage (XSS vulnerable). Cookie for both (CSRF exposure on access token).
Invalidated by: Mobile client required that cannot use httpOnly cookies.
```

**Bad memory (never do this):**
```
Added JWT auth.
```
No tags. No routing key. No rejection record. No invalidation condition. Useless on recall.

---

## §5 — RECALL: When and How

### Session start (mandatory — runs before any implementation):
```
memvid_find { "file": "shared.mv2", "query": "project architecture api contracts auth stack", "mode": "hybrid", "limit": 5 }
memvid_find { "file": "backend.mv2", "query": "recent decisions constraints conventions patterns", "mode": "hybrid", "limit": 5 }
```

### Before any architectural decision:
```
memvid_find { "file": "backend.mv2", "query": "[topic] decision alternatives rejected constraint", "mode": "hybrid", "limit": 5 }
```

### Before building any API endpoint (two queries — both mandatory):
```
memvid_find { "file": "backend.mv2", "query": "[endpoint topic] api convention pattern", "mode": "hybrid", "limit": 5 }
memvid_find { "file": "shared.mv2", "query": "frontend expectations [endpoint topic] response shape error format", "mode": "hybrid", "limit": 5 }
```
The second query catches what the frontend agent has already stored about its expectations.
Skip it → build an endpoint the frontend cannot consume.

### Recall rules:
- Use `"mode": "hybrid"` always (lexical + semantic combined)
- `limit: 5` for targeted decisions; `limit: 10` for broad audits
- A result tagged `[agent:frontend]` = frontend expectation. Do not contradict without surfacing conflict.
- Empty result = no prior decision. Build fresh, then store what you decide.
- **Never skip recall because you are "confident"** — backend.mv2 overrides training memory.

---

## §6 — MEMORY HYGIENE: Updating Reversed Decisions

When a decision is reversed, DO NOT append a new contradicting frame.
Update the original:

```bash
# 1. Find the original frame
memvid_find { "file": "backend.mv2", "query": "[original decision topic]", "mode": "hybrid", "limit": 3 }

# 2. Mark it superseded (preserve history)
memvid_update {
  "file": "backend.mv2",
  "frame_id": [ID],
  "input": "SUPERSEDED [date:YYYY-MM-DD]: [original content]\nSuperseded by: [new decision]. Reason: [why it changed]."
}

# 3. Write the new decision normally
memvid_put { "file": "backend.mv2", "input": "[new decision in standard format]", "embed": true }
```

Use `memvid_delete` only when a frame is actively harmful (wrong security decision,
incorrect contract that could mislead the frontend agent). Stale = update. Dangerous = delete.

---

## §7 — CLI FALLBACK: When MCP Tools Fail

If `memvid_*` MCP tools are unavailable, slow, or returning errors, use the CLI.
State: *"Memvid MCP unavailable. Using CLI fallback."*

```bash
# Health check
memvid --version
memvid stats backend.mv2

# Write a decision
memvid put backend.mv2 \
  "[agent:backend] [type:architectural-decision] [feature:auth] [date:2026-04-20]
Decision: ...
Why: ...
Rejected: ...
Invalidated by: ..." --embed

# Write to shared memory
memvid put shared.mv2 \
  "[agent:backend] [type:api-contract] [feature:auth] [date:2026-04-20]
Decision: ...
Why: ...
Rejected: ...
Invalidated by: ..." --embed

# Search
memvid find backend.mv2 "authentication strategy" --mode hybrid --limit 5
memvid find shared.mv2 "api contract frontend" --mode hybrid --limit 5

# Create missing file
memvid create backend.mv2
memvid create shared.mv2
```

**If CLI also unavailable:**
```bash
# Fall back to DECISION_LOG.md for backend decisions
echo "## [$(date +%Y-%m-%d)] [feature:auth]\nDecision: ...\nWhy: ...\nRejected: ...\nInvalidated by: ..." >> DECISION_LOG.md

# Fall back to SHARED_CONTRACTS.md for cross-agent contracts
echo "## [agent:backend] [$(date +%Y-%m-%d)] [feature:auth]\nContract: ..." >> SHARED_CONTRACTS.md
```

---

## §8 — SHARED MEMORY: Authorship and Conflict Rules

`shared.mv2` is read by both backend and frontend agents. Authorship matters.

**Reading shared.mv2 results:**

| Tag in result | Meaning | Your action |
|---|---|---|
| `[agent:backend]` | You wrote this. | Validate it is still correct. |
| `[agent:frontend]` | Frontend has a stated expectation. | Do not contradict without surfacing conflict. |
| No `[agent:]` tag | Legacy / unknown author. | Treat as potentially stale. Verify before using. |

**Conflict protocol:**
```
⚠️ MEMORY CONFLICT detected in shared.mv2:
  [agent:frontend, date]: "[frontend expectation]"
  [agent:backend, date]: "[backend decision]"
Do not silently pick one. Surface to user. After resolution:
  → memvid_put winner  → memvid_delete loser
```

**Enrich knowledge graph after significant writes:**
```
memvid_enrich { "file": "backend.mv2", "all": false }
```
Run after any `memvid_put` that introduces a new service name, library, or
architectural entity. Omit for constraint or convention updates.
