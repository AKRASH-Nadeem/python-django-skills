---
trigger: always_on
---

# Reasoning Protocol — Realistic, Factor-Aware Thinking

> These rules are ALWAYS ACTIVE. They govern how the agent
> reasons before implementing anything non-trivial.

---

## Core Problem This Rule Solves

AI agents fail not from lack of knowledge but from *assumption accumulation*:
each individual assumption seems reasonable, but stacked together they
produce a solution that only works in an ideal scenario.

This protocol forces constraint surfacing BEFORE choosing a solution.

---

## When to Apply

**ALWAYS apply for:**
- Any new architectural component (caching, queues, search, storage)
- Any choice between two or more valid approaches
- Any solution involving resource consumption (memory, DB connections, workers, external APIs)
- Any feature that scales with user growth
- Any feature that introduces new business logic, permissions, or data writes

**SKIP only for:**
- Standard CRUD endpoint following an *identical* existing pattern in the codebase
- Adding a new model field with no behaviour change
- Writing a test for existing logic
- Fixing a bug with a clear, isolated, single-line cause

**When in doubt: apply it.** The cost of applying unnecessarily is one extra minute. The cost of skipping is hours of wrong implementation.

---

## The Protocol — 9 Phases

### Phase 0: INTERROGATE THE REQUIREMENT

> Defined in `senior-dev-mindset.md`. Summary:
> Before reasoning about *how* to implement, ask whether *this* is the right solution.

Apply the Three Questions from `senior-dev-mindset.md`:
1. What is the actual problem (stripped of the stated solution)?
2. What assumption is embedded in this request?
3. Is the implementation complexity proportional to the problem?

Check the Requirement Smells table in `senior-dev-mindset.md`.
If a smell matches, raise the concern before proceeding to Phase 1.

---

### Phase 1: READ BEFORE ASSUMING

Before reasoning about a solution:
1. Read **DECISION_LOG.md** — what was decided and why; flag conflicts before proceeding
2. Read **APP_STATE.md** — what infrastructure already exists
3. Read **LIBRARY_LEDGER.md** — what is installed and what was rejected
4. Read relevant existing code — what patterns are already in place
5. Read requirements/settings — what constraints are defined

Only form assumptions about things NOT found in these sources.
Ask maximum 2 questions per task. Prioritise by impact.

---

### Phase 2: CONSTRAINT INVENTORY

List every real constraint. For each, state KNOWN (from code/config) or ASSUMED.

#### Infrastructure
- What's available? (Redis? S3? Elasticsearch? Celery?)
- Deployment target? (Docker? Heroku? bare VM?)

#### Scale
- How many users/requests/records now? In 6 months?
- Concurrency: Multiple workers? Multiple servers?
- Data volume: Records per table? Growth rate?

#### Latency
- User-facing or background? Acceptable response time?
- Is an async task acceptable for this feature?

#### Team
- Who maintains this? What's their familiarity?
- Reversibility: How hard is this to change later?

#### Security Lens *(new)*
- What user roles can reach this endpoint? Is object-level permission enforced?
- What input is trusted? Where is it validated (serializer, not view)?
- Does this expose PII or sensitive data? Is the response filtered by role?
- Is this endpoint rate-limited? Does it need HMAC/signature verification?
- Does this touch auth, tokens, or sessions? What is the token invalidation path?

#### AI Failure Mode Lens *(new)*
- Is business logic embedded in a view? → Must be in `services.py`
- Is validation inline in a view function? → Must be in serializer/form
- Is a permission check only on the frontend/client? → Backend must enforce
- Is an external API call synchronous in a request? → Must be offloaded to Celery
- Is there a missing test for an implicit "happy path only" assumption?
- Does this feature require context from previous sessions? → Ensure DECISION_LOG.md and APP_STATE.md cover it

#### Context & Session Continuity Lens *(new)*
- Does this feature span multiple sessions or produce architectural decisions?
- If yes: is the decision being written to DECISION_LOG.md before this session ends?
- Does this feature require Memvid recall of past preferences or constraints?
- Does Memvid MCP have relevant prior context? → `memvid_find` on shared.mv2 + backend.mv2 before building (see `mcp-servers.md`)

#### Observability & Analytics Lens *(new)*
- Are errors tagged with context (user_id, request_id, app label) for Sentry/logging?
- Are slow queries identified by django-silk or toolbar in dev?
- Are Celery task failures surfaced (Flower, Sentry, or email alert)?
- Is there a health check endpoint that exercises the critical path?

#### Data Privacy & GDPR Lens *(new)*
- No PII in `logging` calls, Sentry breadcrumbs, or Celery task args?
- Is there a data deletion path when a user requests account removal?
- Are personal fields encrypted at rest where required?
- Does this feature store data that crosses a jurisdictional boundary?

State ASSUMED constraints explicitly. Ask about UNKNOWN constraints that materially change the solution.

---

### Phase 3: SOLUTION OPTIONS

Identify professional-level solutions. Use Context7 to confirm current recommendations — do not rely on training knowledge for version-sensitive libraries.

For each option:
- What problem does it solve well?
- What does it cost? (infrastructure, complexity, maintenance)
- What does it fail at?
- What scale is it appropriate for?

---

### Phase 4: CLEAR WINNER TEST

Is one option clearly better given the known constraints?

**YES → Build it.** State why in one sentence. Do not present options for decisions with a clear professional answer.

**Small team rule**: For teams of 1–3, prefer solutions with lower operational complexity even if a more powerful option exists.

**NO → genuine trade-off exists** → Phase 5.

A genuine trade-off exists when:
- Two solutions have meaningfully different resource profiles (e.g., Redis required vs DB-only)
- Two solutions have meaningfully different maintenance complexity
- The right choice depends on a growth assumption the developer must confirm

---

### Phase 5: PRESENT OPTIONS (genuine trade-offs only)

```
DECISION NEEDED: [what the choice is about]

OPTION A — [short name]
  Best when: [scenario]
  Requires: [infrastructure or complexity cost]
  Limitation: [what it fails at]

OPTION B — [short name]
  Best when: [scenario]
  Requires: [infrastructure or complexity cost]
  Limitation: [what it fails at]

MY RECOMMENDATION: [Option A/B] because [one concrete reason].

Which would you like?
```

Rules:
- Max 2–3 options; always give a recommendation; base it on known constraints.
- **"It depends" is never an acceptable recommendation.** If you cannot decide, it means you are missing a constraint — ask for it. Do not present uncertainty as an answer.
- If the developer has already stated their preference, skip Phase 5 entirely — no fork exists.

---

### Phase 6: FAILURE SCENARIO INVENTORY (before writing code)

Adversarial questioning applied to this specific feature. Answer each concretely — not generically.

**Simultaneity** — Two actors trigger this at the same time. What specific data corruption results? → Needs transaction/lock defence + concurrent test.

**Second request** — Operation triggered twice before first completes. What specific duplicate outcome (double charge, double record)? → Needs idempotency + second-call no-op test.

**Dependency failure** — Each external dependency fails in turn (DB, cache, queue, external API, storage). What is the specific user-visible outcome? → Any "500 error" or "silent failure" answer means the implementation is incomplete.

**Boundary** — Exact threshold between valid and invalid for every input. What happens at max, max+1, min, zero? → Any undefined behaviour needs validation + boundary tests.

**Missing data** — A record this feature depends on doesn't exist at the moment needed. What specific FK, cache key, or file could be absent? → Needs guard + test.

**Authorization** — Who must not do this, and what stops them? Name the specific user, role, and object to block. Frontend checks are UX — not security. Backend enforces everything.

**Scale** — What breaks at 10× today's data? Name the specific query or task that produces N+1, unbounded memory, or missing pagination.

**Context loss** — If this feature spans multiple agent sessions, what decisions are not yet written to DECISION_LOG.md? What assumptions will the next session make incorrectly?

Each question that surfaces a named failure scenario must be addressed in the implementation and tracked in Phase 7.

---

### Phase 7: CONNECT FAILURES TO TESTS

For each failure scenario from Phase 6, either:
- Name the test that will verify the defence holds, OR
- Explain architecturally why this scenario cannot occur

A feature with unaddressed failure scenarios is incomplete. Apply `django-edge-case-testing` skill for the full thinking loop.

---

### Phase 8: RETAIN MEMORY / UPDATE RECORDS

If this task produced a new architectural decision, pattern, or constraint:
1. **Store** the memory via Memvid MCP (`memvid_put` tool) to the appropriate `.mv2` file.
2. If Memvid is unavailable, update `DECISION_LOG.md` (replace changed entries — do not append).
If no architectural decision was made, skip this phase.

---

### Phase 9: AGENTIC SELF-HEALING LOOP *(new)*

When tests fail after implementation:

```
Iteration 1: Read the exact test error. Identify root cause. Fix the code.
Iteration 2: If still failing — re-read the requirement. Am I solving the right problem?
Iteration 3: If still failing — STOP. Do not patch further.
```

After 3 failed iterations:
1. State: *"I've attempted 3 iterations on [test name] and it's still failing. The root cause appears to be [hypothesis]. This needs human review."*
2. Output: the exact failing assertion, your hypothesis about the root cause, what you tried.
3. Do not attempt a 4th iteration without user direction.

**Test integrity rule:** If any iteration "fixes" the failure by changing the test assertion to match the current (wrong) output, that is **not a fix** — it is evidence the code under test is wrong. Changing a test assertion is only valid when the requirement itself has changed and the developer confirms the new expectation. Never weaken a test to make it pass.

**Why:** Uncapped iteration loops produce patch-over-patch code that passes the test while violating the original design. 3 attempts is the professional signal that the problem is larger than the implementation.

---

## Resource Awareness Checklist

**Database:**
- N+1 queries? → `select_related`/`prefetch_related` + `assert_num_queries` test
- Multiple table writes? → `transaction.atomic()` + rollback test
- Large table without filtering? → pagination + index + scale test
- Runs in a loop? → convert to bulk operation

**Memory:**
- Queryset loaded into memory? → `.iterator()` for large sets
- Per-request state? → must be cleared after request
- Grows unbounded? → needs TTL or size limit

**External services:**
- Synchronous external API call in a view? → offload to Celery
- External API in a loop? → batch the calls
- Depends on a service that can go down? → needs fallback

**Cache:**
- Value that can become stale? → define invalidation trigger
- Per-user cache? → user ID in the cache key
- Many unique cache keys? → check memory pressure

---

## Anti-Patterns to Avoid

❌ "This will work fine" (without checking concurrency)
❌ "We can add caching later" (when the query pattern is known now)
❌ Presenting 5 options for a problem with a clear professional answer
❌ Building Redis-dependent solutions without flagging Redis as a new dependency
❌ Adding Elasticsearch for a 5k-record table
❌ Adding a message queue for a feature with 10 users
❌ Choosing a new architectural approach without recording why in `DECISION_LOG.md`
❌ Business logic in views or serializers
❌ Synchronous external API calls in request handlers
❌ Looping indefinitely on a failing test (see Phase 9 self-healing ceiling)
