---
name: django-async-tasks
description: >
  Apply this skill for all background task, job queue, and scheduled work:
  Celery setup with Django, task design patterns, retry strategies, periodic tasks
  with Celery Beat, task monitoring, error handling, chaining/workflows,
  and transaction-safe task dispatching. Use when work should happen in the background,
  on a schedule, or in response to events.
---

# Django Async Tasks (Celery) Skill

> FIRST: Use Context7 with library ID `/celery/celery` for the latest Celery docs.
> Also query `/celery/django-celery-beat` for periodic task scheduler docs.

---

## 1. When to Use Celery

```
Use Celery for:
✅ Sending emails, SMS, push notifications
✅ Processing file uploads (resize, transcode, parse)
✅ External API calls (payment processing, webhooks delivery)
✅ Report generation
✅ Scheduled jobs (nightly cleanup, daily summaries)
✅ Any operation taking > 200ms
✅ Work that should survive request failures
✅ Fan-out operations (notify 1000 users)

Do NOT use Celery for:
❌ Reading a DB record and returning it (use the ORM directly)
❌ Critical operations that must complete before response
❌ Work that takes < 50ms and doesn't benefit from retry
```

---

## 2. Celery Setup

```python
# config/celery.py
from __future__ import annotations
import os
from celery import Celery
from django.conf import settings

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.production")

app = Celery("project_name")

# Load config from Django settings, namespace 'CELERY_'
app.config_from_object("django.conf:settings", namespace="CELERY")

# Auto-discover tasks in all installed apps
app.autodiscover_tasks(lambda: settings.INSTALLED_APPS)


@app.task(bind=True)
def debug_task(self) -> None:
    """Health check task."""
    print(f"Request: {self.request!r}")


# config/__init__.py
from config.celery import app as celery_app
__all__ = ("celery_app",)
```

### Celery Settings
```python
# config/settings/base.py
CELERY_BROKER_URL = env("CELERY_BROKER_URL", default="redis://localhost:6379/2")
CELERY_RESULT_BACKEND = env("CELERY_RESULT_BACKEND", default="redis://localhost:6379/3")

# Serialization
CELERY_TASK_SERIALIZER = "json"
CELERY_RESULT_SERIALIZER = "json"
CELERY_ACCEPT_CONTENT = ["json"]

# Reliability
CELERY_TASK_ACKS_LATE = True           # Ack AFTER task completes (safer)
CELERY_TASK_REJECT_ON_WORKER_LOST = True
CELERY_WORKER_PREFETCH_MULTIPLIER = 1  # One task at a time per worker (for long tasks)

# Timeouts
CELERY_TASK_SOFT_TIME_LIMIT = 300      # 5 min: raises SoftTimeLimitExceeded → cleanup
CELERY_TASK_TIME_LIMIT = 360           # 6 min: hard kill

# Result expiry
CELERY_RESULT_EXPIRES = 60 * 60 * 24  # 24 hours

# Routing — define queues per task type
CELERY_TASK_ROUTES = {
    "apps.notifications.tasks.*": {"queue": "notifications"},
    "apps.reports.tasks.*": {"queue": "reports"},
    "apps.emails.tasks.*": {"queue": "emails"},
    # Default goes to "celery" queue
}

CELERY_TASK_DEFAULT_QUEUE = "default"

# Beat scheduler (django-celery-beat)
CELERY_BEAT_SCHEDULER = "django_celery_beat.schedulers:DatabaseScheduler"
```

---

## 3. Task Design Patterns

### Standard Task Pattern
```python
# apps/emails/tasks.py
from __future__ import annotations
import logging
from typing import Any
from celery import shared_task
from celery.exceptions import SoftTimeLimitExceeded

logger = logging.getLogger(__name__)


@shared_task(
    bind=True,
    name="emails.send_welcome_email",        # Explicit name — never auto-generated
    max_retries=3,
    default_retry_delay=60,                  # 60 second base delay
    autoretry_for=(ConnectionError, TimeoutError),
    retry_backoff=True,                      # Exponential backoff
    retry_backoff_max=600,                   # Max 10 min between retries
    retry_jitter=True,                       # Add randomness to prevent thundering herd
    soft_time_limit=120,                     # Task-specific timeout
    time_limit=150,
    queue="emails",
    acks_late=True,
)
def send_welcome_email(self, user_id: str) -> dict[str, Any]:
    """
    Send welcome email to new user.

    Args:
        user_id: UUID string of the user.

    Returns:
        dict with status and message_id.
    """
    from django.contrib.auth import get_user_model
    User = get_user_model()

    logger.info("Sending welcome email", extra={"user_id": user_id, "task_id": self.request.id})

    try:
        user = User.objects.get(pk=user_id, is_active=True)
    except User.DoesNotExist:
        logger.warning("User not found for welcome email", extra={"user_id": user_id})
        return {"status": "skipped", "reason": "user_not_found"}

    try:
        from apps.emails.services import EmailService
        result = EmailService.send_welcome(user)
        logger.info("Welcome email sent", extra={"user_id": user_id, "message_id": result["id"]})
        return {"status": "sent", "message_id": result["id"]}

    except SoftTimeLimitExceeded:
        logger.error("Welcome email task timed out", extra={"user_id": user_id})
        raise  # Let Celery handle

    except Exception as exc:
        logger.error(
            "Welcome email failed",
            extra={"user_id": user_id, "error": str(exc)},
            exc_info=True,
        )
        raise self.retry(exc=exc)
```

### Transaction-Safe Dispatching
```python
# apps/users/services.py
from django.db import transaction


def create_user(*, email: str, password: str, **extra: object) -> Any:
    from django.contrib.auth import get_user_model
    User = get_user_model()

    with transaction.atomic():
        user = User.objects.create_user(email=email, password=password, **extra)

        # CRITICAL: Use on_commit to ensure task fires ONLY after DB commits.
        # Without this, the task may run before the user row exists.
        transaction.on_commit(
            lambda: send_welcome_email.delay(str(user.pk))
        )

    return user
```

### Idempotent Tasks — Critical Pattern
```python
@shared_task(bind=True, name="payments.process_payment", acks_late=True)
def process_payment(self, payment_id: str) -> dict[str, Any]:
    """
    Idempotency rule: this task MUST be safe to run multiple times.
    Check idempotency key before processing.
    """
    from apps.payments.models import Payment
    from django.core.cache import cache

    idempotency_key = f"payment_processing:{payment_id}"
    # Use cache lock to prevent double-processing
    if not cache.add(idempotency_key, "1", timeout=3600):
        logger.info("Payment already being processed", extra={"payment_id": payment_id})
        return {"status": "already_processing"}

    try:
        payment = Payment.objects.get(pk=payment_id)
        if payment.status in ("completed", "refunded"):
            return {"status": "already_processed"}

        result = _charge_card(payment)
        payment.mark_completed(transaction_id=result["id"])
        return {"status": "completed", "transaction_id": result["id"]}
    finally:
        cache.delete(idempotency_key)
```

---

## 4. Celery Workflows (Chains, Groups, Chords)

```python
from celery import chain, group, chord, signature


# Chain: sequential tasks (output of one → input of next)
def process_order_workflow(order_id: str) -> None:
    workflow = chain(
        validate_order.s(order_id),
        charge_payment.s(),          # Receives output of validate_order
        update_inventory.s(),
        send_confirmation_email.s(),
    )
    workflow.apply_async()


# Group: parallel tasks (no dependency)
def notify_all_users(user_ids: list[str], message: str) -> None:
    job = group(
        send_notification.s(user_id, message)
        for user_id in user_ids
    )
    job.apply_async()


# Chord: parallel then callback (fan-out → fan-in)
def generate_report(report_id: str, section_ids: list[str]) -> None:
    callback = finalize_report.s(report_id)
    header = group(
        generate_section.s(report_id, section_id)
        for section_id in section_ids
    )
    chord(header)(callback)


# Signature with immutable args (| operator)
from celery import signature
sig = send_email.si(user_id="123")  # .si() = immutable signature (ignores parent result)
```

---

## 5. Periodic Tasks (Celery Beat)

```python
# config/settings/base.py
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    # Run every day at midnight UTC
    "daily-cleanup": {
        "task": "apps.core.tasks.cleanup_expired_tokens",
        "schedule": crontab(hour=0, minute=0),
        "options": {"queue": "maintenance"},
    },
    # Every hour
    "hourly-sync": {
        "task": "apps.integrations.tasks.sync_external_catalog",
        "schedule": crontab(minute=0),
        "options": {"queue": "integrations", "expires": 3500},  # Don't run stale tasks
    },
    # Every 5 minutes
    "process-pending-webhooks": {
        "task": "apps.webhooks.tasks.retry_failed_deliveries",
        "schedule": 300,  # seconds
    },
}

# For dynamic schedules (configurable via admin), use django-celery-beat:
# from django_celery_beat.models import PeriodicTask, IntervalSchedule
```

---

## 6. Task Monitoring

```python
# Flower for monitoring: pip install flower
# Run: celery -A config.celery flower --port=5555

# Custom task event hooks for alerting
from celery.signals import task_failure, task_retry, task_success

@task_failure.connect
def on_task_failure(sender, task_id, exception, args, kwargs, traceback, einfo, **kw):
    logger.error(
        "Task %s failed",
        sender.name,
        extra={"task_id": task_id, "exception": str(exception)},
        exc_info=einfo,
    )
    # Send to Sentry
    import sentry_sdk
    sentry_sdk.capture_exception(exception)

@task_retry.connect
def on_task_retry(sender, request, reason, einfo, **kw):
    logger.warning(
        "Task %s retrying (attempt %d/%d)",
        sender.name,
        request.retries + 1,
        sender.max_retries or "∞",
        extra={"task_id": request.id, "reason": str(reason)},
    )
```

---

## 7. Dead Letter Queue (DLQ) Pattern

```python
# apps/core/tasks.py
@shared_task(bind=True, name="core.dead_letter_requeue", queue="dlq")
def requeue_dead_letter(self, original_task_name: str, args: list, kwargs: dict) -> None:
    """
    Re-enqueue failed tasks from the DLQ after manual review.
    All tasks that exhaust retries should be routed to DLQ for inspection.
    """
    from celery import current_app
    current_app.send_task(original_task_name, args=args, kwargs=kwargs)
    logger.info("Re-queued dead letter task: %s", original_task_name)


# Configure in settings:
# CELERY_TASK_ROUTES also handles DLQ routing via task_failure signal
```

---

## 8. Task Best Practices Summary

```
Task Design:
✅ Pass IDs, not objects (objects can't serialize reliably)
✅ Make tasks idempotent (safe to run multiple times)
✅ Explicit task names (never rely on auto-generation)
✅ Always use transaction.on_commit() when dispatching from a view/service
✅ Short, single-purpose tasks (compose with chains/groups for complexity)
✅ Never access request.session or request.user in tasks
✅ Never rely on shared mutable state between tasks

Reliability:
✅ acks_late=True on all tasks (ack after completion, not receipt)
✅ retry_on appropriate exceptions (network, temporary failures)
✅ Retry with exponential backoff + jitter
✅ Soft time limits for cleanup, hard limits for kill
✅ Log task start, success, and failure with context
✅ Route to named queues (notifications, emails, reports)
✅ Separate workers per queue type for isolation

Scheduling:
✅ expires option on periodic tasks (prevent stale task pile-up)
✅ django-celery-beat for dynamic schedule management
✅ All scheduled tasks are idempotent
✅ Periodic tasks use UTC schedule times
```
