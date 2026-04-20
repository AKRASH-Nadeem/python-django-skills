# Memvid — Persistent Memory for AI Agents (Backend)

> **Load this skill** when: troubleshooting Memvid, configuring `.mv2` files,
> working with knowledge graph tools, session replay, `.mv2e` encryption,
> or any advanced Memvid operation beyond the daily `find/put/ask` workflow.
>
> **Day-to-day recall and storage** is covered in `mcp-servers.md §2` and
> `rules.md Step 1.5`. Read those first.

---

## What is Memvid?

Memvid is a **zero-infrastructure, single-file memory layer** for AI agents.
It replaces complex RAG pipelines with a portable `.mv2` file containing data,
embeddings, search indices, and metadata — all in one file.

**Architecture:**
- **Smart Frames**: Immutable, append-only units with timestamps, checksums, metadata
- **Single file**: Everything in one `.mv2` — no `.wal`, `.lock`, `.shm` sidecars
- **Crash safe**: Embedded WAL ensures data integrity
- **Portable**: Copy the file, carry the memory anywhere

**Capabilities:**
- Hybrid search (lexical via Tantivy + semantic via HNSW vectors)
- Knowledge graph (Logic-Mesh NER entity extraction)
- Time-travel debugging (rewind, replay, branch memory states)
- Session recording and replay
- AES-256-GCM encryption
- Sub-5ms local memory access with predictive caching

**MCP Server:** `Tapiocapioca/memvid-mcp` wraps the memvid Rust CLI, exposing 40 tools.
**Default embedding model:** `nomic` (nomic-embed-text-v1.5) — 768 dimensions, local, no API key.

---

## Memory Topology — 3-File Architecture

```
backend.mv2   → Backend agent's PRIVATE memory
frontend.mv2  → Frontend agent's PRIVATE memory
shared.mv2    → SHARED cross-agent memory
```

| Memory type | File | Examples |
|------------|------|----------|
| Backend-only decisions | `backend.mv2` | Django patterns, Celery strategy, ORM conventions |
| Frontend-only decisions | `frontend.mv2` | Component structure, state choice, CSS approach |
| Cross-agent contracts | `shared.mv2` | API endpoint schema, auth token flow, error envelope, WebSocket events |

**Rule: When in doubt → `shared.mv2`.** Both agents seeing a decision is better than one missing it.

---

## INITIALIZATION PROTOCOL (First Session)

Run once when `.mv2` files do not exist yet:

```
# Step 1: Create all three files
memvid_create { "file": "shared.mv2" }
memvid_create { "file": "backend.mv2" }
memvid_create { "file": "frontend.mv2" }

# Step 2: Verify creation
memvid_stats { "file": "shared.mv2" }
memvid_stats { "file": "backend.mv2" }

# Step 3: Migrate existing decisions
memvid_put_many { "file": "backend.mv2", "input": "DECISION_LOG.md", "embed": true }

# Step 4: Seed shared memory with project context from APP_STATE.md
memvid_put {
  "file": "shared.mv2",
  "input": "[agent:backend] [type:project-context] [date:YYYY-MM-DD]\n[Paste key sections: Tech Stack, External Services, Known Constraints from APP_STATE.md]",
  "embed": true
}

# Step 5: Build knowledge graph
memvid_enrich { "file": "backend.mv2", "all": true }
memvid_enrich { "file": "shared.mv2", "all": true }

# Step 6: Verify embeddings loaded
memvid_stats { "file": "backend.mv2" }
# Confirm: vector_count > 0. If 0 → re-run put with "embed": true
```

---

## SESSION LIFECYCLE

Every coding session follows this lifecycle:

### Session Start
```
# Already covered in rules.md Step 1.5 — shown here for completeness
memvid_stats { "file": "backend.mv2" }  # confirm file healthy
memvid_session { "file": "backend.mv2", "start": "be-[YYYYMMDD-HH]" }
memvid_find { "file": "shared.mv2", "query": "relevant project context", "mode": "hybrid", "limit": 5 }
memvid_find { "file": "backend.mv2", "query": "recent decisions constraints", "mode": "hybrid", "limit": 5 }
```

### During Session — After Each Architectural Decision
```
memvid_put {
  "file": "[backend.mv2 or shared.mv2]",
  "input": "[agent:backend] [type:X] [feature:Y] [date:YYYY-MM-DD]\n...",
  "embed": true
}
# Then optionally:
memvid_enrich { "file": "backend.mv2", "all": false }  # extract new entities
```

### Session End — Consolidation
```
# Synthesize what was decided
memvid_ask {
  "file": "backend.mv2",
  "question": "What architectural decisions were made in the most recent session?"
}

# Review session timeline
memvid_timeline { "file": "backend.mv2" }

# Stop session
memvid_session { "file": "backend.mv2", "stop": true }
```

---

## PROACTIVE STORAGE — WHEN TO WRITE

**Write immediately (do not defer) when:**

| Trigger | File | Tag |
|---------|------|-----|
| Library chosen over alternatives | `backend.mv2` or `shared.mv2` | `[type:architectural-decision]` |
| API endpoint schema finalised | `shared.mv2` | `[type:api-contract]` |
| Error response format established | `shared.mv2` | `[type:api-contract]` |
| Auth token flow decided | `shared.mv2` | `[type:api-contract]` |
| Django app structure confirmed | `backend.mv2` | `[type:convention]` |
| Service/selector pattern established | `backend.mv2` | `[type:convention]` |
| Developer constraint stated ("no Redis", "max N workers") | `backend.mv2` | `[type:constraint]` |
| Integration point with frontend defined | `shared.mv2` | `[type:api-contract]` |
| Decision reversed | Update existing frame with `memvid_update` | Do NOT append |

**Do NOT store:**
- Bug fixes following an already-stored pattern
- New model fields following established conventions
- Test additions or style changes
- Anything that doesn't change how future decisions should be made

---

## MEMORY FORMAT — REQUIRED STRUCTURE

```
memvid_put {
  "file": "shared.mv2 | backend.mv2",
  "input": "[agent:backend] [type:TYPE] [feature:FEATURE] [date:YYYY-MM-DD]
Decision: One sentence.
Why: Core reasoning.
Rejected: Alt 1 (reason). Alt 2 (reason).
Invalidated by: The condition that would make this decision wrong.",
  "embed": true
}
```

**Type values:**
- `architectural-decision` — Library/framework choice, pattern establishment
- `api-contract` — Endpoint schema, payload format, auth flow, error envelope
- `convention` — Naming, file structure, coding pattern team has settled on
- `constraint` — Hard limit from developer or infrastructure
- `project-context` — Stack/infrastructure snapshot

---

## KNOWLEDGE GRAPH WORKFLOW

The knowledge graph extracts entities (services, libraries, patterns) and their relationships.

```
# After storing multiple related memories:
memvid_enrich { "file": "backend.mv2", "all": true }

# Look up a specific entity
memvid_who { "file": "backend.mv2", "query": "CeleryTaskQueue" }

# Follow entity relationships (2 hops)
memvid_follow { "file": "backend.mv2", "entity": "AuthService", "hops": 2 }

# List all known facts
memvid_facts { "file": "backend.mv2" }

# Current memory state (entities + relationships summary)
memvid_state { "file": "backend.mv2" }
```

**When to use:** After seeding a project for the first time, or after a sprint where many new services/libraries were introduced. Run `memvid_enrich` to let the graph catch up.

---

## CROSS-AGENT COMMUNICATION VIA shared.mv2

Both agents write to and read from `shared.mv2`. Since both run concurrently:

**Write sequencing rule:** Each agent writes to its own private file first, then to `shared.mv2`. Never write to `shared.mv2` from two agents simultaneously for the same contract — coordinate via the developer if a conflict appears.

**How backend discovers frontend decisions:**
```
memvid_find { "file": "shared.mv2", "query": "frontend state management component conventions", "mode": "hybrid", "limit": 5 }
```

**How frontend discovers backend API:**
```
memvid_find { "file": "shared.mv2", "query": "backend api endpoint schema error format auth", "mode": "hybrid", "limit": 5 }
```

**Conflict protocol:**
```
⚠️ MEMORY CONFLICT: shared.mv2 has two contradictory entries:
- [agent:frontend, Date]: "[statement 1]"
- [agent:backend, Date]: "[statement 2]"
Which should I follow?
```
After user resolves: `memvid_put` winner, `memvid_delete` loser. Never silently pick one.

---

## ADVANCED OPERATIONS

### Audit report (with citations)
```
memvid_audit { "file": "backend.mv2", "query": "security decisions", "include_snippets": true }
```

### Temporal search (when was something decided?)
```
memvid_when { "file": "backend.mv2", "query": "when was Redis cache strategy decided?" }
```

### Timeline view (chronological)
```
memvid_timeline { "file": "backend.mv2" }
```

### Export for backup
```
memvid_export { "file": "backend.mv2", "output": "backend-backup.json", "format": "json" }
```

### Encryption (for sensitive memory)
```
memvid_lock { "file": "backend.mv2", "output": "backend.mv2e", "password": "your-password" }
memvid_unlock { "file": "backend.mv2e", "output": "backend.mv2", "password": "your-password" }
```

### Bulk import (ingesting a document folder)
```
memvid_put_many { "file": "backend.mv2", "input": "./docs/architecture", "recursive": true, "embed": true }
```

### Fetch and store from URL (ingesting live docs)
```
memvid_api_fetch { "file": "backend.mv2", "url": "https://docs.djangoproject.com/en/6.0/topics/..." }
```

### Quick sketch (lightweight note without full embedding)
```
memvid_sketch { "file": "backend.mv2" }
```

---

## MEMORY HYGIENE

### Stale memory detection
If a recalled decision references a library, file, or pattern that no longer exists:
```
# Update the stale frame (do NOT delete — preserve history)
memvid_update { "file": "backend.mv2", "frame_id": [ID], "input": "SUPERSEDED: [original content]\nSuperseded by: [new decision] on [date]." }
```

### Pruning genuine conflicts
When a decision is truly reversed and the old record is harmful (not just stale):
```
memvid_delete { "file": "backend.mv2", "frame_id": [ID] }
# Then store the new decision with memvid_put
```

### Periodic consolidation (every few sprints)
```
memvid_ask { "file": "backend.mv2", "question": "Which decisions are still active? Which are stale?" }
memvid_timeline { "file": "backend.mv2" }
```

---

## TROUBLESHOOTING

| Issue | Solution |
|-------|---------|
| `memvid_find` returns no results | `memvid_stats` — is `vector_count > 0`? If 0: re-put with `"embed": true` |
| Slow search | Use `"mode": "lex"` for keyword-only (fastest); `"sem"` for semantic-only |
| Memory file too large | `memvid_export` to backup, create fresh with curated memories |
| Model not found | `memvid_models {}` — ensure nomic is installed |
| MCP server won't start | Node.js 18+: `node --version`; memvid CLI: `memvid --version`; check `MEMVID_PATH` env var |
| `memvid_put` text not stored | Use raw text directly in `"input"` field. For file: provide path. Ensure `"embed": true`. |
| Session not recording | `memvid_status { "file": "backend.mv2" }` — check active session |
| Cross-agent conflict | Surface to user with `⚠️ MEMORY CONFLICT:` format — never silently resolve |

---

## Full Tool Catalog (40 Tools)

### Lifecycle (5)
`memvid_create` · `memvid_open` · `memvid_stats` · `memvid_verify` · `memvid_doctor`

### Content (7)
`memvid_put` · `memvid_put_many` · `memvid_view` · `memvid_update` · `memvid_delete` · `memvid_correct` · `memvid_api_fetch`

### Search (5)
`memvid_find` · `memvid_vec_search` · `memvid_ask` · `memvid_timeline` · `memvid_when`

### Knowledge Graph (6)
`memvid_enrich` · `memvid_memories` · `memvid_state` · `memvid_facts` · `memvid_follow` · `memvid_who`

### Session (5)
`memvid_session` · `memvid_binding` · `memvid_status` · `memvid_sketch` · `memvid_nudge`

### Analysis (6)
`memvid_audit` · `memvid_debug_segment` · `memvid_export` · `memvid_tables` · `memvid_schema` · `memvid_models`

### Encryption (2)
`memvid_lock` · `memvid_unlock`

### Utility (4)
`memvid_process_queue` · `memvid_verify_single_file` · `memvid_config` · `memvid_version`
