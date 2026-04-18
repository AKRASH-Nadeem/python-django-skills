---
name: django-core-conventions
description: >
  Apply this skill for any Django-specific implementation: models, managers, querysets,
  views (CBV/FBV), URL routing, templates, forms, signals, admin, middleware, settings,
  migrations, custom commands, and Django ORM patterns. Use when writing or reviewing
  Django model design, view logic, URL patterns, or any core Django feature.
---

# Django Core Conventions Skill

> FIRST: Use Context7 with library ID `/websites/djangoproject_en_6_0` to get
> up-to-date Django 6 documentation before implementing any feature.

---

## 1. Project Layout — Feature-Based Architecture

```
project_root/
├── config/                     # Project configuration (not an app)
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py             # Shared settings
│   │   ├── development.py      # Dev overrides
│   │   ├── testing.py          # Test overrides
│   │   └── production.py       # Production overrides
│   ├── urls.py                 # Root URLconf
│   ├── asgi.py
│   └── wsgi.py
├── apps/                       # All Django apps live here
│   ├── core/                   # Shared utilities, base classes, exceptions
│   │   ├── models.py           # Abstract base models (TimestampedModel, etc.)
│   │   ├── exceptions.py
│   │   ├── middleware.py
│   │   └── utils.py
│   ├── users/                  # User management domain
│   │   ├── models.py
│   │   ├── managers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   ├── serializers.py
│   │   ├── services.py         # Business logic
│   │   ├── selectors.py        # Query logic (read operations)
│   │   ├── signals.py
│   │   ├── tasks.py
│   │   ├── admin.py
│   │   ├── tests/
│   │   │   ├── test_models.py
│   │   │   ├── test_views.py
│   │   │   ├── test_services.py
│   │   │   └── factories.py
│   │   └── migrations/
│   └── orders/                 # Another business domain
│       └── ... (same structure)
├── requirements/
│   ├── base.txt
│   ├── development.txt
│   ├── testing.txt
│   └── production.txt
├── static/
├── media/
├── docs/
├── .env.example
├── manage.py
└── pyproject.toml
```

---

## 2. Settings — Layered, Environment-Driven

```python
# config/settings/base.py
from __future__ import annotations
import environ
from pathlib import Path

env = environ.Env()

BASE_DIR = Path(__file__).resolve().parent.parent.parent

# Read .env file if present (development only)
environ.Env.read_env(BASE_DIR / ".env")

SECRET_KEY = env("DJANGO_SECRET_KEY")
DEBUG = env.bool("DJANGO_DEBUG", default=False)
ALLOWED_HOSTS = env.list("DJANGO_ALLOWED_HOSTS", default=[])

# Application definition — explicit ordering matters
DJANGO_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
]

THIRD_PARTY_APPS = [
    "rest_framework",
    "django_filters",
    "corsheaders",
    "django_celery_beat",
]

LOCAL_APPS = [
    "apps.core",
    "apps.users",
    "apps.orders",
]

INSTALLED_APPS = DJANGO_APPS + THIRD_PARTY_APPS + LOCAL_APPS

# AUTH — always use custom user model
AUTH_USER_MODEL = "users.User"

# Databases — use dj-database-url or django-environ
DATABASES = {
    "default": env.db("DATABASE_URL", default="sqlite:///db.sqlite3")
}
DATABASES["default"]["CONN_MAX_AGE"] = env.int("DB_CONN_MAX_AGE", default=60)
DATABASES["default"]["OPTIONS"] = {"connect_timeout": 10}

# Cache — Redis in production, locmem in dev
# NOTE: REDIS_URL has a localhost default for dev convenience only.
# In production settings, override with no default:
#   CACHES["default"]["LOCATION"] = env("REDIS_URL")  # No default — crash if missing
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": env("REDIS_URL", default="redis://localhost:6379/0"),
    }
}

# Internationalization
LANGUAGE_CODE = "en-us"
TIME_ZONE = "UTC"  # ALWAYS UTC — convert in views/serializers
USE_I18N = True
USE_TZ = True

# Static & Media
STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "media"

# Default primary key
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"

# Logging configuration
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "verbose": {
            "format": "{levelname} {asctime} {module} {process:d} {thread:d} {message}",
            "style": "{",
        },
        "json": {
            "()": "pythonjsonlogger.jsonlogger.JsonFormatter",
            "format": "%(asctime)s %(name)s %(levelname)s %(message)s",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "json",
        },
    },
    "root": {"handlers": ["console"], "level": "INFO"},
    "loggers": {
        "django": {"handlers": ["console"], "level": "INFO", "propagate": False},
        "django.db.backends": {
            "handlers": ["console"],
            "level": "WARNING",  # Set to DEBUG to log all queries
            "propagate": False,
        },
        "apps": {"handlers": ["console"], "level": "DEBUG", "propagate": False},
    },
}
```

---

## 3. Abstract Base Models — Never Repeat Yourself

```python
# apps/core/models.py
from __future__ import annotations
import uuid
from django.db import models


class TimestampedModel(models.Model):
    """Abstract base providing created_at and updated_at timestamps."""
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class UUIDModel(models.Model):
    """Abstract base using UUID as primary key."""
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)

    class Meta:
        abstract = True


class SoftDeleteModel(models.Model):
    """Abstract base for soft deletion support."""
    deleted_at = models.DateTimeField(null=True, blank=True, db_index=True)
    is_deleted = models.BooleanField(default=False, db_index=True)

    class Meta:
        abstract = True

    def soft_delete(self) -> None:
        from django.utils import timezone
        self.deleted_at = timezone.now()
        self.is_deleted = True
        self.save(update_fields=["deleted_at", "is_deleted"])

    def restore(self) -> None:
        self.deleted_at = None
        self.is_deleted = False
        self.save(update_fields=["deleted_at", "is_deleted"])


class BaseModel(UUIDModel, TimestampedModel, SoftDeleteModel):
    """Combine all base behaviors. Use this for most domain models."""
    class Meta:
        abstract = True
        ordering = ["-created_at"]
```

---

## 4. Custom User Model — Always Define Before First Migration

```python
# apps/users/models.py
from __future__ import annotations
from django.contrib.auth.models import AbstractBaseUser, PermissionsMixin
from django.db import models
from apps.core.models import TimestampedModel
from apps.users.managers import UserManager


class User(AbstractBaseUser, PermissionsMixin, TimestampedModel):
    """
    Custom user model. MUST be defined before any migrations are created.
    Set AUTH_USER_MODEL = 'users.User' in settings.
    """
    email = models.EmailField(unique=True, db_index=True)
    first_name = models.CharField(max_length=150, blank=True)
    last_name = models.CharField(max_length=150, blank=True)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)

    USERNAME_FIELD = "email"
    REQUIRED_FIELDS = ["first_name", "last_name"]

    objects: UserManager = UserManager()

    class Meta:
        verbose_name = "user"
        verbose_name_plural = "users"
        ordering = ["-date_joined"]

    def get_full_name(self) -> str:
        return f"{self.first_name} {self.last_name}".strip()

    def __str__(self) -> str:
        return self.email
```

---

## 5. Managers & QuerySets — Encapsulate Query Logic

```python
# apps/users/managers.py
from __future__ import annotations
from django.contrib.auth.models import BaseUserManager
from django.db import models
from django.utils import timezone


class ActiveUserQuerySet(models.QuerySet["User"]):
    def active(self) -> "ActiveUserQuerySet":
        return self.filter(is_active=True, is_deleted=False)

    def staff(self) -> "ActiveUserQuerySet":
        return self.filter(is_staff=True)

    def with_orders(self) -> "ActiveUserQuerySet":
        return self.prefetch_related("orders")


class UserManager(BaseUserManager):
    def get_queryset(self) -> ActiveUserQuerySet:
        return ActiveUserQuerySet(self.model, using=self._db)

    def active(self) -> ActiveUserQuerySet:
        return self.get_queryset().active()

    def create_user(
        self, email: str, password: str | None = None, **extra_fields: object
    ) -> "User":
        if not email:
            raise ValueError("Email is required")
        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_superuser(self, email: str, password: str, **extra_fields: object) -> "User":
        extra_fields.setdefault("is_staff", True)
        extra_fields.setdefault("is_superuser", True)
        return self.create_user(email, password, **extra_fields)
```

---

## 6. Service Layer — Business Logic Belongs Here

```python
# apps/users/services.py
"""
Services contain all business logic.
Rules:
- Functions, not classes (unless stateful service is needed)
- Take primitives or validated data as input
- Return domain objects or raise domain exceptions
- Wrap multi-step writes in transaction.atomic()
- Trigger async side effects via transaction.on_commit()
"""
from __future__ import annotations
import logging
from django.contrib.auth import get_user_model
from django.db import transaction
from apps.core.exceptions import ValidationError, NotFoundError

logger = logging.getLogger(__name__)
User = get_user_model()


def create_user(
    *,
    email: str,
    password: str,
    first_name: str,
    last_name: str,
) -> "User":
    """Create a new user account. Raises ValidationError on duplicate email."""
    if User.objects.filter(email=email).exists():
        raise ValidationError(f"User with email '{email}' already exists.")

    with transaction.atomic():
        user = User.objects.create_user(
            email=email,
            password=password,
            first_name=first_name,
            last_name=last_name,
        )
        logger.info("User created", extra={"user_id": str(user.pk)})
        # Trigger async side effects AFTER the transaction commits
        transaction.on_commit(lambda: _send_welcome_email.delay(str(user.pk)))

    return user


def _send_welcome_email(user_id: str) -> None:
    """Import lazily to avoid circular imports."""
    from apps.users.tasks import send_welcome_email
    send_welcome_email.delay(user_id)
```

---

## 7. Selectors — Encapsulate Read Queries

```python
# apps/users/selectors.py
"""
Selectors contain all read/query logic.
Rules:
- Never perform writes
- Always return QuerySets or model instances
- Apply all optimization (select_related, prefetch_related) here
- Never duplicate query logic in views
"""
from __future__ import annotations
from django.contrib.auth import get_user_model
from django.db.models import QuerySet

User = get_user_model()


def get_active_users() -> QuerySet:
    return User.objects.active().order_by("-created_at")


def get_user_by_email(email: str) -> "User | None":
    return User.objects.filter(email=email, is_active=True).first()


def get_user_with_profile(user_id: str) -> "User | None":
    return (
        User.objects
        .select_related("profile")
        .prefetch_related("orders", "orders__items")
        .filter(pk=user_id, is_active=True)
        .first()
    )
```

---

## 8. Views — Thin, Orchestrating Layer

```python
# apps/users/views.py
"""
Views ONLY:
1. Parse the request
2. Call the service/selector
3. Serialize and return the response
They do NOT contain business logic.
"""
from __future__ import annotations
from rest_framework import status
from rest_framework.request import Request
from rest_framework.response import Response
from rest_framework.views import APIView
from apps.core.exceptions import ValidationError, NotFoundError
from apps.users import selectors, services
from apps.users.serializers import UserCreateSerializer, UserDetailSerializer


class UserCreateView(APIView):
    def post(self, request: Request) -> Response:
        serializer = UserCreateSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        try:
            user = services.create_user(**serializer.validated_data)
        except ValidationError as exc:
            return Response(
                {"error": exc.message, "code": exc.code},
                status=status.HTTP_400_BAD_REQUEST,
            )

        return Response(
            UserDetailSerializer(user).data,
            status=status.HTTP_201_CREATED,
        )
```

---

## 9. URL Configuration

```python
# config/urls.py
from django.conf import settings
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/v1/", include("config.api_router")),
]

if settings.DEBUG:
    from django.conf.urls.static import static
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)

# config/api_router.py
from django.urls import path, include

urlpatterns = [
    path("users/", include("apps.users.urls", namespace="users")),
    path("orders/", include("apps.orders.urls", namespace="orders")),
]

# apps/users/urls.py
from django.urls import path
from apps.users.views import UserCreateView, UserDetailView

app_name = "users"  # namespace

urlpatterns = [
    path("", UserCreateView.as_view(), name="user-list"),
    path("<uuid:user_id>/", UserDetailView.as_view(), name="user-detail"),
]
```

---

## 10. Signals — Use Sparingly, Document Always

```python
# apps/users/signals.py
"""
Signal rules:
- Use signals for cross-app decoupled notifications (e.g., user created → audit log)
- Do NOT use signals for business logic within the same app
- ALWAYS use post_save with created=True check, not pre_save for creation side effects
- NEVER do heavy work in signals — dispatch to Celery
- ALWAYS connect in AppConfig.ready()
"""
from __future__ import annotations
import logging
from django.db.models.signals import post_save
from django.dispatch import receiver, Signal
from django.contrib.auth import get_user_model

logger = logging.getLogger(__name__)

# Custom domain signals
user_registered = Signal()  # Provides: user, request
order_placed = Signal()  # Provides: order, user


@receiver(post_save, sender="users.User")
def on_user_created(sender: type, instance: object, created: bool, **kwargs: object) -> None:
    if not created:
        return
    from apps.audit.tasks import log_user_creation
    from django.db import transaction
    transaction.on_commit(lambda: log_user_creation.delay(str(instance.pk)))


# apps/users/apps.py
from django.apps import AppConfig

class UsersConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "apps.users"

    def ready(self) -> None:
        import apps.users.signals  # noqa: F401 — registers signal handlers
```

---

## 11. Migrations — Best Practices

```python
# Rules:
# 1. NEVER edit a migration that has been applied to any shared environment
# 2. ALWAYS review auto-generated migrations before committing
# 3. Rename auto-generated migration files to describe the change
# 4. Data migrations must use RunPython with named functions (not lambdas)
# 5. Large table data migrations must be run in batches with atomic=False

# Example data migration:
from django.db import migrations


def populate_email_hash(apps: object, schema_editor: object) -> None:
    """Backfill email_hash field from existing email values."""
    User = apps.get_model("users", "User")
    import hashlib
    for user in User.objects.only("id", "email").iterator(chunk_size=500):
        user.email_hash = hashlib.sha256(user.email.encode()).hexdigest()
        user.save(update_fields=["email_hash"])


def reverse_populate_email_hash(apps: object, schema_editor: object) -> None:
    User = apps.get_model("users", "User")
    User.objects.update(email_hash="")


class Migration(migrations.Migration):
    dependencies = [("users", "0002_user_email_hash")]

    operations = [
        migrations.RunPython(populate_email_hash, reverse_populate_email_hash),
    ]
```

---

## 12. Admin — Secure & Useful

```python
# apps/users/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin as BaseUserAdmin
from django.contrib.auth import get_user_model

User = get_user_model()


@admin.register(User)
class UserAdmin(BaseUserAdmin):
    list_display = ("email", "first_name", "last_name", "is_active", "created_at")
    list_filter = ("is_active", "is_staff", "created_at")
    search_fields = ("email", "first_name", "last_name")
    ordering = ("-created_at",)
    readonly_fields = ("created_at", "updated_at", "last_login")

    fieldsets = (
        (None, {"fields": ("email", "password")}),
        ("Personal Info", {"fields": ("first_name", "last_name")}),
        ("Permissions", {"fields": ("is_active", "is_staff", "is_superuser", "groups")}),
        ("Dates", {"fields": ("last_login", "created_at", "updated_at")}),
    )
    add_fieldsets = (
        (None, {
            "classes": ("wide",),
            "fields": ("email", "password1", "password2", "first_name", "last_name"),
        }),
    )

    # SECURITY: Limit admin to non-deleted users only
    def get_queryset(self, request: object) -> object:
        return super().get_queryset(request).filter(is_deleted=False)
```

---

## 13. ORM Anti-Patterns to Avoid

```python
# ❌ WRONG — N+1 query
for order in Order.objects.all():
    print(order.user.email)  # hits DB for every order

# ✅ CORRECT
for order in Order.objects.select_related("user").all():
    print(order.user.email)  # single JOIN query

# ❌ WRONG — loading entire queryset into memory
users = list(User.objects.all())  # could be millions

# ✅ CORRECT — use iterator() for large sets
for user in User.objects.iterator(chunk_size=500):
    process(user)

# ❌ WRONG — save() on full model when only updating one field
user.last_login = timezone.now()
user.save()  # UPDATE ALL COLUMNS

# ✅ CORRECT
user.last_login = timezone.now()
user.save(update_fields=["last_login"])  # UPDATE only last_login

# ❌ WRONG — multiple round trips
count = User.objects.count()
if count > 0:
    users = User.objects.all()

# ✅ CORRECT
users = User.objects.all()
if users.exists():  # single efficient EXISTS query
    ...

# ❌ WRONG — check in Python
emails = [u.email for u in User.objects.all()]
if target_email in emails:
    ...

# ✅ CORRECT — check in DB
if User.objects.filter(email=target_email).exists():
    ...
```

---

## 14. Model Meta — Always Complete

```python
class Order(BaseModel):
    class Meta:
        verbose_name = "order"
        verbose_name_plural = "orders"
        ordering = ["-created_at"]
        indexes = [
            models.Index(fields=["user", "status"]),
            models.Index(fields=["created_at"]),
        ]
        constraints = [
            models.CheckConstraint(
                check=models.Q(total__gte=0),
                name="order_total_non_negative",
            ),
            models.UniqueConstraint(
                fields=["user", "reference"],
                name="unique_user_order_reference",
            ),
        ]
```
