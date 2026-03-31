# Reasoning Protocol — Realistic, Factor-Aware Thinking

> These rules are ALWAYS ACTIVE. They govern how the agent
> reasons before implementing anything non-trivial.

---

## Core Problem This Rule Solves

AI agents fail not from lack of knowledge but from *assumption accumulation*:
each individual assumption seems reasonable, but stacked together they
produce a solution that only works in an ideal scenario that doesn't exist.

Example failure chain:
- Assumes infinite memory → picks an in-memory solution
- Assumes single server → skips distributed locking
- Assumes small dataset → skips pagination on a query
- Assumes no concurrent writes → skips transaction safety

This protocol forces the agent to surface constraints BEFORE
choosing a solution, not after.

---

## When to Apply This Protocol

ALWAYS apply for:
- Any new architectural component (caching, queues, search, storage)
- Any choice between two or more valid implementation approaches
- Any solution that involves resource consumption (memory, DB connections,
  background workers, external API calls)
- Any feature that will scale with user growth

SKIP for (just build it well):
- Standard CRUD endpoints with a clear serializer
- Adding a new model field
- Writing a test
- Fixing a bug with a clear cause

---

## The Protocol — 6 Phases

### Phase 1: READ BEFORE ASSUMING

Before reasoning about a solution:
1. Read APP_STATE.md (what infrastructure already exists?)
2. Read relevant existing code (what patterns are already in place?)
3. Read requirements/settings (what constraints are already defined?)

Only form assumptions about things NOT found in these sources.
If clarification is needed, ask maximum 2 questions per task.
Prioritise by impact: which unknown would most change the solution?
For less-critical unknowns, state your assumption explicitly.

### Phase 2: CONSTRAINT INVENTORY

List every real constraint that applies. For each constraint, state
whether it is KNOWN (from code/config/APP_STATE.md) or ASSUMED.

Constraint categories to consider:
- Infrastructure: What's actually available? (Redis? S3? Elasticsearch?)
- Scale: How many users/requests/records now? In 6 months?
- Concurrency: Multiple workers? Multiple servers?
- Data volume: Records per table? Growth rate?
- Latency: User-facing or background? Acceptable response time?
- Cost: Cloud resource cost sensitivity?
- Team: Who maintains this? What's their familiarity level?
- Reversibility: How hard is it to change this later?

For any ASSUMED constraint, state it explicitly.
For any UNKNOWN constraint that materially changes the solution, ask.

### Phase 3: SOLUTION OPTIONS

Identify the professional-level solutions that apply.
For library-specific options, use Context7 or web search to confirm
current recommendations — do not rely on training knowledge for
fast-moving libraries or anything version-sensitive.

For each option, answer:
- What problem does it solve well?
- What does it cost? (infrastructure, complexity, maintenance)
- What does it fail at? (limits, known failure modes)
- What scale is it appropriate for?

### Phase 4: CLEAR WINNER TEST

Is one option clearly better given the known constraints?

If YES: Build it. State why it's the right choice in one sentence.
Do not present options to the developer for decisions that have
a clear professional answer — that wastes their time.

Additionally: if the team is small (1–3 developers), prefer solutions
with lower operational complexity, even if a more powerful option
exists. Maintenance cost over time outweighs theoretical power gains
for small teams.

If NO — genuine trade-off exists: Move to Phase 5.

A genuine trade-off exists when:
- Two solutions have meaningfully different resource profiles
  (e.g., Redis required vs DB-only)
- Two solutions have meaningfully different maintenance complexity
- The right choice depends on a growth assumption the developer
  must confirm (e.g., "Will this table exceed 1M rows?")
- The developer's priorities determine the answer
  (e.g., fastest to ship vs most scalable)

### Phase 5: PRESENT OPTIONS (only on genuine trade-offs)

Format for presenting options to the developer:

```
DECISION NEEDED: [what the choice is about]

OPTION A — [short name]
  Best when: [scenario]
  Requires: [infrastructure or complexity cost]
  Limitation: [what it fails at or scales poorly for]

OPTION B — [short name]
  Best when: [scenario]
  Requires: [infrastructure or complexity cost]
  Limitation: [what it fails at or scales poorly for]

MY RECOMMENDATION: [Option A/B] because [one concrete reason
based on what I know about your stack and constraints].

Which would you like to proceed with?
```

Rules for presenting options:
- Maximum 2–3 options. More options = decision paralysis.
- Always give a recommendation. "It depends" is not helpful.
- The recommendation must be based on known constraints,
  not a hedge. If truly unknown, state what would change it.
- Do not present options for decisions that have a clear
  professional standard — just apply the standard.

### Phase 6: FAILURE SCENARIO INVENTORY (before writing code)

After choosing a solution, run this inventory before writing any
implementation code. This is not a checklist — it is adversarial
questioning applied to the specific feature. Each question must be
answered concretely for this feature, not generically.

**Ask about each of these failure axes:**

Simultaneity — what if two actors trigger this at the same time?
Name the specific actors and the specific data corruption that results.
If an answer exists, the implementation needs a defence and the tests
need to prove it holds under concurrent execution.

Second request — what if this exact operation is triggered twice?
Name the specific duplicate outcome (double charge, double record,
double notification). If an answer exists, the implementation needs
idempotency and the tests need to verify the second call is a no-op.

Dependency failure — what if each external dependency fails in turn?
For each one (DB, cache, queue, external API, file storage), name the
specific user-visible outcome when it is unavailable or returns bad data.
If the answer is "a 500 error" or "a silent failure", the implementation
is incomplete before it is written.

Boundary — what is the exact threshold between valid and invalid for
every input? Name the specific value at max, at max+1, at min, at zero.
If any of these produce undefined behaviour, the implementation needs
validation and the tests need to cover both sides of each boundary.

Missing data — what if a record this feature depends on does not exist
at the moment it is needed? Name the specific FK, cache key, or file
that could be absent. If the answer is "an unhandled DoesNotExist",
the implementation needs a guard and the tests need to prove it.

Authorization — who must not be able to do this, and what stops them?
Name the specific user, role, and object that must be blocked.
If the answer relies on "the frontend won't let them", the
implementation needs object-level permission checks.

Scale — what breaks when the data volume is 10x today's? Name the
specific query, the specific endpoint, or the specific task that
produces N+1 queries, unbounded memory, or missing pagination.

**Outcome of this phase:**
Each question that produces a named failure scenario must be addressed
in the implementation design and tracked for test coverage in Phase 7.
Questions that produce "no issue" can be documented as such.
No question can be skipped.

### Phase 7: CONNECT FAILURES TO TESTS

The failure scenarios from Phase 6 are not design notes — they are
test obligations. Before considering implementation complete:

State each failure scenario identified in Phase 6.
For each one, either:
- Name the test that will verify the defence holds, OR
- Explain why this scenario cannot occur (with evidence from the code)

If a failure scenario has no corresponding test and no architectural
reason it cannot occur, the feature is incomplete. Tests written after
the fact that are not rooted in a specific failure scenario are padding.

Apply `django-edge-case-testing` skill for the full thinking process
and test verification loop.

---

## Resource Awareness Checklist

For any solution that consumes infrastructure resources, verify:

Database:
- Does this add N+1 queries? → Requires select_related/prefetch + assert_num_queries test
- Does this write to multiple tables? → Requires transaction.atomic() + rollback test
- Does this read a large table without filtering? → Requires pagination + index + scale test
- Does this run in a loop? → Convert to bulk operation

Memory:
- Does this load a queryset into memory? → Use .iterator() for large sets
- Does this store per-request state? → Must be cleared after request
- Does this grow unbounded? → Requires TTL or size limit

External services:
- Does this call an external API synchronously? → Offload to Celery
- Does this call an external API in a loop? → Batch the calls
- Does this depend on a service that can go down? → Requires fallback

Cache:
- Does this cache a value that can become stale? → Define invalidation trigger
- Does this cache per-user? → User ID must be part of the cache key
- Does this generate many unique cache keys? → Check memory pressure

---

## Anti-Patterns the Agent Must Avoid

IDEAL WORLD REASONING:
❌ "This will work fine" (without checking concurrency)
❌ "We can add caching later" (when the query pattern is known now)
❌ "This is fast enough" (without knowing the data volume)
✅ State the scale assumption explicitly. If unknown, ask.

ANALYSIS PARALYSIS:
❌ Presenting 5 options for a problem with a clear answer
❌ Asking the developer to choose between two things that are
   both correct and the difference is minor
✅ Apply professional judgment. Ask only on genuine forks.

ASSUMPTION HIDING:
❌ Building a Redis-dependent solution without noting Redis is required
❌ Using a cloud-only service without noting it doesn't work locally
✅ Any new infrastructure dependency gets flagged immediately and
   added to APP_STATE.md.

PREMATURE OPTIMIZATION:
❌ Adding Elasticsearch for a 5k-record table
❌ Adding a message queue for a feature with 10 users
✅ Match the solution to the current scale. Build with the
   right architecture to grow into, but don't build infrastructure
   the application hasn't earned yet.

UNDER-OPTIMIZATION OF KNOWN BOTTLENECKS:
❌ Using a synchronous HTTP call in a view when the endpoint is
   already identified as high-traffic
❌ Loading 10k records into memory when pagination is standard
✅ Apply optimizations where the constraint is already visible.
   Do not defer obvious performance work "for later."
