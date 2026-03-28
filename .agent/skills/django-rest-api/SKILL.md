---
name: django-rest-api
description: >
  Apply this skill for all REST API work: DRF serializers, viewsets, routers, 
  authentication (JWT, session, API key), permissions, filtering, pagination,
  versioning, error handling, and API design conventions. Use when building or
  reviewing any HTTP REST API endpoint in Django.
---

# Django REST API Skill

> FIRST: Use Context7 with library IDs `/websites/django-rest-framework` and
> `/websites/djangoproject_en_6_0` to get current DRF and Django 6 docs.

---

## 1. REST API Design Principles

```
Rules:
✅ Resources are nouns, never verbs   → /api/v1/orders/ NOT /api/v1/getOrders/
✅ Use plural nouns                   → /orders/ NOT /order/
✅ HTTP methods carry semantic meaning:
   GET    → Retrieve (idempotent, safe)
   POST   → Create
   PUT    → Replace entire resource (idempotent)
   PATCH  → Partial update (idempotent)
   DELETE → Remove (idempotent)
✅ Nested routes for ownership max 2 levels deep:
   /orders/{id}/items/         ← OK
   /orders/{id}/items/{id}/    ← OK
   /orders/{id}/items/{id}/sub/ ← NO — flatten with query param instead
✅ Version in URL path: /api/v1/ (not header, easier to debug)
✅ Filter/search/sort via query params: ?status=active&ordering=-created_at
✅ Use 201 for creation, 204 for deletion, 200 for updates
✅ Consistent error envelope (see section 8)
✅ Paginate ALL list endpoints — never return unbounded lists
```

---

## 2. DRF Global Configuration

```python
# config/settings/base.py
REST_FRAMEWORK = {
    # Authentication
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
    # Permissions — deny by default, open explicitly
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
    # Pagination — always paginate
    "DEFAULT_PAGINATION_CLASS": "apps.core.pagination.StandardResultsPagination",
    "PAGE_SIZE": 20,
    # Filtering
    "DEFAULT_FILTER_BACKENDS": [
        "django_filters.rest_framework.DjangoFilterBackend",
        "rest_framework.filters.SearchFilter",
        "rest_framework.filters.OrderingFilter",
    ],
    # Versioning
    "DEFAULT_VERSIONING_CLASS": "rest_framework.versioning.URLPathVersioning",
    "DEFAULT_VERSION": "v1",
    "ALLOWED_VERSIONS": ["v1", "v2"],
    # Exception handler
    "EXCEPTION_HANDLER": "apps.core.exceptions.custom_exception_handler",
    # Throttling
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "anon": "60/hour",
        "user": "1000/hour",
        "burst": "30/minute",
    },
    # Parser & Renderer
    "DEFAULT_PARSER_CLASSES": [
        "rest_framework.parsers.JSONParser",
        "rest_framework.parsers.MultiPartParser",
    ],
    "DEFAULT_RENDERER_CLASSES": [
        "rest_framework.renderers.JSONRenderer",
    ],
}
```

---

## 3. Serializers — Strict, Explicit, No Magic

```python
# apps/orders/serializers.py
from __future__ import annotations
from decimal import Decimal
from typing import Any
from rest_framework import serializers
from django.contrib.auth import get_user_model
from apps.orders.models import Order, OrderItem

User = get_user_model()


class OrderItemCreateSerializer(serializers.Serializer):
    """Serializer for creating order items. Validates business rules."""
    product_id = serializers.UUIDField()
    quantity = serializers.IntegerField(min_value=1, max_value=100)
    unit_price = serializers.DecimalField(max_digits=10, decimal_places=2, min_value=Decimal("0.01"))


class OrderCreateSerializer(serializers.Serializer):
    """
    Input serializer for order creation.
    Rule: Input serializers inherit from Serializer, not ModelSerializer,
    when the input shape differs from the model shape.
    """
    items = OrderItemCreateSerializer(many=True, min_length=1, max_length=50)
    notes = serializers.CharField(max_length=500, required=False, allow_blank=True)
    shipping_address_id = serializers.UUIDField()

    def validate_items(self, items: list[dict[str, Any]]) -> list[dict[str, Any]]:
        """Validate no duplicate product_ids in same order."""
        product_ids = [item["product_id"] for item in items]
        if len(product_ids) != len(set(str(p) for p in product_ids)):
            raise serializers.ValidationError("Duplicate products are not allowed.")
        return items


class OrderItemDetailSerializer(serializers.ModelSerializer):
    """Output serializer for order items."""
    product_name = serializers.CharField(source="product.name", read_only=True)

    class Meta:
        model = OrderItem
        fields = ["id", "product_id", "product_name", "quantity", "unit_price", "subtotal"]
        read_only_fields = fields


class OrderDetailSerializer(serializers.ModelSerializer):
    """
    Output serializer — always explicit fields list, never __all__.
    Nested serializers for related data.
    """
    items = OrderItemDetailSerializer(many=True, read_only=True)
    user_email = serializers.EmailField(source="user.email", read_only=True)
    status_display = serializers.CharField(source="get_status_display", read_only=True)

    class Meta:
        model = Order
        fields = [
            "id", "reference", "user_email", "status", "status_display",
            "total", "items", "notes", "created_at", "updated_at",
        ]
        read_only_fields = fields


class OrderListSerializer(serializers.ModelSerializer):
    """Lightweight serializer for list views — fewer fields = faster queries."""
    class Meta:
        model = Order
        fields = ["id", "reference", "status", "total", "created_at"]
        read_only_fields = fields
```

---

## 4. Custom Fields & Validators

```python
# apps/core/serializer_fields.py
from rest_framework import serializers
import re


class PhoneNumberField(serializers.CharField):
    """Validates and normalizes E.164 phone numbers."""
    def to_internal_value(self, data: object) -> str:
        value = super().to_internal_value(data)
        cleaned = re.sub(r"[\s\-\(\)]", "", str(value))
        if not re.match(r"^\+?[1-9]\d{6,14}$", cleaned):
            raise serializers.ValidationError("Enter a valid phone number.")
        return cleaned


class TimestampField(serializers.DateTimeField):
    """Return timestamps as Unix milliseconds for frontend convenience."""
    def to_representation(self, value: object) -> int | None:
        if value is None:
            return None
        dt = super().to_representation(value)
        import datetime
        if isinstance(value, datetime.datetime):
            return int(value.timestamp() * 1000)
        return dt
```

---

## 5. ViewSets — Structured, Documented

```python
# apps/orders/views.py
from __future__ import annotations
from rest_framework import viewsets, status, mixins
from rest_framework.decorators import action
from rest_framework.request import Request
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from django_filters.rest_framework import DjangoFilterBackend
from apps.core.exceptions import NotFoundError, PermissionDeniedError
from apps.orders import services, selectors
from apps.orders.serializers import (
    OrderCreateSerializer, OrderDetailSerializer, OrderListSerializer
)
from apps.orders.filters import OrderFilter
from apps.orders.permissions import IsOrderOwner


class OrderViewSet(
    mixins.CreateModelMixin,
    mixins.RetrieveModelMixin,
    mixins.ListModelMixin,
    viewsets.GenericViewSet,
):
    """
    Order management endpoints.

    list:    GET  /api/v1/orders/           → User's order list
    create:  POST /api/v1/orders/           → Place new order
    retrieve:GET  /api/v1/orders/{id}/      → Order detail
    cancel:  POST /api/v1/orders/{id}/cancel/ → Cancel order
    """
    permission_classes = [IsAuthenticated]
    filter_backends = [DjangoFilterBackend]
    filterset_class = OrderFilter

    def get_queryset(self):
        # SECURITY: Always scope to authenticated user
        return selectors.get_user_orders(user=self.request.user)

    def get_serializer_class(self):
        if self.action == "create":
            return OrderCreateSerializer
        if self.action == "list":
            return OrderListSerializer
        return OrderDetailSerializer

    def get_permissions(self):
        if self.action in ("retrieve", "cancel"):
            return [IsAuthenticated(), IsOrderOwner()]
        return [IsAuthenticated()]

    def create(self, request: Request, *args: object, **kwargs: object) -> Response:
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        try:
            order = services.place_order(
                user=request.user,
                **serializer.validated_data
            )
        except NotFoundError as exc:
            return Response({"error": exc.message}, status=status.HTTP_404_NOT_FOUND)

        return Response(
            OrderDetailSerializer(order).data,
            status=status.HTTP_201_CREATED,
        )

    @action(detail=True, methods=["post"], url_path="cancel")
    def cancel(self, request: Request, pk: str | None = None) -> Response:
        """Cancel an order. Only allowed if status is PENDING."""
        order = self.get_object()  # Handles 404 + object permissions

        try:
            updated = services.cancel_order(order=order, cancelled_by=request.user)
        except PermissionDeniedError as exc:
            return Response({"error": exc.message}, status=status.HTTP_403_FORBIDDEN)

        return Response(OrderDetailSerializer(updated).data)
```

---

## 6. Routing

```python
# apps/orders/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from apps.orders.views import OrderViewSet

app_name = "orders"

router = DefaultRouter()
router.register("", OrderViewSet, basename="order")

urlpatterns = [
    path("", include(router.urls)),
]
```

---

## 7. Permissions

```python
# apps/orders/permissions.py
from rest_framework.permissions import BasePermission
from rest_framework.request import Request
from rest_framework.views import APIView


class IsOrderOwner(BasePermission):
    """Allow access only if the order belongs to the requesting user."""
    message = "You do not have permission to access this order."

    def has_object_permission(self, request: Request, view: APIView, obj: object) -> bool:
        return obj.user_id == request.user.pk


class IsAdminOrReadOnly(BasePermission):
    """Read-only for authenticated users, write for admins."""
    def has_permission(self, request: Request, view: APIView) -> bool:
        if request.method in ("GET", "HEAD", "OPTIONS"):
            return request.user and request.user.is_authenticated
        return request.user and request.user.is_staff
```

---

## 8. Custom Exception Handler — Consistent Error Envelope

```python
# apps/core/exceptions.py
from __future__ import annotations
import logging
from typing import Any
from rest_framework.views import exception_handler
from rest_framework.response import Response
from rest_framework import status
from rest_framework.request import Request

logger = logging.getLogger(__name__)


def custom_exception_handler(exc: Exception, context: dict[str, Any]) -> Response | None:
    """
    Standardized error response format:
    {
        "error": {
            "code": "validation_error",
            "message": "Human-readable message",
            "details": {...}  # optional field-level errors
        }
    }
    """
    response = exception_handler(exc, context)

    if response is not None:
        # DRF handled it — reformat
        error_data: dict[str, Any] = {
            "code": _get_error_code(exc, response),
            "message": _get_error_message(exc, response),
        }
        if isinstance(response.data, dict) and "detail" not in response.data:
            error_data["details"] = response.data
        response.data = {"error": error_data}
    else:
        # Unhandled exception — 500
        logger.exception("Unhandled exception in API view", exc_info=exc)
        response = Response(
            {"error": {"code": "internal_error", "message": "An unexpected error occurred."}},
            status=status.HTTP_500_INTERNAL_SERVER_ERROR,
        )

    return response


def _get_error_code(exc: Exception, response: Response) -> str:
    if hasattr(exc, "default_code"):
        return exc.default_code
    return {
        400: "bad_request",
        401: "unauthorized",
        403: "forbidden",
        404: "not_found",
        405: "method_not_allowed",
        429: "rate_limited",
        500: "internal_error",
    }.get(response.status_code, "error")


def _get_error_message(exc: Exception, response: Response) -> str:
    if hasattr(exc, "detail") and isinstance(exc.detail, str):
        return exc.detail
    if hasattr(exc, "message"):
        return exc.message
    return str(exc)
```

---

## 9. Pagination

```python
# apps/core/pagination.py
from rest_framework.pagination import PageNumberPagination
from rest_framework.response import Response
from typing import Any


class StandardResultsPagination(PageNumberPagination):
    """
    Standard pagination returning:
    {
        "count": 100,
        "next": "...",
        "previous": "...",
        "page": 1,
        "total_pages": 5,
        "results": [...]
    }
    """
    page_size = 20
    page_size_query_param = "page_size"
    max_page_size = 100
    page_query_param = "page"

    def get_paginated_response(self, data: list[Any]) -> Response:
        return Response({
            "count": self.page.paginator.count,
            "next": self.get_next_link(),
            "previous": self.get_previous_link(),
            "page": self.page.number,
            "total_pages": self.page.paginator.num_pages,
            "results": data,
        })

    def get_paginated_response_schema(self, schema: dict[str, Any]) -> dict[str, Any]:
        return {
            "type": "object",
            "properties": {
                "count": {"type": "integer"},
                "next": {"type": "string", "nullable": True},
                "previous": {"type": "string", "nullable": True},
                "page": {"type": "integer"},
                "total_pages": {"type": "integer"},
                "results": schema,
            },
        }


class CursorResultsPagination(PageNumberPagination):
    """Cursor-based pagination for real-time feeds. Stable under insertions."""
    from rest_framework.pagination import CursorPagination
    ordering = "-created_at"
    page_size = 20
    cursor_query_param = "cursor"
```

---

## 10. Filtering

```python
# apps/orders/filters.py
import django_filters
from apps.orders.models import Order


class OrderFilter(django_filters.FilterSet):
    status = django_filters.MultipleChoiceFilter(choices=Order.Status.choices)
    created_after = django_filters.DateTimeFilter(field_name="created_at", lookup_expr="gte")
    created_before = django_filters.DateTimeFilter(field_name="created_at", lookup_expr="lte")
    min_total = django_filters.NumberFilter(field_name="total", lookup_expr="gte")
    max_total = django_filters.NumberFilter(field_name="total", lookup_expr="lte")

    class Meta:
        model = Order
        fields = ["status", "created_after", "created_before", "min_total", "max_total"]
```

---

## 11. JWT Authentication Setup

```python
# config/settings/base.py
from datetime import timedelta

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),  # Short-lived
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "ROTATE_REFRESH_TOKENS": True,
    "BLACKLIST_AFTER_ROTATION": True,
    "UPDATE_LAST_LOGIN": True,
    "ALGORITHM": "HS256",
    "SIGNING_KEY": env("JWT_SIGNING_KEY"),  # NEVER use SECRET_KEY for JWT
    "AUTH_HEADER_TYPES": ("Bearer",),
    "AUTH_HEADER_NAME": "HTTP_AUTHORIZATION",
    "USER_ID_FIELD": "id",
    "USER_ID_CLAIM": "user_id",
    "TOKEN_OBTAIN_SERIALIZER": "apps.users.serializers.CustomTokenObtainPairSerializer",
}

# apps/users/serializers.py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        # Add custom claims
        token["email"] = user.email
        token["role"] = user.role if hasattr(user, "role") else "user"
        return token
```

---

## 12. API Versioning Pattern

```python
# apps/orders/views.py — version-aware view
class OrderViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        version = getattr(self.request, "version", "v1")
        if version == "v2":
            return OrderDetailSerializerV2
        return OrderDetailSerializer

# URL setup for versioned routes:
# /api/v1/orders/
# /api/v2/orders/
urlpatterns = [
    path("api/<str:version>/", include("config.api_router")),
]
```

---

## 13. API Security Checklist

```
✅ All endpoints behind authentication (deny by default in settings)
✅ Object-level permissions on all detail views
✅ Rate limiting on all endpoints, stricter on auth endpoints
✅ Input validation via serializers (never trust raw request.data)
✅ CORS configured (django-cors-headers) — explicit allowed origins
✅ HTTPS enforced (SECURE_SSL_REDIRECT=True in production)
✅ Sensitive fields (password, token) never in serializer output
✅ IDs are UUIDs (don't expose sequential integer PKs)
✅ No stack traces in error responses (custom exception handler)
✅ Request body size limit (DATA_UPLOAD_MAX_MEMORY_SIZE in settings)
```

---

## 14. Throttling — Custom Rates Per View

```python
from rest_framework.throttling import UserRateThrottle, AnonRateThrottle

class LoginRateThrottle(AnonRateThrottle):
    """Strict throttle for login endpoint — prevent brute force."""
    scope = "login"
    # In settings: "DEFAULT_THROTTLE_RATES": {"login": "5/minute"}

class AuthTokenView(APIView):
    throttle_classes = [LoginRateThrottle]
    permission_classes = []  # Public endpoint
```

---

## 15. Response Envelope Standards

```json
// Success — single resource
{
  "data": { "id": "...", "email": "..." }
}

// Success — list
{
  "count": 100,
  "next": "...",
  "previous": null,
  "page": 1,
  "total_pages": 5,
  "results": [...]
}

// Error
{
  "error": {
    "code": "validation_error",
    "message": "The submitted data is invalid.",
    "details": {
      "email": ["Enter a valid email address."],
      "password": ["This field is required."]
    }
  }
}
```
