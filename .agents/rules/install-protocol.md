---
trigger: model_decision
description: Load when installing/upgrading packages, skill activates with non-empty requires:, "module not found" errors, requirements files changed, or new project scaffolding. Skip for bug fixes, refactors, view changes, or code reviews.
---

> **Two sources of truth for libraries:**
> - `versions.lock.md` (in `.agents/`) — baseline versions for skill example code. Agent-internal only.
> - `LIBRARY_LEDGER.md` (in the project root) — every library the agent has installed, why, and what happened. Lives in the codebase, committed to git.
>
> **Context7 is the source of truth for install commands and API usage. Never use training memory.**

---

# Install Protocol — Python / Django

---

## IP1. Context7 — Always First

Before writing any install command or library code, call Context7.

**Why:** Package names, CLI flags, and setup steps change between versions. Training memory may reflect a version that is 6–18 months old. Context7 gives the current docs.

**Exempt from Context7:** Python standard library, stable Django ORM basics. All external packages: Context7 first.

### Query patterns

```
Installation:  "[package] django installation setup"
               "celery django redis setup"
               "drf-spectacular django rest framework installation"
               "django-storages s3 boto3 setup"

API usage:     "[package] [specific feature]"
               "celery retry strategies beat periodic"
               "simplejwt token refresh sliding token"
               "drf-spectacular extend_schema decorator"

Migration:     "[package] v[N] to v[M] migration"
               "[package] v[N] breaking changes"
```

Record the exact query string — it goes into `LIBRARY_LEDGER.md`.

---

## IP2. Skill Activation Pre-Check

When a skill activates, check its required packages before writing any code:

### Step 1 — Check what is installed

```bash
# Check in requirements files
grep "[package-name]" requirements/*.txt pyproject.toml 2>/dev/null

# Or with pip
pip show [package-name] 2>/dev/null | grep Version
```

### Step 2 — Compare against skill's `requires:` list and `versions.lock.md`

| Installed state | Action |
|-----------------|--------|
| Installed at matching major | ✅ Proceed |
| Installed at higher minor/patch | ✅ Proceed — semver compatible |
| Installed at different major | ⚠️ Version mismatch — see IP3 |
| Not installed | Call Context7 → install → write ledger entry |

### Step 2.5 — Auto Context7 query on version mismatch

If installed major ≠ skill baseline major:

```
1. Get exact installed version:
   pip show [package-name] | grep Version

2. Form the Context7 query:
   "[package-name] v[INSTALLED_MAJOR] [the feature being implemented]"

   Examples:
     project has celery@5.x, skill expects v6:
     → "celery v5 retry strategies periodic beat"
     project has drf@3.x, skill expects v4:
     → "django rest framework v3 serializer viewset"

3. Call Context7 with that exact query string.
4. Replace skill example API calls with Context7 output.
5. When writing code, state: "Adapting to v[X] via Context7 (skill baseline: v[Y])"
```

### Step 2.75 — Pre-install validation (NON-NEGOTIABLE)

Before running `pip install`, run ALL of these checks. **Do not skip any step**, even under time pressure.

```bash
# Check maintenance health (last release date)
pip index versions [package-name] 2>/dev/null | head -5

# Check for known vulnerabilities — NON-NEGOTIABLE
pip-audit --requirement requirements/base.txt

# Check package size (PyPI download size indicator)
pip install --dry-run [package-name] 2>&1 | grep -i "would install"
```

**CVE response protocol:**

| pip-audit severity | Action |
|---|---|
| Critical / High | **STOP.** Do not install. Report exact CVE ID, affected package, and severity to the user. Propose alternative or patched version. |
| Medium | Report to user with recommendation. Proceed only if user confirms acceptable risk. |
| Low | Note in LIBRARY_LEDGER.md. Proceed with install. |
| None | Proceed normally. |

Never ignore a Critical/High CVE. If the vulnerable package is a transitive dependency, find the parent package that pulls it in and check for a version that resolves the CVE.

### Step 3 — Install missing packages via Context7

```
1. Call Context7: "[package] django installation"
2. Follow the docs exactly
3. Use version ranges from versions.lock.md as a floor
4. Run the install command
5. Verify: pip show [package-name]
6. Write the ledger entry
```

```bash
# Confirm install succeeded AND version matches expectation
pip show [package-name]
```

**Version verification after install:**
If `pip show` returns a version different from what was requested (e.g., you ran `pip install package==2.0` but `pip show` reports `1.9.3`), **STOP**. The install did not produce the expected version. Report the discrepancy and investigate (pip resolver conflict, cached wheel, etc.) before writing any code.

### Step 4 — Write the ledger entry

After every successful install, update `LIBRARY_LEDGER.md`. See `library-ledger.md` LL2 format.

### Step 4.5 — Check for new env vars

After installing a library, check whether it requires new environment variables:
1. Review the Context7 setup docs for required settings (e.g., `CELERY_BROKER_URL`, `STRIPE_SECRET_KEY`)
2. For every new env var required:
   - Add it to `.env.example` with a comment explaining its purpose
   - Add it to `APP_STATE.md` External Services section
   - Add the env var name to the ledger entry's "Dependencies added" field
3. If the library requires a new external service (Redis, S3, etc.), flag it as a new infrastructure dependency before proceeding.

### Step 5 — Only write code after confirming the package is installed

If a package fails to install: **stop and report the exact error to the user**. Never write code that imports an unconfirmed package.

### Step 6 — Verify imports resolve after writing code

```bash
# Type-check the whole project
mypy apps/ --ignore-missing-imports  # or pyright

# Run the test suite to confirm imports resolve
python -m pytest --collect-only -q
```

Zero mypy errors and zero collection failures = imports are valid.

---

## IP3. Version Mismatch

If a project has a package at a different major than the skill's `requires:`:

1. **State the mismatch clearly:**
   > "This skill requires `celery@^6` but the project has `celery@^5` installed."

2. **Do not assume the skill's examples are compatible.** The API may have changed significantly.

3. **Search the migration guide via Context7:** `[package] v[N] to v[M] migration`

4. **Assess the blast radius:**
   - Straightforward (few usages, no breaking changes in used APIs) → upgrade and note what changed
   - Affects many files → stop and present the proposal to the user before touching anything

5. **Write an upgrade ledger entry** after the migration completes.

---

## IP4. Project Setup Order

When initialising a new project, install and configure in this exact order.
Each step must complete and verify before the next begins.

```
0. Create LIBRARY_LEDGER.md   Create the file before any installs — see library-ledger.md LL1
1. Python environment         python -m venv .venv && source .venv/bin/activate
2. Requirements structure     mkdir requirements; touch requirements/{base,development,testing,production}.txt
3. Django                     Context7: "django installation" → pip install django
4. Database adapter           psycopg[binary] for Postgres; sqlite built-in for dev
5. Core packages              djangorestframework, django-environ — Context7 each
6. Auth                       SimpleJWT or allauth — Context7; only if required
7. API docs                   drf-spectacular — Context7; install if REST API exists
8. Feature libs               Install only what activated skills require — Context7 each
9. Dev tooling                pytest-django, factory_boy, ruff, mypy, pre-commit — Context7 each
10. Verify                    python manage.py check --deploy (dev settings); migrations apply cleanly
```

Write a ledger entry for every install in steps 3–9.

**Never skip step 10.** A clean management check after scaffolding is the non-negotiable baseline.

---

## IP5. Requirements File Discipline

```
requirements/
├── base.txt          # Runtime production deps — Django, DRF, psycopg, etc.
├── development.txt   # -r base.txt + dev tools: django-debug-toolbar, django-silk
├── testing.txt       # -r base.txt + test tools: pytest-django, factory-boy, coverage
└── production.txt    # -r base.txt + prod-specific: gunicorn, sentry-sdk, whitenoise
```

Rules:
- `base.txt` — pin exact versions for production stability: `Django==6.0.1`
- Development/testing — use compatible ranges: `pytest-django>=4.9,<5`
- Regenerate with `pip-compile` (pip-tools) or `poetry lock` after any change
- Never commit `requirements.txt` flat file — only split files

---

## IP6. Packages Where Setup Changed Significantly Between Major Versions

When working with these, always fetch the migration guide via Context7:

| Package | What changed |
|---------|-------------|
| `celery` v4 → v5 | Task `apply_async` changes, `CELERY_` prefix deprecated, `--pool` default changed |
| `django-allauth` v0.6x → v0.7x | Settings namespace changed, headless API restructured |
| `drf-spectacular` v0.2x → v0.3x | `SCHEMA_PATH_PREFIX` handling, new decorator API |
| `djangorestframework-simplejwt` v4 → v5 | Token rotation settings renamed |
| `django-storages` v1.12 → v1.14 | `STORAGES` dict API replaces individual `DEFAULT_FILE_STORAGE` |
| `psycopg` v2 → v3 | Package renamed `psycopg` (no `[binary]` by default), async support native |
| `django` v4 → v5 | `db_default`, composite PKs, `LoginRequiredMiddleware` |
| `django` v5 → v6 | Native CSP, security middleware changes — check release notes |

For all of these: `Context7 query: "[package] v[installed] migration"` before touching any code.

---

## IP7. When to Re-Run the Pre-Check

Run IP2 again whenever:
- A new skill is activated mid-project
- `requirements/*.txt` is pulled from git with new or updated dependencies
- A peer dependency warning appears during any install
- An `ImportError` or `ModuleNotFoundError` appears
