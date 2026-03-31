---
name: django-edge-case-testing
description: >
  Apply this skill for ALL feature implementation and test writing in Django.
  Teaches HOW to think about failure — not what failures to look for.
  Activates alongside django-testing-quality whenever a feature is being
  designed, implemented, or tested.
  Trigger on: any new feature, service function, API endpoint, Celery task,
  model change, "write tests", "add tests", "what could go wrong",
  "implement this", "how should I test this".
---

# Django — Failure-First Reasoning & Edge-Case Testing Philosophy

> FIRST: Use Context7 `/websites/djangoproject_en_6_0` to verify Django
> transaction, signal, and async patterns before writing any test infrastructure.

---

## Why This Skill Exists

Code is written for what should happen.
Tests must be written for what will actually happen — which is different.

The gap between intent and reality is where production breaks at 2am.
This skill closes that gap by making failure-scenario thinking
automatic — before any code is written, not after.

A test suite that only verifies the happy path does not test the
software. It tests the developer's optimism.

---

## The Failure-First Mindset

Before writing implementation code, the agent adopts this posture:

> "This feature will break under some condition. My job is to discover
> that condition before the user does, and prove my defence works."

This mindset drives both design and testing. It is applied through a
series of adversarial questions asked against the agent's own plan.
The questions are not a checklist — they are a thinking process that
generates the specific scenarios for THIS feature in THIS codebase.

---

## Phase 1 — Adversarial Questioning (before writing code)

These questions must be applied to every feature. The answers are not
generic — they name the specific actors, data, and timing for this
particular implementation. If an answer is generic ("add a lock"),
the question has not been answered — keep going until it is specific.

### The Simultaneity Question
What if two independent actors trigger this operation at the same moment
when the design assumed they would not?

Think about every place the feature reads data and then writes based on
what it read. Think about every condition check followed by an action.
The gap between the read and the write is a race window. What data
corruption or double-processing becomes possible in that window? The
answer to this determines whether `select_for_update()`, unique
constraints, atomic transactions, or idempotency keys are needed — and
which specific one, for which specific operation.

### The Second-Request Question
What if this exact operation is triggered twice, immediately after the
first succeeded?

Users double-click. Networks retry on timeout. Webhook providers deliver
twice. Payment processors confirm twice. Every operation that creates or
modifies data must produce the same observable outcome on the second
invocation as on the first. If it does not, the design needs an
idempotency mechanism. The question surfaces what that mechanism should
be and where it belongs — not whether one is needed.

### The Dependency Failure Question
What if every dependency works correctly except one, and it fails at the
worst possible moment?

Work through each dependency the feature uses: the database, the cache,
the queue, the file storage, the external APIs. For each one, ask: what
happens to the user and to the data if this dependency is unavailable,
slow, or returns malformed data at the moment the feature needs it? The
answer defines the required error handling and fallback behaviour. A
feature that has not answered this question for each dependency is not
production-ready, regardless of how clean the happy path is.

### The Boundary Question
What is the exact threshold between valid and invalid for every piece
of data this feature accepts, and what happens on both sides?

Not just "what is valid" — the boundaries are where validation logic
has the most bugs. What is the maximum? What happens at maximum+1?
What is the minimum? What happens at zero, at negative, at null?
What is the longest string? What happens at max_length, at max_length+1?
The question is not answered until a specific threshold is identified
and both sides of it are covered by a test that would catch a regression.

### The Missing Data Question
What if the data this feature expects to exist has been deleted or was
never created, at the moment the feature tries to use it?

Think about every foreign key, every cache read, every session value,
every file path. A product can be deleted between when a user adds it
to a cart and when they check out. A user record can be deleted between
when a Celery task is enqueued and when it runs. A cache entry can
expire between when a request starts and when it is read. Every one of
these is a production scenario, not a theoretical one. The question
surfaces whether each lookup has a defined, tested behaviour when it
returns nothing.

### The Authorization Boundary Question
Who must not be able to do this, and what stops them?

Authentication (is the user logged in) and authorisation (can THIS user
touch THIS specific object) are different problems. A queryset that is
not filtered to `user=request.user` is an authorisation bug. A test that
only verifies a logged-in user can access their own data does not test
authorisation — it tests authentication. The question must surface the
specific object types, the specific user roles, and the specific actions
that must be blocked, then verify each is blocked.

### The Scale Question
What breaks in this design when the data volume is 10x or 100x what it
is today?

A queryset without an index takes milliseconds on 100 rows and seconds
on 100,000. A list endpoint without pagination is a denial-of-service
vector. A Celery task that opens a new connection per item is efficient
at 10 items and catastrophic at 100,000. The question is not about
premature optimisation — it is about whether the design makes assumptions
about data volume that will eventually be false. Those assumptions
produce the production incidents that are hardest to fix.

---

## Phase 2 — Scenarios to Test Cases

The adversarial questions produce scenarios. A scenario is a test case
when it can be stated as:

> "Given [specific system state], when [specific action is taken],
> then [specific outcome must be true and specific wrong outcome must not]."

If the scenario cannot be stated this way, it is not concrete enough.
Keep asking the adversarial question until it produces a scenario that
is this specific.

The test case must fail before the defensive code is written and pass
after it. If it passes before any defence exists, it is testing nothing.

---

## Phase 3 — Verifying the Tests Work

For every defensive mechanism added to the implementation:

1. Write the test that proves the defence holds
2. Temporarily remove or break the defensive code
3. Confirm the test now fails
4. Restore the defence
5. Confirm the test passes again

This loop — write, break, verify failure, restore, verify pass — is
the only proof that a test actually tests what it claims to. It takes
minutes and prevents years of false confidence.

If step 3 does not produce a failing test, the test is not testing the
defence. Rewrite the test until step 3 fails.

---

## Phase 4 — The Test Coverage Standard

Coverage percentage measures which lines were executed. It does not
measure whether the right scenarios were tested.

A service function's tests are complete when every adversarial question
from Phase 1 has been asked, answered with a specific scenario, and that
scenario has a test that would catch a regression. Not when coverage is
100%. When the questions have been answered.

An endpoint's tests are complete when: an unauthorised user cannot
access it, an invalid payload produces a useful error, a boundary-
violating payload is rejected, a valid payload produces the correct
response, and the correct user cannot access another user's data.
Each of these must have a test that would catch a regression.

A Celery task's tests are complete when: the task handles a missing
target object, handles its external dependencies failing, produces the
same result when run twice, and fires after the DB transaction commits
rather than before. Each must have a test.

---

## Phase 5 — Asking Context7 for Real-World Patterns

When the adversarial questions surface a scenario the agent does not
know how to defend against, the right response is not to guess. Use
Context7 to find the current recommended Django pattern for that
specific situation.

Query format: "[scenario type] Django [version] [specific pattern]"

Examples of queries this skill generates:
- "Django 6 select_for_update concurrent writes"
- "Django atomic transaction idempotency"
- "Celery on_commit task dispatch before commit bug"
- "Django test race condition TransactionTestCase threading"
- "pytest-django assert_num_queries N+1 detection"
- "Django signal test verify deferred to celery"

The adversarial questions generate the queries. Context7 provides the
current implementation patterns. The combination produces defences that
are both correct and up-to-date.

---

## Relationship to django-testing-quality

`django-testing-quality`: how to set up tests, write fixtures, configure
pytest, use factory_boy, structure test files.

`django-edge-case-testing`: what to test and how to think about whether
enough has been tested.

Both apply on every feature. They are not alternatives — they answer
different questions.
