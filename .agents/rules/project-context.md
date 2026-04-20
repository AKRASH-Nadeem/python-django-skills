---
trigger: always_on
---

> PROJECT CONTEXT MANDATE: At the start of every session, establish what is being built,
> what is approved, and what the current state is. Build one feature at a time. Never move
> to the next feature until the current one passes all validation gates.

---

# Project Context & Incremental Implementation Standards

---

## PC0. Why This Rule Exists

AI agents have no persistent memory between sessions. Without explicit context management:
- The agent forgets what infrastructure was approved in the previous session
- It reintroduces libraries the user already rejected
- It builds feature N+1 on top of an unverified feature N
- The codebase drifts into inconsistency
- The agent assumes Redis is available because it was once mentioned, not because it's in APP_STATE.md

This rule prevents all of that.

---

## PC1. Session Start Protocol

State project context before doing anything else.

### If context was established previously:
```
## Session context — [Project name]
Stack: [from APP_STATE.md Tech Stack section]
Architecture: Feature-Based Django Apps
Last verified state: [what was working at end of last session]
Current feature: [what we are building now]
Pending features: [what comes after]
Starting from: [app / module / endpoint]
```

If the user does not provide context:
*"Before we begin — can you confirm the current state? What was the last working feature, and what are we building today?"*

Then:
1. Read `APP_STATE.md` fully
2. Read `DECISION_LOG.md` fully — flag any conflicts with the current request
3. Read `LIBRARY_LEDGER.md` summary table
4. Attempt Memvid recall via `memvid_find` if MCP is connected (see `mcp-servers.md`)

### If new project: run stack proposal from `tech-stack.md` TS1 first. No code until stack is approved.

---

## PC2. Feature Definition

A feature is too vague to implement if it can't be described in one sentence with a clear completion condition.

```
Feature: [name]
Description: [one sentence — what it does, from the user's perspective]
Scope: [apps, models, services, serializers, API endpoints, tasks involved]
Out of scope: [tempting things that belong in a later feature]
Done when:
  - [verifiable condition 1 — e.g., "POST /api/v1/orders/ returns 201 with order data"]
  - [mypy passes with zero errors]
  - [pytest passes with 80%+ coverage on new code]
  - [no migration conflicts]
```

---

## PC3. Incremental Implementation Protocol

**One feature. Fully complete. Verified. Then the next.**

Implementation sequence per feature:
```
1. Define with PC2 — confirm with user
2. List all files that need to change before touching any
3. Implement in dependency order:
   models/migrations → managers/querysets → services → selectors → serializers → views → urls → tasks → tests
4. After each file: run mypy — zero errors before continuing
5. After all files: run pytest — all tests pass
6. Run validation gates (PC4)
7. Present: "Feature [X] complete. Here's what was built: [summary]"
8. Update APP_STATE.md, LIBRARY_LEDGER.md, DECISION_LOG.md as required
9. Wait for validation confirmation before starting next feature
```

**Dependency order:** data shapes → data access → business logic → HTTP interface → async workers → tests.
Never a view before its service. Never a service before its model.

**3+ features:** List full sequence → get confirmation on order → implement one at a time with gates.

---

## PC4. Validation Gates

A feature is **not complete** until ALL applicable gates pass.

### Gate 1 — Type Checking
```bash
mypy apps/ --ignore-missing-imports
# Zero errors required
# or: pyright apps/
```

### Gate 2 — Test Suite
```bash
python -m pytest --cov=apps --cov-report=term-missing -x
# Zero failures required
# 80%+ coverage on new code required
```

### Gate 3 — Migration consistency
```bash
python manage.py makemigrations --check
python manage.py migrate --run-syncdb
# Zero unapplied migrations required
```

### Gate 4 — Django system check
```bash
python manage.py check
# Zero errors required (warnings are acceptable)
```

### Gate 5 — All failure scenarios addressed
- [ ] Simultaneous request scenario handled (transaction.atomic or idempotency key)
- [ ] Duplicate request scenario handled (idempotency)
- [ ] Dependency failure scenario handled (external service down)
- [ ] Boundary conditions validated (serializer enforces limits)
- [ ] Missing FK / cache key scenario handled (guard clause + test)
- [ ] Authorization verified server-side (not just in the view)
- [ ] N+1 query verified absent (select_related / prefetch_related)

### Gate 6 — Security baseline
- [ ] No hardcoded secrets or URLs in any new file
- [ ] User input is validated in serializer, not view
- [ ] Object-level permission check present (not just `IsAuthenticated`)
- [ ] No PII in logging calls, Sentry breadcrumbs, or Celery task arguments
- [ ] New env vars are in `.env.example` with comments

### Gate 7 — Linting
```bash
ruff check apps/
ruff format --check apps/
# Zero violations required
```

### Gate 8 — Build check (if Docker/CI available)
```bash
docker build --target production . && echo "Build OK"
# Zero errors required
```

**Gate 9 — Project Scaffold Complete** (new project only):
```bash
# All mandatory files exist
test -f APP_STATE.md && test -f LIBRARY_LEDGER.md && \
test -f DECISION_LOG.md && test -f .env.example && \
test -f requirements/base.txt && test -f requirements/development.txt
# Dev server starts without errors
python manage.py check --deploy 2>&1 | head -20
python manage.py runserver --check
# All installed packages have ledger entries
# Compare pip freeze count with LIBRARY_LEDGER.md entry count
```

If any gate fails: fix before proceeding. Never present an unfinished feature as complete.

---

## PC5. Scope Creep Prevention

If implementing feature N cleanly requires feature N+1:

1. **Do not silently expand scope.** State: *"To implement [N] cleanly, I also need [N+1 thing]. Include now or stub?"*
2. If include: update PC2 definition and pending feature list.
3. If stub: clean interface, no inline TODOs without a ticket reference.

---

## PC6. Context Refresh After Long Sessions

After 10+ exchanges on a complex feature, proactively re-state:

> *"Quick context check: building [feature], completed [X, Y, Z], remaining [A, B]. Still correct?"*

---

## PC7. Definition of Done

| Criterion | How to verify |
|---|---|
| All acceptance criteria from PC2 met | Review "Done when" list |
| Type checking passes | `mypy apps/` zero errors |
| All test failure scenarios implemented | Gate 5 checklist |
| Security baseline met | Gate 6 checklist |
| Migrations consistent | Gate 3 |
| Django system check clean | Gate 4 |
| Linting passes | Gate 7 |
| APP_STATE.md updated (if structural change) | Check mandatory trigger table in `rules.md` |
| LIBRARY_LEDGER.md updated (if package added) | Check `library-ledger.md` |
| DECISION_LOG.md updated (if architectural decision) | Check `senior-dev-mindset.md` |
| DECISION_LOG.md consistency verified | Review existing entries — flag any that this task invalidated |
| Memvid memory stored (if Memvid connected) | `memvid_put { "file": "shared.mv2", "input": "DECISION_LOG.md", "embed": true }` |
| User has confirmed it works | Explicit sign-off |

---

## PC8. Session Handoff & Context Window Protocol

### 8.1 — Context utilisation checkpoint

When a session exceeds **30 tool calls** OR when approaching a long pause, proactively write state to files:

1. Ensure all architectural decisions are written to `DECISION_LOG.md`, then call Memvid `memvid_put` on it to ensure `shared.mv2` is synced.
2. Update `APP_STATE.md` if any structural change was made.
3. Store memories in Memvid if connected.
4. State: *"Context checkpoint written. Continuing."*

### 8.2 — 85% context window rule

If you detect that the context window is approaching capacity:

**STOP immediately and do all of the following before any more code:**

1. Write a handoff document at the project root: `HANDOFF.md`

```markdown
# Session Handoff — [date]

## Status
[In progress | Blocked | Complete — be specific]

## Files Changed This Session
- [file path] — [what changed and why]

## Decisions Made
- [decision] — [rationale] — see DECISION_LOG.md [entry name]

## Blocked On
- [blocker] — [what is needed to unblock]

## Next Steps (in order)
1. [specific next action]
2. [specific next action]

## Danger Zones
- [anything partially implemented that could break if touched wrong]

## Gates Not Yet Passed
- [gate name] — [current status]
```

2. Run `mypy apps/` and `python manage.py check` — note errors in HANDOFF.md.
3. Tell the user: *"Context window is near capacity. I've written a handoff document at HANDOFF.md. Starting a new session with that document will restore full context. Should I continue or start fresh?"*

### 8.3 — Multi-session resume

At the start of any session on a project with a `HANDOFF.md`:
1. Read `HANDOFF.md` fully before reading any other file.
2. State the status from the handoff before asking what to do next.
3. Delete `HANDOFF.md` after the user confirms the context is correct.

---

## Summary

| Situation | Action |
|---|---|
| Session start | State context — stack, last verified state, current feature |
| New feature | PC2 define → confirm → implement → all gates |
| 3+ features | List sequence → confirm order → one at a time with gates |
| Gate fails | Fix before proceeding |
| Scope expands | State explicitly, get approval, update feature list |
| 30+ tool calls | Write decisions to files (PC8.1) |
| Context at capacity | Write HANDOFF.md, stop, inform user (PC8.2) |
| Session resumes | Read HANDOFF.md first (PC8.3) |
