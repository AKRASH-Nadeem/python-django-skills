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

**SKIP only for:**
- Standard CRUD endpoint following an *identical* existing pattern in the codebase
- Adding a new model field with no behaviour change
- Writing a test for existing logic
- Fixing a bug with a clear, isolated, single-line cause

**When in doubt: apply it.** The cost of applying unnecessarily is one extra minute. The cost of skipping is hours of wrong implementation.

---

## The Protocol — 8 Phases

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
3. Read relevant existing code — what patterns are already in place
4. Read requirements/settings — what constraints are defined

Only form assumptions about things NOT found in these sources.
Ask maximum 2 questions per task. Prioritise by impact.

---

### Phase 2: CONSTRAINT INVENTORY

List every real constraint. For each, state KNOWN (from code/config) or ASSUMED.

- Infrastructure: What's available? (Redis? S3? Elasticsearch?)
- Scale: How many users/requests/records now? In 6 months?
- Concurrency: Multiple workers? Multiple servers?
- Data volume: Records per table? Growth rate?
- Latency: User-facing or background? Acceptable response time?
- Team: Who maintains this? What's their familiarity?
- Reversibility: How hard is this to change later?

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

Rules: max 2–3 options; always give a recommendation; base it on known constraints.

---

### Phase 6: FAILURE SCENARIO INVENTORY (before writing code)

Adversarial questioning applied to this specific feature. Answer each concretely — not generically.

**Simultaneity** — Two actors trigger this at the same time. What specific data corruption results? → Needs transaction/lock defence + concurrent test.

**Second request** — Operation triggered twice before first completes. What specific duplicate outcome (double charge, double record)? → Needs idempotency + second-call no-op test.

**Dependency failure** — Each external dependency fails in turn (DB, cache, queue, external API, storage). What is the specific user-visible outcome? → Any "500 error" or "silent failure" answer means the implementation is incomplete.

**Boundary** — Exact threshold between valid and invalid for every input. What happens at max, max+1, min, zero? → Any undefined behaviour needs validation + boundary tests.

**Missing data** — A record this feature depends on doesn't exist at the moment needed. What specific FK, cache key, or file could be absent? → Needs guard + test.

**Authorization** — Who must not do this, and what stops them? Name the specific user, role, and object to block. Frontend checks are UX — not security.

**Scale** — What breaks at 10× today's data? Name the specific query or task that produces N+1, unbounded memory, or missing pagination.

Each question that surfaces a named failure scenario must be addressed in the implementation and tracked in Phase 7.

---

### Phase 7: CONNECT FAILURES TO TESTS

For each failure scenario from Phase 6, either:
- Name the test that will verify the defence holds, OR
- Explain architecturally why this scenario cannot occur

A feature with unaddressed failure scenarios is incomplete. Apply `django-edge-case-testing` skill for the full thinking loop.

---

### Phase 8: UPDATE DECISION_LOG.md

If this task produced a new architectural decision, update `DECISION_LOG.md`.
Replace changed entries — do not append.
If no architectural decision was made, skip this phase.

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
