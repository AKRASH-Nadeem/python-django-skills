# Theoretical Test Cases — Engineering Philosophy & Reasoning Protocol
# Full test run with failure analysis and fix tracking

---

## BATCH 1: Engineering Philosophy / APP_STATE.md (T001–T030)

| # | Scenario | Expected | Result | Fix |
|---|---|---|---|---|
| T001 | Agent starts a fresh Django project — does it create APP_STATE.md? | Yes — no structural change without it | ✅ PASS | Rule 4 triggers on any new app or package add |
| T002 | Agent adds Celery — does it update APP_STATE.md? | Yes — new external service + env vars | ✅ PASS | "New third-party package added" is a mandatory trigger |
| T003 | Agent fixes a serializer bug — does it update APP_STATE.md? | No — bug fix is not a trigger | ✅ PASS | Rule 4 explicitly excludes "bug fixes, view changes, serializer changes" |
| T004 | Agent adds a hardcoded `SECRET_KEY = "mykey"` in settings | Rule catches it — must be env var, no default | ✅ PASS | Rule 1 lists SECRET_KEY; sensitive values have NO default |
| T005 | Agent uses `if settings.DEBUG:` to switch email backends in a view | Wrong — DEBUG-branching belongs in settings only | ✅ PASS | Rule 2 explicitly: "`DEBUG` is for Django internals only" |
| T006 | Developer has Redis in prod but not locally. Agent adds Redis cache. | Agent must note this creates an env difference and update the matrix | ✅ PASS | Rule 4 matrix update + Rule 5: state assumption about Redis availability |
| T007 | Agent writes `DATABASE_URL = env("DATABASE_URL", default="sqlite:///db.sqlite3")` | Valid — sqlite is safe dev default | ✅ PASS | Rule 1: "Safe defaults only for non-sensitive operational values" |
| T008 | Agent writes `STRIPE_SECRET_KEY = env("STRIPE_SECRET_KEY", default="sk_test_xxx")` | WRONG — secret key must have no default | ❌ FAIL | Rule 1 says no default for secrets but example conflicts: test key looks "safe" |

---

### T008 FAIL ANALYSIS

**Problem:** `sk_test_xxx` looks like a safe test key, so a developer (and the agent) might argue it's acceptable. But the rule says sensitive values have NO default. The conflict arises because "test keys" exist in a grey zone.

**Invalidation of "test keys are fine as defaults":** Test keys can accidentally end up in production if someone forgets to set the env var. They also get committed to version control if the `.env` file isn't properly excluded. The rule must be absolute: API keys of any kind — test or live — have no hardcoded default.

**Fix applied to Rule 1 in engineering-philosophy.md:**
Add explicit note: "This includes test/sandbox API keys. `default=''` or `default=None` with a startup check is acceptable; hardcoded key strings are not."

---

| T009 | APP_STATE.md says "Cache: None" — agent adds Redis caching without updating it | Violation — agent must update the matrix | ✅ PASS | Rule 4: adding Redis is a mandatory update trigger |
| T010 | APP_STATE.md says "Auth: JWT" — agent sees session auth middleware in code | Contradiction — agent must flag discrepancy, not silently assume | ✅ PASS | Rule 4: "flags discrepancy to developer before proceeding" |
| T011 | Agent adds a new env var `FEATURE_X_ENABLED` but doesn't add it to APP_STATE.md matrix | Missing — env var adds are a mandatory trigger | ✅ PASS | Rule 4: "A setting that differs between environments is added" |
| T012 | APP_STATE.md doesn't exist yet and agent makes an architectural decision | Agent must create it first | ✅ PASS | Rule 4 says "agent creates and maintains" — implies creation on first trigger |
| T013 | Two feature flags both use `DEBUG` as a proxy — agent writes `if settings.DEBUG: feature_x = True` | Wrong — DEBUG is for Django internals only | ✅ PASS | Rule 2 covers this explicitly |
| T014 | Agent needs to add `django-debug-toolbar` — should it go in base.txt or development.txt? | development.txt only — Rule 2: dev-only tools in INSTALLED_APPS conditional on DEBUG | ✅ PASS | Rule 2 covers dev-only apps in INSTALLED_APPS |
| T015 | Agent adds `silk` profiler to `INSTALLED_APPS` unconditionally | Wrong — silk is development-only | ✅ PASS | Rule 2: dev-only middleware/apps belong in settings/development.py |
| T016 | Agent uses `os.environ.get("KEY")` instead of `django-environ` | Rule prefers django-environ pattern | ✅ PASS | Rule 1 says "Use django-environ or python-decouple" |
| T017 | Developer asks "what env vars does my app need?" | Agent reads APP_STATE.md "External Services" section and lists them | ✅ PASS | Rule 4 schema includes "External Services & Required Env Vars" |
| T018 | Developer switches from SQLite to PostgreSQL — does agent update APP_STATE.md? | Yes — "Database engine changed" is a mandatory trigger | ✅ PASS | Rule 4 lists "Database engine or major version changed" |
| T019 | APP_STATE.md "Run Instructions" section is missing — new developer can't run the app | Agent must populate it when creating or when instructions are not present | ❌ FAIL | Rule 4 schema shows the section but doesn't say WHEN to write it |

---

### T019 FAIL ANALYSIS

**Problem:** The rule defines the schema but doesn't tell the agent when to write the "Run Instructions" section. A new project will have an empty APP_STATE.md and a new developer won't know how to run it.

**Fix:** Add to the update triggers: "When APP_STATE.md is first created, the agent must populate Run Instructions with the current known setup steps."

---

| T020 | Agent adds Elasticsearch — does it add it to External Services with env vars? | Yes — "new external service integrated" is a trigger | ✅ PASS | Rule 4 explicitly lists Elasticsearch as an example |
| T021 | Dev uses `.env.example` — agent adds a new env var but doesn't add it to `.env.example` | Incomplete — new env vars must go in `.env.example` too | ❌ FAIL | Rules mention env vars but don't mention .env.example |

---

### T021 FAIL ANALYSIS

**Problem:** The engineering-philosophy.md doesn't mention `.env.example`. If the agent adds a new env var but doesn't add a placeholder to `.env.example`, the next developer to clone the repo won't know what's needed.

**Fix:** Add to Rule 1: "Every new env var must also be added to `.env.example` with a safe placeholder value and a comment describing what it's for."

---

| T022 | Agent writes `CONN_MAX_AGE = 60` in base settings without making it configurable | Borderline — this is a safe default for most setups | ✅ PASS | Rule 1: "Safe defaults only for non-sensitive operational values" — CONN_MAX_AGE qualifies |
| T023 | Environment matrix in APP_STATE.md has no "Staging" column — developer has no staging env | Valid — matrix should reflect the developer's actual environments | ✅ PASS | Rule 4 says "(fill in)" — agent adapts to the actual project |
| T024 | Agent researches before choosing a caching pattern because "Redis caching best practices" may have changed | Yes — version-sensitive configuration | ✅ PASS | Rule 3: "library API or settings names may have changed" is a trigger |
| T025 | Agent adds a new migration but doesn't update APP_STATE.md | Correct — migrations are not a trigger | ✅ PASS | Rule 4 excludes minor feature additions and code changes |
| T026 | APP_STATE.md says "Storage: Local filesystem" in production — agent catches this as a bad practice | Agent must flag this as a known constraint issue | ✅ PASS | Rule 4: "Known Constraints" section + agent reads APP_STATE.md first |
| T027 | Agent sees APP_STATE.md says "Task Queue: None" but developer asks for async email sending | Agent must flag that this will add Celery as a new dependency before implementing | ✅ PASS | Rule 5: any new infrastructure dependency is flagged immediately |
| T028 | Developer asks agent to "just hardcode the API URL for now" | Agent should still make it an env var but notes the developer's intent | ✅ PASS | Rule 1 is non-negotiable for URLs — agent applies it even if asked otherwise |
| T029 | APP_STATE.md is 500 lines long after many updates — it's become unreadable | Agent should keep APP_STATE.md concise — it's a snapshot, not a log | ❌ FAIL | Rules don't define a size discipline for APP_STATE.md |

---

### T029 FAIL ANALYSIS

**Problem:** Over time an agent might grow APP_STATE.md by adding but never pruning. A 500-line state file defeats the purpose of a quick-reference snapshot.

**Fix:** Add to Rule 4: "APP_STATE.md is a SNAPSHOT, not a log. When updating, replace outdated entries rather than appending. Target: under 100 lines. If it exceeds 150 lines, the agent must consolidate before adding more."

---

| T030 | Developer clones the project — APP_STATE.md tells them exactly what env vars to set and how to run the app | Goal achieved | ✅ PASS | Rule 4 schema covers this after T019 fix |

---

## BATCH 2: Reasoning Protocol (T031–T060)

| # | Scenario | Expected | Result | Fix |
|---|---|---|---|---|
| T031 | Simple CRUD endpoint for a User model | Agent skips protocol, just builds it | ✅ PASS | Protocol explicitly: "SKIP for standard CRUD endpoints" |
| T032 | Caching strategy needed for a product list endpoint | Agent runs protocol, identifies Redis vs DB-level, presents if genuine trade-off | ✅ PASS | Protocol Phase 3 covers caching options |
| T033 | Agent assumes Redis is available for caching without checking APP_STATE.md | Violation — Phase 1 says read APP_STATE.md first | ✅ PASS | Phase 1: "Read APP_STATE.md first" is explicit |
| T034 | Developer asks "should I use Celery or Django Q?" | Agent runs Phase 3, identifies trade-offs, gives recommendation | ✅ PASS | Phase 4/5 covers this — genuine trade-off → present options with recommendation |
| T035 | Agent presents 4 options for a simple caching decision | Wrong — max 2–3 options | ✅ PASS | Phase 5: "Maximum 2–3 options. More = decision paralysis." |
| T036 | Agent says "it depends" instead of giving a recommendation | Wrong — Phase 5 requires a recommendation | ✅ PASS | Phase 5: "'It depends' is not helpful" is explicit |
| T037 | Agent discovers a feature needs Redis but Redis isn't in APP_STATE.md | Agent must flag the new dependency before building | ✅ PASS | Anti-patterns: "Any new infrastructure dependency gets flagged immediately" |
| T038 | Developer says "just build it fast, I don't care about trade-offs" | Agent applies Phase 6 (self-invalidation) anyway — at minimum, validates its choice | ✅ PASS | Phase 6 is internal — not shown to developer unless it surfaces a real issue |
| T039 | Agent chooses Elasticsearch for a 5k-record table | Anti-pattern: premature optimization | ✅ PASS | Anti-patterns section explicitly covers this |
| T040 | Agent defers adding an index to a growing table "for later" | Anti-pattern: under-optimization of known bottleneck | ✅ PASS | Anti-patterns section covers this |
| T041 | Protocol Phase 2 says "ask if unknown" — developer gets asked 6 questions | Too many questions — agent must narrow to what's truly decision-blocking | ❌ FAIL | Protocol says "ask" but doesn't limit the number of questions |

---

### T041 FAIL ANALYSIS

**Problem:** "Ask the developer about ambiguities" could trigger a wall of questions that annoys the developer and breaks flow. The agent needs a limit.

**Fix:** Add to Phase 1/2: "Ask maximum 2 clarifying questions per task. Prioritize by: which unknown would most change the solution. If 3+ things are unknown, state your assumptions for the less-critical ones and ask only the most important."

---

| T042 | Agent knows the table will have 1M+ records from APP_STATE.md constraints — still presents DB-level solution as an option | Should filter options by known constraints before presenting | ✅ PASS | Phase 3 says "What scale is it appropriate for?" — ineligible options excluded |
| T043 | Agent uses Context7 for a simple `str.split()` question | Wrong — overkill for non-library standard Python | ✅ PASS | Rule 3 in engineering-philosophy.md defines when research is needed |
| T044 | Agent's Phase 6 self-invalidation finds a concurrency issue | Agent must address it before presenting the solution | ✅ PASS | Phase 6: "surfaces a new constraint → address it in the design" |
| T045 | Developer asks for a feature and the agent has 3 unknown constraints | Agent states assumptions for 2 less-critical ones, asks about the 1 most critical | ✅ PASS after T041 fix | T041 fix handles this |
| T046 | Agent recommends adding a background task for email sending — developer has no Celery | Agent checked APP_STATE.md first and knows this | ✅ PASS | Phase 1 forces reading APP_STATE.md before solution selection |
| T047 | Phase 3 requires research but agent skips it and uses training knowledge for a library | Violation — library-specific options require Context7 verification | ✅ PASS | Rule 3 engineering-philosophy.md is always-active and applies during Phase 3 |
| T048 | Agent presents trade-off options but the recommendation is opposite to what APP_STATE.md implies | Contradiction — recommendation must be consistent with known stack | ✅ PASS | Phase 5: "based on what I know about your stack and constraints" |
| T049 | Developer is building for 10 users. Agent suggests Kubernetes-level scaling solution | Anti-pattern: building infrastructure the app hasn't earned yet | ✅ PASS | Anti-patterns section: "Match the solution to the current scale" |
| T050 | Agent identifies a genuine trade-off between two approaches of similar complexity — both valid | Agent still gives a recommendation — picks based on maintainability or reversibility | ✅ PASS | Phase 5: "always give a recommendation" |
| T051 | Constraint inventory shows "team: 1 developer" — agent picks a solution requiring complex ops | Should factor team size into solution complexity | ❌ FAIL | Protocol lists "Team: familiarity level" as a constraint category but doesn't say how to use it |

---

### T051 FAIL ANALYSIS

**Problem:** The protocol says to note team size as a constraint but doesn't define how it affects solution choice. A 1-person team shouldn't be given Kubernetes + Kafka + Elasticsearch if a simpler stack achieves the same goal.

**Fix:** Add to Phase 4 (Clear Winner Test): "If the team is small (1–3 developers), prefer solutions with lower operational complexity even if a more powerful option exists. The maintenance cost of a complex solution over time outweighs its theoretical benefits for small teams."

---

| T052 | Agent's self-invalidation (Phase 6) finds the solution is fine — but the developer later reports a race condition | Phase 6 catches what it can; it's not omniscient | ✅ PASS | Phase 6 is best-effort, not a guarantee — that's honest |
| T053 | Agent applies full protocol for adding a new field to an existing model | Wrong — this is in the SKIP list | ✅ PASS | Protocol: "SKIP for adding a new model field" |
| T054 | Resource checklist asks "Does this run in a loop?" — yes → "Convert to bulk operation" | Agent correctly flags the loop | ✅ PASS | Resource checklist covers this |
| T055 | Agent calls external API in a loop synchronously | Resource checklist catches it: "calls an external API in a loop → batch" | ✅ PASS | Resource checklist: "external API in a loop → batch the calls" |
| T056 | Developer asks to choose between Postgres full-text search and Elasticsearch — no scale info | Agent asks one question: "Approximately how many records will this table have?" | ✅ PASS | Most critical unknown → ask. T041 fix limits to max 2 questions. |
| T057 | Agent's solution requires a new background worker process — developer didn't ask for that | Agent must flag the new infrastructure requirement, not silently add it | ✅ PASS | Anti-pattern: "Any new infrastructure dependency gets flagged immediately" |
| T058 | Developer explicitly says "I know the trade-offs, just use Redis" | Agent skips Phase 5 (developer has decided) and moves to Phase 6 then builds | ✅ PASS | Protocol: "only ask on genuine forks" — if developer already decided, no fork exists |
| T059 | Agent applies protocol but ignores the "Reversibility" constraint | Reversibility is in the constraint list — agent should weight it | ✅ PASS | Phase 2 constraint list includes "Reversibility: How hard is it to change this later?" |
| T060 | Agent recommends a stateful in-memory cache for a multi-worker setup | Assumption failure: in-memory cache doesn't survive across workers | ✅ PASS | Phase 2 concurrency: "Multiple workers?" + resource checklist |

---

## Round 2 — Apply All Fixes, Re-Test

| Fix | Scenarios Fixed | Verification |
|---|---|---|
| T008: test/sandbox keys also have no default | T008 | ✅ |
| T019: Run Instructions written when APP_STATE.md first created | T019 | ✅ |
| T021: New env vars also go in .env.example | T021 | ✅ |
| T029: APP_STATE.md size discipline (max 100 lines, consolidate at 150) | T029 | ✅ |
| T041: Max 2 clarifying questions; state assumptions for non-critical unknowns | T041 | ✅ |
| T051: Small team → prefer operationally simpler solution | T051 | ✅ |

**Re-test all 60 scenarios after fixes: 60/60 PASS ✅**

---

## Final Summary

```
Total scenarios:      60
Round 1 failures:      6  (T008, T019, T021, T029, T041, T051)
Round 2 after fixes:  60/60 PASS

Key improvements:
  - Test/sandbox API keys explicitly excluded from "safe defaults"
  - Run Instructions section populated on first APP_STATE.md creation
  - .env.example kept in sync with new env vars
  - APP_STATE.md size discipline (snapshot not log, max 100 lines)
  - Clarifying questions capped at 2 per task
  - Team size factored into solution complexity recommendation
```
