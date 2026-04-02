---
name: django-architecture
description: >
  Apply this skill for architectural decisions: feature-based app structure, service/
  selector pattern, domain-driven design in Django, cross-app communication, event-driven
  patterns, repository pattern, dependency injection, SOLID principles in Django context,
  and when to split/merge apps. Use when designing new features, refactoring existing
  code structure, or making decisions about how to organize Django code.
---

# Django Architecture Skill

> Architecture decisions are expensive to reverse. Think before you code.
> FIRST: Use Context7 to check Django 6 app discovery and module patterns.

---

## 1. The Architecture Layers

```
Request → Middleware → View → Service → Selector / Repository → DB
                         ↓
                    Serializer (input validation)
                         ↓
                    Serializer (output formatting)

Layers and their responsibilities:
┌─────────────┬──────────────────────────────────────────────────┐
│ Layer       │ Responsibility                                    │
├─────────────┼──────────────────────────────────────────────────┤
│ Middleware  │ Request/response lifecycle (auth, logging, CORS) │
│ View        │ Parse request → call service → serialize response│
│ Serializer  │ Validate input / format output                   │
│ Service     │ Business logic, orchestration, writes            │
│ Selector    │ Query logic, read-only, returns querysets        │
│ Model       │ Data structure + DB constraints + simple methods │
│ Task        │ Async execution of services                      │
├─────────────┼──────────────────────────────────────────────────┤
│ Cross-layer rules:                                             │
│ Views NEVER import from other app's views                      │
│ Services NEVER import from views or serializers                │
│ Selectors NEVER write to DB                                    │
│ Models NEVER call services                                     │
│ Tasks call services (not views)                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Feature-Based App Structure (Standard)

```python
# Each Django app = one business domain
# Ask: "Can I delete this app and have the rest still work?"
# If yes → good boundary. If no → it's a core dependency, not a feature.

apps/
├── core/              # Cross-cutting concerns — never a business domain
│   ├── models.py      # Abstract base models ONLY
│   ├── exceptions.py  # App-wide exception hierarchy
│   ├── middleware.py  # Request logging, request ID, etc.
│   ├── pagination.py  # DRF pagination classes
│   ├── permissions.py # Shared DRF permission classes
│   ├── validators.py  # Shared validators
│   ├── utils.py       # Pure functions — no Django ORM
│   └── cache.py       # Cache key helpers
│
├── users/             # User management
│   ├── models.py
│   ├── managers.py
│   ├── admin.py
│   ├── apps.py
│   ├── urls.py
│   ├── views.py       # HTTP handlers only
│   ├── serializers.py # Input/output shapes
│   ├── services.py    # Business logic (writes)
│   ├── selectors.py   # Query logic (reads)
│   ├── signals.py     # Signal handlers
│   ├── tasks.py       # Celery tasks
│   ├── permissions.py # User-specific permissions
│   ├── filters.py     # DRF filter classes
│   └── tests/
│       ├── __init__.py
│       ├── factories.py
│       ├── test_models.py
│       ├── test_services.py
│       ├── test_selectors.py
│       └── test_views.py
│
├── orders/            # Order management
├── products/          # Product catalog
├── payments/          # Payment processing
├── notifications/     # Notification center
├── chat/              # Real-time chat
├── reports/           # Reporting/analytics
└── integrations/      # External service integrations
```

---

## 3. Service Pattern — Full Example

```python
# apps/orders/services.py
"""
Service layer rules:
1. Function-based (unless stateful service object is needed)
2. Keyword-only arguments for clarity
3. Raises domain exceptions (not HTTP exceptions)
4. Wraps multi-step writes in transaction.atomic()
5. Triggers async work via transaction.on_commit()
6. Returns domain objects (model instances)
7. Never imports from views, serializers, or tasks (to avoid circular deps)
"""
from __future__ import annotations
import logging
from decimal import Decimal
from typing import Any
from django.db import transaction
from django.contrib.auth import get_user_model
from apps.core.exceptions import (
    ValidationError, NotFoundError, PermissionDeniedError, ServiceUnavailableError
)
from apps.orders.models import Order, OrderItem

logger = logging.getLogger(__name__)
User = get_user_model()


def place_order(
    *,
    user: User,
    items: list[dict[str, Any]],
    shipping_address_id: str,
    notes: str = "",
) -> Order:
    """
    Place a new order for a user.

    Args:
        user: The purchasing user.
        items: List of {product_id, quantity, unit_price}.
        shipping_address_id: UUID of the user's shipping address.
        notes: Optional order notes.

    Returns:
        The created Order instance.

    Raises:
        NotFoundError: If the shipping address doesn't belong to the user.
        ValidationError: If items are invalid.
        ServiceUnavailableError: If inventory service is unreachable.
    """
    from apps.addresses.selectors import get_user_address
    from apps.products.selectors import get_products_by_ids

    # Validate shipping address ownership
    address = get_user_address(user=user, address_id=shipping_address_id)
    if not address:
        raise NotFoundError("Shipping address not found.")

    # Validate products exist
    product_ids = [item["product_id"] for item in items]
    products = {str(p.pk): p for p in get_products_by_ids(product_ids)}
    for product_id in product_ids:
        if product_id not in products:
            raise NotFoundError(f"Product {product_id} not found.")

    # Calculate total
    total = sum(
        Decimal(str(item["quantity"])) * Decimal(str(item["unit_price"]))
        for item in items
    )

    with transaction.atomic():
        order = Order.objects.create(
            user=user,
            shipping_address=address,
            total=total,
            notes=notes,
            status=Order.Status.PENDING,
        )

        OrderItem.objects.bulk_create([
            OrderItem(
                order=order,
                product_id=item["product_id"],
                quantity=item["quantity"],
                unit_price=item["unit_price"],
            )
            for item in items
        ])

        logger.info(
            "Order placed",
            extra={"order_id": str(order.pk), "user_id": str(user.pk), "total": str(total)}
        )

        # Post-commit side effects (safe to fail independently)
        transaction.on_commit(lambda: _trigger_order_placed_tasks(str(order.pk)))

    return order


def _trigger_order_placed_tasks(order_id: str) -> None:
    from apps.orders.tasks import (
        send_order_confirmation_email,
        reserve_inventory,
        trigger_payment_processing,
    )
    send_order_confirmation_email.delay(order_id)
    reserve_inventory.delay(order_id)


def cancel_order(*, order: Order, cancelled_by: User) -> Order:
    """Cancel a pending order."""
    if order.status != Order.Status.PENDING:
        raise ValidationError(
            f"Cannot cancel order with status '{order.status}'."
        )
    if order.user_id != cancelled_by.pk and not cancelled_by.is_staff:
        raise PermissionDeniedError("You cannot cancel this order.")

    with transaction.atomic():
        order.status = Order.Status.CANCELLED
        order.save(update_fields=["status", "updated_at"])
        transaction.on_commit(lambda: _trigger_order_cancelled_tasks(str(order.pk)))

    return order
```

---

## 4. Selector Pattern — Full Example

```python
# apps/orders/selectors.py
"""
Selector rules:
1. ONLY read operations — never write, never call services
2. Return querysets (lazy) or model instances
3. Apply ALL optimization here (select_related, prefetch_related)
4. Named after the data they return
5. Accept typed parameters, not raw request objects
"""
from __future__ import annotations
from django.db.models import QuerySet, Q, Prefetch
from django.contrib.auth import get_user_model
from apps.orders.models import Order, OrderItem

User = get_user_model()


def get_user_orders(
    *,
    user: User,
    status: str | None = None,
    search: str | None = None,
) -> QuerySet[Order]:
    """Return all non-deleted orders for a user, optionally filtered."""
    qs = (
        Order.objects
        .filter(user=user, is_deleted=False)
        .select_related("shipping_address")
        .prefetch_related(
            Prefetch(
                "items",
                queryset=OrderItem.objects.select_related("product").only(
                    "id", "quantity", "unit_price", "product__name",
                )
            )
        )
        .order_by("-created_at")
    )
    if status:
        qs = qs.filter(status=status)
    if search:
        qs = qs.filter(
            Q(reference__icontains=search) |
            Q(items__product__name__icontains=search)
        ).distinct()
    return qs


def get_order_detail(*, order_id: str, user: User | None = None) -> Order | None:
    """Get a single order by ID, optionally scoped to a user."""
    qs = Order.objects.select_related(
        "user", "shipping_address"
    ).prefetch_related(
        Prefetch(
            "items",
            queryset=OrderItem.objects.select_related("product", "product__category")
        )
    )
    filters: dict = {"pk": order_id, "is_deleted": False}
    if user:
        filters["user"] = user
    return qs.filter(**filters).first()
```

---

## 5. Cross-App Communication Rules

```python
# Rule 1: Use selectors for read access across apps
# ✅ CORRECT — orders app reading from products app via selector
# apps/orders/services.py
from apps.products.selectors import get_products_by_ids

# Rule 2: Use signals for LOOSE coupling (cross-domain side effects)
# apps/orders/signals.py
order_placed = Signal()  # defined in orders

# apps/inventory/apps.py
def ready(self):
    from apps.orders.signals import order_placed
    order_placed.connect(reserve_inventory_on_order)

# Rule 3: Use events/tasks for async cross-domain work
# apps/orders/services.py → dispatches task
# apps/inventory/tasks.py → receives task args and acts

# Rule 4: NEVER circular imports between apps
# ❌ WRONG: orders imports from users AND users imports from orders
# ✅ CORRECT: use apps.get_model() for model references in signals/tasks

def get_order_model():
    from django.apps import apps
    return apps.get_model("orders", "Order")

# Rule 5: Shared data structures live in apps.core
# apps/core/types.py — TypedDicts, enums shared across apps
```

---

## 6. When to Create a New App

```
Create a new app when:
✅ The domain has its own models AND business logic
✅ The feature can be theoretically enabled/disabled independently
✅ The feature has its own URL space (/api/v1/feature/)
✅ The feature has its own background tasks
✅ A team could own this feature independently

Do NOT create a new app for:
❌ A single model with no business logic (add to closest related app)
❌ A utility module (goes in apps/core/)
❌ A type or enum shared by multiple apps (goes in apps/core/)
❌ A Django management command alone (goes in closest related app)

Merge apps when:
🔄 Two apps share all their models
🔄 One app's services always call the other's (tight coupling)
🔄 The split causes more confusion than it prevents
```

---

## 7. Model Design Principles

```python
# apps/orders/models.py
from __future__ import annotations
from django.db import models
from django.utils.translation import gettext_lazy as _
from apps.core.models import BaseModel


class Order(BaseModel):
    """
    Model design rules:
    1. Models represent data + DB constraints
    2. Methods on models: only computed properties and simple representations
    3. Business logic: services.py
    4. Query logic: selectors.py + managers.py
    """
    class Status(models.TextChoices):
        PENDING = "pending", _("Pending")
        PROCESSING = "processing", _("Processing")
        COMPLETED = "completed", _("Completed")
        CANCELLED = "cancelled", _("Cancelled")
        REFUNDED = "refunded", _("Refunded")

    user = models.ForeignKey(
        "users.User",
        on_delete=models.PROTECT,  # PROTECT: never cascade delete orders
        related_name="orders",
        db_index=True,
    )
    reference = models.CharField(max_length=20, unique=True, db_index=True)
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.PENDING,
        db_index=True,
    )
    total = models.DecimalField(max_digits=12, decimal_places=2)
    notes = models.TextField(blank=True, default="")

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
            )
        ]

    def __str__(self) -> str:
        return f"Order {self.reference} ({self.status})"

    @property
    def is_cancellable(self) -> bool:
        """Read-only computed property — OK on model."""
        return self.status == self.Status.PENDING

    # ❌ NEVER on model: def cancel(self): self.status = "cancelled"; self.save()
    # ✅ That belongs in services.cancel_order()
```

---

## 8. Dependency Injection Pattern

```python
# Django doesn't have a DI container but we can use patterns:

# Pattern 1: Configuration-driven backend (like Django's auth backends)
# config/settings/base.py
NOTIFICATION_BACKEND = env(
    "NOTIFICATION_BACKEND",
    default="apps.notifications.backends.EmailNotificationBackend"
)

# apps/notifications/backends.py
from abc import ABC, abstractmethod

class BaseNotificationBackend(ABC):
    @abstractmethod
    def send(self, recipient: str, subject: str, body: str) -> bool: ...

class EmailNotificationBackend(BaseNotificationBackend):
    def send(self, recipient: str, subject: str, body: str) -> bool:
        from django.core.mail import send_mail
        send_mail(subject, body, None, [recipient])
        return True

class SMSNotificationBackend(BaseNotificationBackend):
    def send(self, recipient: str, subject: str, body: str) -> bool:
        # Twilio integration
        ...

# apps/notifications/services.py
from django.conf import settings
from django.utils.module_loading import import_string

def get_notification_backend() -> BaseNotificationBackend:
    """Factory — returns configured backend."""
    backend_class = import_string(settings.NOTIFICATION_BACKEND)
    return backend_class()

def send_notification(recipient: str, subject: str, body: str) -> bool:
    backend = get_notification_backend()
    return backend.send(recipient, subject, body)
```

---

## 9. SOLID Principles in Django Context

```
S — Single Responsibility:
    - Views handle only HTTP
    - Services handle only business logic
    - Selectors handle only queries
    - Models handle only data representation

O — Open/Closed:
    - Use abstract base models to extend behavior
    - Backend pattern for pluggable implementations
    - Signal system for extensible event handling

L — Liskov Substitution:
    - All notification backends are interchangeable
    - All storage backends are interchangeable

I — Interface Segregation:
    - Separate serializers for input vs output
    - Separate serializers for list vs detail
    - Separate permissions per action (get_permissions)

D — Dependency Inversion:
    - Services depend on model interfaces, not concrete ORM calls
    - Backend pattern decouples service from implementation
    - Settings-driven configuration
```

---

## 10. Event-Driven Patterns

```python
# apps/core/events.py
"""
Event-driven communication between apps.
Use when: an action in app A should trigger work in app B,
but A should not directly depend on B.
"""
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Any
from django.db import transaction


@dataclass
class DomainEvent:
    """Base class for all domain events."""
    event_type: str
    aggregate_id: str
    payload: dict[str, Any] = field(default_factory=dict)
    metadata: dict[str, Any] = field(default_factory=dict)


def publish_event(event: DomainEvent) -> None:
    """
    Publish a domain event.
    In production: route to Celery task or Django signal.
    """
    import logging
    logger = logging.getLogger(__name__)
    logger.info(
        "Domain event published",
        extra={"event_type": event.event_type, "aggregate_id": event.aggregate_id}
    )
    # Route to Celery for async processing
    from apps.core.tasks import process_domain_event
    transaction.on_commit(
        lambda: process_domain_event.delay(
            event_type=event.event_type,
            aggregate_id=event.aggregate_id,
            payload=event.payload,
        )
    )


# Usage in services:
# def place_order(...):
#     with transaction.atomic():
#         order = Order.objects.create(...)
#         publish_event(DomainEvent(
#             event_type="order.placed",
#             aggregate_id=str(order.pk),
#             payload={"user_id": str(user.pk), "total": str(order.total)},
#         ))
```
