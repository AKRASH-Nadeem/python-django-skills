# Memvid — Persistent Memory for AI Agents

> **Load this skill** when the task involves: memory, remembering, recalling,
> storing decisions, cross-agent context, architectural history, `.mv2` files,
> knowledge graph, timeline, session recording, or any reference to Memvid.

---

## What is Memvid?

Memvid is a **zero-infrastructure, single-file memory layer** for AI agents.
It replaces complex RAG pipelines and server-based vector databases with a
portable `.mv2` file that contains data, embeddings, search indices, and metadata.

**Architecture:**
- **Smart Frames**: Immutable, append-only units with timestamps, checksums, metadata
- **Single file**: Everything in one `.mv2` file — no `.wal`, `.lock`, `.shm` sidecars
- **Crash safe**: Embedded WAL ensures data integrity
- **Portable**: Copy the file, carry the memory anywhere

**Capabilities:**
- Hybrid search (lexical via Tantivy + semantic via HNSW vectors)
- Knowledge graph (Logic-Mesh NER entity extraction)
- Time-travel debugging (rewind, replay, branch memory states)
- Session recording and replay
- AES-256-GCM encryption
- Sub-5ms local memory access with predictive caching

---

## MCP Server

Memvid runs as an MCP server via [Tapiocapioca/memvid-mcp](https://github.com/Tapiocapioca/memvid-mcp),
exposing **40 tools** to the agent. The server wraps the `memvid` Rust CLI.

### Default Embedding Model

**`nomic`** (nomic-embed-text-v1.5) — 768 dimensions, local, no API key, no rate limits.
All embedding operations use `nomic` unless explicitly overridden.

---

## Memory Topology — 3-File Architecture

```
frontend.mv2   → Frontend agent's PRIVATE memory (component patterns, UI decisions, hooks, state management)
backend.mv2    → Backend agent's PRIVATE memory (model conventions, service patterns, Django/API decisions)
shared.mv2     → SHARED cross-agent memory (API contracts, auth strategy, error formats, integration points)
```

### When to write to which file

| Memory type | File | Examples |
|------------|------|----------|
| Frontend-only decisions | `frontend.mv2` | "Chose Zustand over Redux", "Component folder structure", "Form validation approach" |
| Backend-only decisions | `backend.mv2` | "Chose Knox over SimpleJWT", "Celery task pattern", "Migration strategy" |
| Cross-agent contracts | `shared.mv2` | "API error envelope format", "Auth token flow", "Endpoint naming convention", "WebSocket event schema" |

### Rule: When in doubt, write to `shared.mv2`

If a decision could affect the other agent, it belongs in shared memory.
Better to have a frontend decision visible to the backend agent than to have
the backend agent make an incompatible choice.

---

## Core Operations — Quick Reference

### 1. Store a Memory (`memvid_put`)

```
memvid_put {
  "file": "shared.mv2",
  "input": "[agent:frontend] [type:architectural-decision] [feature:auth] [date:2025-11-01]\nDecision: Chose Zustand for client-side auth state.\nWhy: Lightweight, no boilerplate, works with React Compiler.\nRejected: Redux (overkill), Context (re-render issues at scale).\nInvalidated by: Need for server-side state sync or DevTools time-travel.",
  "embed": true
}
```

**Always include:**
- `"embed": true` — enables semantic search via nomic embeddings
- Tag prefix: `[agent:frontend]` or `[agent:backend]`
- Type tag: `[type:architectural-decision]`, `[type:api-contract]`, `[type:convention]`, `[type:bug-fix]`, `[type:constraint]`

### 2. Recall a Memory (`memvid_find`)

```
memvid_find {
  "file": "shared.mv2",
  "query": "authentication state management",
  "mode": "hybrid",
  "limit": 5
}
```

**Search modes:**
- `"hybrid"` — Lexical + semantic (recommended for most queries)
- `"lex"` — Lexical only (exact keyword matches)
- `"sem"` — Semantic only (meaning-based, uses embeddings)

### 3. Ask a Question (`memvid_ask`)

```
memvid_ask {
  "file": "shared.mv2",
  "question": "What authentication strategy was chosen and why?"
}
```

RAG-powered: retrieves relevant memories, synthesizes an answer. Use for
complex questions that need reasoning across multiple stored memories.

### 4. Check Memory Stats (`memvid_stats`)

```
memvid_stats {
  "file": "shared.mv2"
}
```

Returns frame count, vector count, file size, and metadata.

---

## Full Tool Catalog (40 Tools)

### Lifecycle (5 tools)

| Tool | Purpose | Example |
|------|---------|---------|
| `memvid_create` | Create a new `.mv2` file | `{ "file": "project.mv2" }` |
| `memvid_open` | Open existing `.mv2` | `{ "file": "project.mv2" }` |
| `memvid_stats` | File statistics | `{ "file": "project.mv2" }` |
| `memvid_verify` | Verify file integrity | `{ "file": "project.mv2" }` |
| `memvid_doctor` | Diagnose and repair | `{ "file": "project.mv2" }` |

### Content Management (7 tools)

| Tool | Purpose | Example |
|------|---------|---------|
| `memvid_put` | Store content with embeddings | `{ "file": "x.mv2", "input": "...", "embed": true }` |
| `memvid_put_many` | Bulk store (batch) | `{ "file": "x.mv2", "input": "./docs", "recursive": true, "embed": true }` |
| `memvid_view` | View a specific frame | `{ "file": "x.mv2", "frame_id": 42 }` |
| `memvid_update` | Update frame content | `{ "file": "x.mv2", "frame_id": 42, "input": "new content" }` |
| `memvid_delete` | Delete a frame | `{ "file": "x.mv2", "frame_id": 42 }` |
| `memvid_correct` | Correct content errors | `{ "file": "x.mv2", "frame_id": 42, "correction": "fixed text" }` |
| `memvid_api_fetch` | Fetch and store from URL | `{ "file": "x.mv2", "url": "https://..." }` |

### Search (5 tools)

| Tool | Purpose | Example |
|------|---------|---------|
| `memvid_find` | Hybrid search (lex + vec) | `{ "file": "x.mv2", "query": "...", "mode": "hybrid", "limit": 5 }` |
| `memvid_vec_search` | Semantic-only search | `{ "file": "x.mv2", "query": "..." }` |
| `memvid_ask` | RAG Q&A (retrieve + synthesize) | `{ "file": "x.mv2", "question": "..." }` |
| `memvid_timeline` | Time-ordered memory view | `{ "file": "x.mv2" }` |
| `memvid_when` | Temporal query | `{ "file": "x.mv2", "query": "when was auth decided?" }` |

### Knowledge Graph (6 tools)

| Tool | Purpose | Example |
|------|---------|---------|
| `memvid_enrich` | Extract entities (NER) | `{ "file": "x.mv2", "all": true }` |
| `memvid_memories` | List stored memories | `{ "file": "x.mv2" }` |
| `memvid_state` | Current knowledge state | `{ "file": "x.mv2" }` |
| `memvid_facts` | Extract/list facts | `{ "file": "x.mv2" }` |
| `memvid_follow` | Follow entity relationships | `{ "file": "x.mv2", "entity": "AuthService", "hops": 2 }` |
| `memvid_who` | Entity lookup | `{ "file": "x.mv2", "query": "OAuth" }` |

### Session Management (5 tools)

| Tool | Purpose | Example |
|------|---------|---------|
| `memvid_session` | Start/stop/replay sessions | `{ "file": "x.mv2", "start": "session-name" }` |
| `memvid_binding` | Bind context to session | `{ "file": "x.mv2" }` |
| `memvid_status` | Current session status | `{ "file": "x.mv2" }` |
| `memvid_sketch` | Quick memory sketch | `{ "file": "x.mv2" }` |
| `memvid_nudge` | Nudge agent toward memory | `{ "file": "x.mv2" }` |

### Analysis (6 tools)

| Tool | Purpose | Example |
|------|---------|---------|
| `memvid_audit` | Audit report with sources | `{ "file": "x.mv2", "query": "security", "include_snippets": true }` |
| `memvid_debug_segment` | Debug specific segments | `{ "file": "x.mv2" }` |
| `memvid_export` | Export to JSON/backup | `{ "file": "x.mv2", "output": "backup.json", "format": "json" }` |
| `memvid_tables` | View structured tables | `{ "file": "x.mv2" }` |
| `memvid_schema` | View data schema | `{ "file": "x.mv2" }` |
| `memvid_models` | List installed models | `{}` |

### Encryption (2 tools)

| Tool | Purpose | Example |
|------|---------|---------|
| `memvid_lock` | Encrypt `.mv2` → `.mv2e` | `{ "file": "x.mv2", "output": "x.mv2e", "password": "..." }` |
| `memvid_unlock` | Decrypt `.mv2e` → `.mv2` | `{ "file": "x.mv2e", "output": "x.mv2", "password": "..." }` |

### Utility (4 tools)

| Tool | Purpose | Example |
|------|---------|---------|
| `memvid_process_queue` | Process queued operations | `{}` |
| `memvid_verify_single_file` | Verify one file | `{ "file": "x.mv2" }` |
| `memvid_config` | View/set configuration | `{}` |
| `memvid_version` | CLI version info | `{}` |

---

## Cross-Agent Memory Protocol

### Writing to shared memory

When storing a decision that affects both agents:

1. **Tag with agent identity**: Always prefix with `[agent:frontend]` or `[agent:backend]`
2. **Tag with type**: `[type:api-contract]`, `[type:architectural-decision]`, `[type:convention]`
3. **Include invalidation conditions**: What would cause this to change?
4. **Enable embeddings**: Always `"embed": true`

### Reading shared memory

Before making any architectural decision:

1. **Search shared memory first**: `memvid_find` on `shared.mv2` for related context
2. **Check for conflicts**: Does the new decision contradict existing shared memories?
3. **If conflict found**: Flag it explicitly — do NOT silently override

### Conflict Resolution

```
MEMVID CONFLICT: Memory in shared.mv2 says [existing decision].
  Current request appears to contradict this.
  The original decision was made because [constraint from memory].
  Does [constraint] still apply?
```

---

## Memory Hygiene

### When to store (DO store)
- Architectural decisions (state management, routing, auth strategy)
- API contracts between frontend and backend
- Library selections and rejections with reasoning
- Constraints from the developer ("no Redux", "bundle < 200KB")
- Integration points (endpoint schemas, error formats, WebSocket events)

### When NOT to store
- Bug fixes following existing patterns
- New components following established patterns
- Style changes, copy changes
- Test additions
- Anything that doesn't change architectural reasoning

### Pruning stale memories
- When a decision is reversed: use `memvid_update` on the original frame, do NOT append a new contradicting memory
- When a memory is obsolete: use `memvid_delete` to remove it
- Periodically use `memvid_timeline` to review memory evolution

---

## Fallback: When Memvid MCP is Unavailable

If the Memvid MCP server is not connected or not responding:

1. **Do NOT block the task** — proceed without memory
2. **Fall back to `DECISION_LOG.md`** in the project root
3. **Log the fallback**: Note that Memvid was unavailable and the decision was recorded in DECISION_LOG.md
4. **On next session with Memvid available**: Migrate DECISION_LOG.md entries to the appropriate `.mv2` file

---

## Troubleshooting

| Issue | Solution |
|-------|---------|
| `memvid_find` returns no results | Check `memvid_stats` — is `vector_count > 0`? If not, re-put with `"embed": true` |
| Slow search performance | Use `"mode": "lex"` for keyword-only search (fastest) |
| Memory file too large | Use `memvid_export` to backup, then create a fresh file with curated memories |
| Model not found | Run `memvid models list --model-type embedding` — ensure nomic is installed |
| MCP server won't start | Check Node.js 18+: `node --version`; verify memvid CLI: `memvid --version` |
