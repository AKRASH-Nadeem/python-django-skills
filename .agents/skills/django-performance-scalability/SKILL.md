---
name: django-performance-scalability
description: >
  Apply this skill for all performance, caching, and scalability work: Redis caching
  strategies, queryset optimization, database indexing, connection pooling, query
  analysis, bulk operations, pagination of large datasets, horizontal scaling patterns,
  and CDN integration. Use when queries are slow, pages load too slowly, or the
  system needs to handle more load.
---

# Django Performance & Scalability Skill

> FIRST: Use Context7 `/websites/djangoproject_en_6_0` for caching and ORM docs.
> Profile before optimizing. Never guess — measure.

---

## 1. The Performance Hierarchy

```
(Highest ROI → Lowest ROI)

1. Database query optimization (N+1 elimination, indexing)
2. Caching (Redis — service layer, not view layer)
3. Async processing (offload work to Celery)
4. Connection pooling (pgBouncer)
5. CDN (static/media files)
6. DB read replicas
7. Horizontal scaling (more workers)
8. Code-level micro-optimizations (last resort)

Rule: Always profile first. Use django-silk or django-debug-toolbar
to identify bottlenecks before touching code.
```

---

## 2. Database Query Optimization

### The Query Optimization Checklist
```python
# For EVERY queryset returning related objects, ask:
# 1. Which FK/O2O relations do I traverse? → select_related()
# 2. Which M2M/reverse FK relations do I traverse? → prefetch_related()
# 3. Do I need all fields? → .only() or .values()
# 4. Do I need the result as objects or dicts? → .values() for aggregation/API
# 5. Do I evaluate this queryset more than once? → cache the result
# 6. Is this queryset in a loop? → STOP — move the query outside the loop

# Pattern: Selector functions own all optimization
# apps/orders/selectors.py
from django.db.models import Prefetch, QuerySet
from apps.orders.models import Order, OrderItem
from django.contrib.auth import get_user_model

User = get_user_model()


def get_orders_for_dashboard(user_id: str) -> QuerySet:
    """Fully optimized queryset for order dashboard."""
    return (
        Order.objects
        .filter(user_id=user_id, is_deleted=False)
        .select_related("user", "shipping_address")
        .prefetch_related(
            Prefetch(
                "items",
                queryset=OrderItem.objects.select_related("product").only(
                    "id", "quantity", "unit_price", "product__name", "product__sku"
                ),
            )
        )
        .only(
            "id", "reference", "status", "total", "created_at",
            "user__email", "user__first_name",
            "shipping_address__city", "shipping_address__country",
        )
        .order_by("-created_at")
    )
```

### Bulk Operations — Never Loop Saves
```python
# ❌ WRONG — 1000 DB round trips
for product_id in product_ids:
    OrderItem.objects.create(order=order, product_id=product_id, quantity=1)

# ✅ CORRECT — 1 DB round trip
OrderItem.objects.bulk_create([
    OrderItem(order=order, product_id=pid, quantity=1)
    for pid in product_ids
], batch_size=500)  # batch_size prevents memory explosion

# ✅ CORRECT — Bulk update
Order.objects.filter(status="pending").update(
    status="processing",
    updated_at=timezone.now(),
)

# ✅ CORRECT — Bulk update with different values per row
from django.db.models import Case, When, Value, CharField
Order.objects.filter(id__in=order_ids).update(
    status=Case(
        *[When(id=oid, then=Value(s)) for oid, s in id_status_map.items()],
        output_field=CharField(),
    )
)
```

### Annotations — Compute in DB, Not Python
```python
from django.db.models import Count, Sum, Avg, F, Q, ExpressionWrapper, DecimalField
from django.db.models.functions import TruncDate, Coalesce

# ✅ Aggregate in DB
stats = Order.objects.filter(user=user).aggregate(
    total_orders=Count("id"),
    total_spent=Coalesce(Sum("total"), 0),
    avg_order_value=Avg("total"),
)

# ✅ Annotate counts for list display (no N+1)
users = User.objects.annotate(
    order_count=Count("orders", filter=Q(orders__is_deleted=False)),
    total_spent=Coalesce(Sum("orders__total"), 0),
)

# ✅ Computed fields
items = OrderItem.objects.annotate(
    subtotal=ExpressionWrapper(
        F("quantity") * F("unit_price"),
        output_field=DecimalField(max_digits=10, decimal_places=2),
    )
)
```

---

## 3. Database Indexing Strategy

```python
class Order(BaseModel):
    user = models.ForeignKey(User, on_delete=models.PROTECT, related_name="orders")
    status = models.CharField(max_length=20, choices=Status.choices, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True)
    total = models.DecimalField(max_digits=12, decimal_places=2)

    class Meta:
        indexes = [
            # Composite: most common filter pattern
            models.Index(fields=["user", "status"], name="order_user_status_idx"),
            # Partial index: only active orders (PostgreSQL)
            models.Index(
                fields=["status", "created_at"],
                name="order_active_created_idx",
                condition=models.Q(is_deleted=False),
            ),
            # For date-range queries
            models.Index(fields=["created_at"], name="order_created_idx"),
        ]

# Index rules:
# ✅ Index FK fields (Django does this for you, but explicit is better)
# ✅ Index fields used in WHERE clauses
# ✅ Index fields used in ORDER BY
# ✅ Composite index for multi-field filter patterns
# ✅ Partial indexes for status fields with common filters
# ❌ Don't index every field — writes become slower
# ❌ Don't index low-cardinality fields (boolean) alone
```

---

## 4. Caching — Redis Strategies

```python
# apps/core/cache.py
"""
Cache strategy rules:
- Cache at SERVICE layer, not view layer
- Use structured cache keys: {app}:{model}:{id}:{version}
- Always set TTL — never cache indefinitely
- Cache invalidation: delete on write, not on read
- Use cache versioning for bulk invalidation
"""
from __future__ import annotations
from functools import wraps
from typing import Any, Callable, TypeVar
from django.core.cache import cache
import logging

logger = logging.getLogger(__name__)

F = TypeVar("F", bound=Callable[..., Any])

CACHE_VERSION = 1  # Bump to invalidate all cached data


def cache_key(*parts: Any) -> str:
    """Build a structured, safe cache key."""
    return ":".join(str(p) for p in ["v" + str(CACHE_VERSION)] + list(parts))


def cached(key_fn: Callable[..., str], ttl: int = 300) -> Callable[[F], F]:
    """
    Decorator for caching function results.
    key_fn receives the same args as the decorated function.
    """
    def decorator(func: F) -> F:
        @wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            key = key_fn(*args, **kwargs)
            result = cache.get(key)
            if result is not None:
                return result
            result = func(*args, **kwargs)
            if result is not None:
                cache.set(key, result, timeout=ttl)
            return result
        return wrapper  # type: ignore[return-value]
    return decorator


# apps/products/selectors.py
@cached(
    key_fn=lambda product_id: cache_key("products", "detail", product_id),
    ttl=3600,  # 1 hour
)
def get_product_detail(product_id: str) -> dict[str, Any] | None:
    from apps.products.models import Product
    product = (
        Product.objects
        .select_related("category", "brand")
        .prefetch_related("images", "variants")
        .filter(pk=product_id, is_active=True)
        .first()
    )
    if product is None:
        return None
    from apps.products.serializers import ProductDetailSerializer
    return ProductDetailSerializer(product).data


# apps/products/services.py
def update_product(product_id: str, **data: Any) -> Any:
    """Update product and invalidate its cache."""
    from apps.products.models import Product
    product = Product.objects.get(pk=product_id)
    for field, value in data.items():
        setattr(product, field, value)
    product.save(update_fields=list(data.keys()) + ["updated_at"])
    # Invalidate
    cache.delete(cache_key("products", "detail", product_id))
    return product
```

### Cache Many-at-Once (Cache Stampede Prevention)
```python
def get_product_details_bulk(product_ids: list[str]) -> dict[str, Any]:
    """Fetch many products, hitting cache first."""
    keys = {pid: cache_key("products", "detail", pid) for pid in product_ids}
    cached_results = cache.get_many(list(keys.values()))

    missing_ids = [
        pid for pid, key in keys.items()
        if key not in cached_results
    ]

    if missing_ids:
        from apps.products.models import Product
        from apps.products.serializers import ProductDetailSerializer
        db_results = Product.objects.filter(pk__in=missing_ids)
        to_cache = {}
        for product in db_results:
            data = ProductDetailSerializer(product).data
            to_cache[keys[str(product.pk)]] = data
        cache.set_many(to_cache, timeout=3600)
        cached_results.update(to_cache)

    return {pid: cached_results.get(keys[pid]) for pid in product_ids}
```

---

## 5. Database Connection Pooling

```python
# config/settings/production.py
# Use django-db-geventpool or pgBouncer + CONN_MAX_AGE
DATABASES = {
    "default": {
        **env.db("DATABASE_URL"),
        "CONN_MAX_AGE": 60,  # Reuse connections for 60 seconds
        "OPTIONS": {
            "connect_timeout": 10,
            "options": "-c default_transaction_isolation=read committed",
        },
        "TEST": {
            "NAME": "test_db",
        },
    }
}

# For async (Channels/async views) use a separate async DB backend:
# django-db-geventpool or use sync_to_async with thread_sensitive=True
```

---

## 6. Queryset Optimization for Large Datasets

```python
# apps/core/utils.py
from django.db.models import QuerySet
from typing import Generator, TypeVar

M = TypeVar("M")


def queryset_iterator(
    queryset: QuerySet,
    chunk_size: int = 1000,
) -> Generator[Any, None, None]:
    """
    Memory-efficient iteration over large querysets.
    Uses .iterator() to avoid loading all records into memory.
    Avoids issues with queryset caching on large datasets.
    """
    for obj in queryset.iterator(chunk_size=chunk_size):
        yield obj


def paginated_queryset(
    queryset: QuerySet,
    page: int,
    page_size: int = 20,
    max_page_size: int = 100,
) -> tuple[QuerySet, int]:
    """Return paginated slice and total count."""
    page_size = min(page_size, max_page_size)
    offset = (page - 1) * page_size
    # Use separate count() query — faster than len()
    total = queryset.count()
    return queryset[offset:offset + page_size], total


# Efficient count for large tables (skip exact count)
def estimated_count(model_class: type) -> int:
    """PostgreSQL-specific fast estimated count for huge tables."""
    from django.db import connection
    with connection.cursor() as cursor:
        cursor.execute(
            "SELECT reltuples::bigint FROM pg_class WHERE relname = %s",
            [model_class._meta.db_table],
        )
        result = cursor.fetchone()
        return result[0] if result else 0
```

---

## 7. Deferred & Lazy Loading

```python
# Use .only() to load only needed fields
users = User.objects.only("id", "email", "first_name")

# Use .defer() to exclude large fields
posts = Post.objects.defer("body", "raw_html")  # Load all except body/raw_html

# Use .values() for read-only data (returns dicts, not model instances — faster)
emails = User.objects.filter(is_active=True).values_list("email", flat=True)

# Use .values() for aggregation exports
order_data = Order.objects.filter(
    created_at__gte=start
).values("status").annotate(count=Count("id"), total=Sum("total"))
```

---

## 8. Database Read Replicas

```python
# config/settings/production.py
DATABASES = {
    "default": env.db("DATABASE_PRIMARY_URL"),  # Read-write
    "replica": env.db("DATABASE_REPLICA_URL"),   # Read-only
}

# Database router
# apps/core/db_router.py
class PrimaryReplicaRouter:
    """Route reads to replica, writes to primary."""

    READ_DB = "replica"
    WRITE_DB = "default"
    READ_APPS = {"products", "catalog", "reports"}  # Apps safe to read from replica

    def db_for_read(self, model, **hints):
        if model._meta.app_label in self.READ_APPS:
            return self.READ_DB
        return self.WRITE_DB

    def db_for_write(self, model, **hints):
        return self.WRITE_DB

    def allow_relation(self, obj1, obj2, **hints):
        db_set = {self.READ_DB, self.WRITE_DB}
        return obj1._state.db in db_set and obj2._state.db in db_set

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == self.WRITE_DB

# config/settings/production.py
DATABASE_ROUTERS = ["apps.core.db_router.PrimaryReplicaRouter"]
```

---

## 9. Caching Patterns Summary

```
┌──────────────────┬────────────────────────────────────────────────┐
│ Pattern          │ When to Use                                    │
├──────────────────┼────────────────────────────────────────────────┤
│ Cache-aside      │ Default. Read from cache; miss → DB → cache    │
│                  │ Best for: product details, user profiles        │
├──────────────────┼────────────────────────────────────────────────┤
│ Write-through    │ Write to DB AND cache simultaneously           │
│                  │ Best for: high-read, infrequent-write data     │
├──────────────────┼────────────────────────────────────────────────┤
│ Write-behind     │ Write to cache; async flush to DB              │
│                  │ Best for: counters, view counts, likes          │
├──────────────────┼────────────────────────────────────────────────┤
│ Invalidate on    │ Delete cache key on any related write           │
│ write            │ Best for: data correctness over performance    │
├──────────────────┼────────────────────────────────────────────────┤
│ Time-based TTL   │ Accept slightly stale data; expire after N sec │
│                  │ Best for: public catalogs, config data          │
└──────────────────┴────────────────────────────────────────────────┘
```

---

## 10. Performance Monitoring Setup

```python
# config/settings/development.py
INSTALLED_APPS += ["silk"]
MIDDLEWARE += ["silk.middleware.SilkyMiddleware"]
SILKY_PYTHON_PROFILER = True
SILKY_MAX_RECORDED_REQUESTS = 1000
SILKY_MAX_REQUEST_BODY_SIZE = 1024  # bytes

# Production: Use Sentry Performance
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration
from sentry_sdk.integrations.celery import CeleryIntegration
from sentry_sdk.integrations.redis import RedisIntegration

sentry_sdk.init(
    dsn=env("SENTRY_DSN"),
    integrations=[
        DjangoIntegration(transaction_style="url"),
        CeleryIntegration(),
        RedisIntegration(),
    ],
    traces_sample_rate=0.1,  # 10% of requests
    profiles_sample_rate=0.01,
    environment=env("ENVIRONMENT", default="production"),
    send_default_pii=False,
)
```

---

## 11. Scalability Checklist

```
Database:
✅ All querysets use select_related/prefetch_related
✅ No N+1 queries (verified with django-silk in dev)
✅ Bulk create/update instead of loops
✅ Partial indexes for common filter patterns
✅ CONN_MAX_AGE set for connection reuse
✅ Read replica for heavy read workloads

Caching:
✅ Redis cache configured
✅ Service layer caches expensive queries
✅ Cache invalidated on write (not time-only)
✅ Cache stampede prevention for concurrent requests

Application:
✅ Heavy work offloaded to Celery
✅ File serving via CDN (S3 + CloudFront)
✅ Static files via whitenoise + CDN
✅ Async views for I/O-bound operations (ASGI server)
✅ Paginate ALL list endpoints

Infrastructure:
✅ Gunicorn/Uvicorn with multiple workers
✅ pgBouncer for PostgreSQL connection pooling
✅ Redis sentinel/cluster for high availability
✅ Horizontal scaling via stateless workers (no local state)
```
