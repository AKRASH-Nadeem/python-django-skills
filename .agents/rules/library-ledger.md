---
trigger: always_on
---

> LIBRARY LEDGER MANDATE: Every project has a `LIBRARY_LEDGER.md` at its root.
> The agent creates it on project init and updates it on every library add, upgrade,
> or removal. This file is the project's source of truth for what is installed, why,
> and what happened. It is committed to version control.

---

# Library Ledger Standards

---

## LL0. What This Is and Why

`versions.lock.md` (in `.agents/`) records the baseline versions for skill example
code snippets — it is an agent-internal reference.

`LIBRARY_LEDGER.md` (in the project root) is different. It:
- Lives inside the actual project, committed with the codebase
- Tracks every library the agent has installed, with full context
- Records Context7 queries used so the exact same lookup can be repeated
- Documents why alternatives were rejected
- Captures any dependency warnings or install issues encountered
- Serves as a handoff document — any developer or agent session can understand every dependency decision without asking

**The agent creates `LIBRARY_LEDGER.md` during project init (before any `pip install`) and
treats every install event as a ledger write.**

---

## LL1. Creating the Ledger (new project)

During project setup (TS1 from `tech-stack.md`), create the file before installing anything:

```bash
touch LIBRARY_LEDGER.md
```

Write the initial header:

```markdown
# Library Ledger

> Auto-maintained by the AI agent. Updated on every library add, upgrade, or removal.
> Do not edit manually — run changes through the agent so context is preserved.

## Stack profile
[Copy the approved stack profile from tech-stack.md TS1 here]

## Summary table

| Package | Version | Added | Feature | Req file | Status |
|---------|---------|-------|---------|----------|--------|

## Full history

[Entries appended below as libraries are installed]
```

---

## LL2. Ledger Entry Format

Every library addition produces one entry. Write it immediately after the install
succeeds — never batch entries.

```markdown
### [package-name]

| Field | Value |
|-------|-------|
| **Installed version** | [exact version from `pip show [package]`] |
| **Version pin in requirements** | [e.g. `Django==6.0.1` in base.txt] |
| **Requirements file** | [base.txt / development.txt / testing.txt / production.txt] |
| **Date** | [YYYY-MM-DD] |
| **Feature / task** | [the feature or task that required this library] |
| **Why this library** | [1-2 sentences — what problem it solves] |
| **Alternatives considered** | [what was evaluated and why rejected] |
| **Context7 query used** | `[exact query string passed to Context7]` — **MUST NOT be blank** |
| **Install command** | `[exact command run]` |
| **Dependencies added** | [list any new transitive deps, or "none significant"] |
| **Env vars required** | [list any new env vars this library needs, or "none"] |
| **Issues during install** | [any warnings, conflicts, or workarounds, or "none"] |
| **Status** | Active |
```

**Example entry:**

```markdown
### djangorestframework

| Field | Value |
|-------|-------|
| **Installed version** | 3.15.2 |
| **Version pin in requirements** | `djangorestframework==3.15.2` |
| **Requirements file** | base.txt |
| **Date** | 2026-04-01 |
| **Feature / task** | Project setup — all REST API endpoints |
| **Why this library** | Provides serializers, ViewSets, authentication, and pagination for DRF. The standard for Django REST APIs — battle-tested and Django 6 compatible. |
| **Alternatives considered** | FastAPI — no Django ORM integration; Ninja — smaller ecosystem, fewer features at this team size. |
| **Context7 query used** | `django rest framework 3.15 installation setup django 6` |
| **Install command** | `pip install djangorestframework==3.15.2` |
| **Dependencies added** | none |
| **Issues during install** | none |
| **Status** | Active |
```

---

## LL3. Updating the Summary Table

After every entry, update the summary table at the top of the file:

```markdown
| Package | Version | Added | Feature | Req file | Status |
|---------|---------|-------|---------|----------|--------|
| Django | 6.0.1 | 2026-04-01 | Project scaffold | base.txt | Active |
| djangorestframework | 3.15.2 | 2026-04-01 | REST API | base.txt | Active |
| celery | 5.4.0 | 2026-04-05 | Email background tasks | base.txt | Active |
```

The table is the quick-scan view. The full history section has the context.

---

## LL4. Upgrade Entries

When a library is upgraded (major or minor), add an upgrade entry below the original
entry — do not overwrite it:

```markdown
### [package-name] — upgrade

| Field | Value |
|-------|-------|
| **Previous version** | [old version] |
| **New version** | [new version] |
| **Date** | [YYYY-MM-DD] |
| **Reason for upgrade** | [bug fix / security / feature needed / Django compatibility] |
| **Breaking changes** | [list API changes that required code updates, or "none"] |
| **Context7 query used** | `[package] v[N] to v[M] migration` |
| **Files changed** | [list files updated for the migration] |
| **Status** | Active |
```

Update the version in the summary table.

---

## LL5. Removal Entries

When a library is removed, do not delete the history entry. Mark it as removed:

```markdown
### [package-name] — removed

| Field | Value |
|-------|-------|
| **Removed version** | [version that was removed] |
| **Date** | [YYYY-MM-DD] |
| **Reason** | [replaced by X / feature removed / no longer needed] |
| **Replaced by** | [new library or approach, or "nothing"] |
| **Files changed** | [list files updated] |
```

Update the summary table status to `Removed`.

---

## LL6. Context7 Integration

Context7 is called for two purposes in the library flow:

### LL6.1 Before installing — get the current install command

Never use training memory for install commands. Package names, CLI flags, and
setup steps change between versions.

```
Context7 query pattern for installation:
  "[package-name] django installation setup"

Examples:
  "celery django redis installation setup"
  "django-storages s3 boto3 setup"
  "drf-spectacular django rest framework installation"
  "django-allauth headless jwt setup"
  "psycopg3 django database setup"
```

Record the exact query string in the ledger entry (`Context7 query used` field).

### LL6.2 Before writing code — get the current API

Even after installing, call Context7 before writing any code that uses the library.

```
Context7 query pattern for usage:
  "[package-name] [specific feature or API]"

Examples:
  "celery v5 task retry max_retries countdown"
  "simplejwt token obtain pair custom claims"
  "drf-spectacular extend_schema request response"
  "django-storages s3 signed url private storage"
```

This is separate from the ledger — it happens every time you write code using
the library, not just on install.

### LL6.3 For upgrades — get the migration guide

```
Context7 query pattern for migrations:
  "[package-name] v[N] to v[M] migration"
  "[package-name] v[N] breaking changes"

Examples:
  "celery v5 migration from v4 breaking changes"
  "django-allauth 0.60 migration headless api changes"
  "django-storages 1.14 STORAGES dict migration"
```

Record the query in the upgrade ledger entry.

---

## LL7. When the Ledger Doesn't Exist Yet

If a user asks the agent to work on an existing project that has no `LIBRARY_LEDGER.md`:

1. State: *"This project doesn't have a LIBRARY_LEDGER.md yet. I'll create one now and backfill the current dependencies."*
2. Read `requirements/base.txt` (and other requirements files) to get current deps
3. Create the file with the header (LL1)
4. For each installed dependency, add a backfill entry:

```markdown
### [package-name] — backfilled

| Field | Value |
|-------|-------|
| **Installed version** | [from requirements file] |
| **Requirements file** | [base.txt / development.txt / etc.] |
| **Date** | Unknown (pre-ledger) |
| **Feature / task** | Unknown (pre-ledger) |
| **Why this library** | [agent's best assessment based on what the library does] |
| **Status** | Active |
```

5. Going forward, all new installs use the full LL2 format.

---

## LL8. The Ledger and APP_STATE.md

The ledger records *what happened*. `APP_STATE.md` records *what is true now*.

When a library is added:
- `APP_STATE.md` Tech Stack section is updated with the new library (one-liner)
- `LIBRARY_LEDGER.md` gets the full context entry

When a library is removed:
- `APP_STATE.md` Tech Stack section is updated (remove the entry)
- `LIBRARY_LEDGER.md` gets a removal entry (never delete history)

The ledger entry is written after approval is received (TS5 in `tech-stack.md`),
after the install succeeds, and after `mypy` and `python manage.py check` pass.

---

## Summary

| Event | Ledger action |
|---|---|
| New project init | Create `LIBRARY_LEDGER.md` with header before any installs |
| Library installed | Write full LL2 entry + update summary table |
| Library upgraded | Write upgrade entry + update summary table version |
| Library removed | Write removal entry + update summary table status |
| Existing project, no ledger | Backfill from requirements files using LL7 format |
| Install command needed | Context7 query first — record exact query in ledger |
| Writing code for a library | Context7 query for current API — every time, not just on install |
