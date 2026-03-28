---
name: django-drf-spectacular
description: >
  Apply this skill automatically whenever the project has a REST API
  (i.e., djangorestframework is installed or any ViewSet/APIView exists).
  Covers: full installation and settings, environment-aware schema serving,
  JWT security scheme registration, standard error envelope documentation,
  the @extend_schema decision tree (when it IS vs IS NOT needed), @extend_schema_view
  for ViewSets, inline_serializer, extend_schema_field, OpenApiParameter,
  OpenApiResponse, OpenApiExample, custom pagination response shape, polymorphic
  serializers, CI/CD schema validation, and the sidecar offline setup.
  Also covers: what NOT to document (health checks, internal endpoints),
  operation_id collision prevention for versioned APIs, and tag strategy.
  Use when building any endpoint, writing any serializer, or reviewing any view.
---

# DRF Spectacular — OpenAPI Schema Skill

> FIRST: Use Context7 with library ID `/tfranzel/drf-spectacular` to get the
> latest settings names and decorator API before implementing anything.
> drf-spectacular releases frequently — do not assume training-data API is current.

---

## ⚙️ Reasoning & Invalidation Log

The following decisions were reached through explicit invalidation loops.
Each rule below has a documented reason — do not change them without re-running
the invalidation logic.

```
DECISION 1: SERVE_PUBLIC = False in production
  Initial: True for easy developer access
  Invalidated: Exposes full API surface area publicly → attacker reconnaissance
  Solution: env-var-driven; True only in DEBUG, False in production

DECISION 2: COMPONENT_SPLIT_REQUEST = True (always)
  Initial: Optional setting, skip for simplicity
  Invalidated: Without it, read_only_fields appear as required in POST bodies.
               Client generators produce wrong request types.
  Solution: Always True — docs explicitly call it "highly recommended"

DECISION 3: Create standard ErrorResponseSerializer
  Initial: Document only happy-path responses per endpoint
  Invalidated: Custom exception handler (apps.core.exceptions) produces an
               envelope {error: {code, message, details}} that AutoSchema
               can never infer. Client generators produce wrong error types.
  Solution: Define ONCE as a reusable OpenApiResponse; attach to every endpoint

DECISION 4: CI schema validation step
  Initial: Validate manually before releases
  Invalidated: Schema drift silently breaks clients — devs forget @extend_schema,
               types change, fields are removed. No PR gate = no safety net.
  Solution: `manage.py spectacular --validate` as mandatory CI step

DECISION 5: JWT security extension required
  Initial: Rely on DEFAULT_AUTHENTICATION_CLASSES auto-detection
  Invalidated: SimpleJWT's JWTAuthentication is not natively known to spectacular.
               Without explicit extension, the Swagger "Authorize" button never
               appears and no security scheme is emitted.
  Solution: Register OpenApiAuthenticationExtension for JWTAuthentication

DECISION 6: Sidecar for offline assets
  Initial: Serve Swagger/Redoc from CDN
  Invalidated: CSP headers block CDN JS in production. Air-gapped envs can't
               reach CDN at all. CDN outage = broken docs.
  Solution: drf-spectacular-sidecar; controlled by SPECTACULAR_USE_SIDECAR env var

DECISION 7: Annotate @action endpoints always
  Initial: AutoSchema handles everything
  Invalidated: Custom @action methods always need @extend_schema — AutoSchema
               cannot infer the response shape from a non-CRUD method name.
  Solution: The decision tree below is MANDATORY for the agent

DECISION 8: Exclude health/internal endpoints
  Initial: Document everything
  Invalidated: Health check, readiness probe, internal debug views pollute the
               schema and confuse API consumers. exclude=True is the right tool.
  Solution: Use @extend_schema(exclude=True) on non-API utility views

DECISION 9: SCHEMA_PATH_PREFIX strips versioning
  Initial: Let path prefix auto-detect
  Invalidated: /api/v1/orders/ and /api/v2/orders/ both generate tag "v1" or
               nothing useful. Tag extraction breaks. Operation IDs collide.
  Solution: SCHEMA_PATH_PREFIX = r'/api/v[0-9]+' strips version from tag extraction

DECISION 10: Custom pagination shape must be explicit
  Initial: drf-spectacular auto-detects pagination
  Invalidated: Standard PageNumberPagination is detected, but our custom
               StandardResultsPagination adds {page, total_pages} fields
               that AutoSchema never discovers.
  Solution: Postprocessing hook or explicit response annotation on list endpoints
```

---

## 1. When to Apply This Skill

```
This skill activates when ANY of these are true:
✅ djangorestframework is in INSTALLED_APPS
✅ Any file contains APIView, ViewSet, @api_view
✅ REST_FRAMEWORK exists in settings.py
✅ Any URL routes to a DRF view

Auto-setup rule (for the AI agent):
When building a new endpoint → also write @extend_schema annotations.
When creating a new app with views → also add to TAGS in SPECTACULAR_SETTINGS.
When adding a new serializer for responses → also define its error counterpart.
```

---

## 2. Installation

```
# requirements/base.txt
drf-spectacular
drf-spectacular-sidecar    # Self-hosted UI assets (offline / CSP-safe)
```

---

## 3. Settings — Full Production Configuration

```python
# config/settings/base.py
import environ
env = environ.Env()

INSTALLED_APPS = [
    ...
    "rest_framework",
    "drf_spectacular",
    "drf_spectacular_sidecar",  # For offline / CSP-safe UI assets
    ...
]

REST_FRAMEWORK = {
    # ── CRITICAL: This must be set for drf-spectacular to work ────
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
    # ... rest of your DRF settings ...
}

# ── Spectacular: environment-driven configuration ─────────────────
_USE_SIDECAR = env.bool("SPECTACULAR_USE_SIDECAR", default=not env.bool("DJANGO_DEBUG", default=False))

SPECTACULAR_SETTINGS = {
    # ── Metadata ──────────────────────────────────────────────────
    "TITLE": env("API_TITLE", default="Project API"),
    "DESCRIPTION": env("API_DESCRIPTION", default="REST API documentation"),
    "VERSION": env("API_VERSION", default="1.0.0"),
    "CONTACT": {
        "name": env("API_CONTACT_NAME", default="API Support"),
        "email": env("API_CONTACT_EMAIL", default=""),
    },
    "TERMS_OF_SERVICE": env("API_TERMS_URL", default=""),

    # ── Schema accuracy ────────────────────────────────────────────
    # DECISION 2: Always True — separates request (writable) from
    # response (readable) components so read_only_fields are not shown
    # as required in POST body docs.
    "COMPONENT_SPLIT_REQUEST": True,
    "COMPONENT_SPLIT_PATCH": True,
    # Adds minLength:1 to non-blank fields in request components
    "ENFORCE_NON_BLANK_FIELDS": True,

    # ── OpenAPI version ────────────────────────────────────────────
    "OAS_VERSION": "3.0.3",

    # ── Path / tag extraction ──────────────────────────────────────
    # DECISION 9: Strip the version prefix from tag extraction so
    # /api/v1/orders/ produces tag "orders" not "v1"
    "SCHEMA_PATH_PREFIX": r"/api/v[0-9]+",
    "SCHEMA_PATH_PREFIX_TRIM": True,
    # Coerce {user_pk} → {user_id} for nested routers
    "SCHEMA_COERCE_PATH_PK_SUFFIX": True,

    # ── Tags (add one entry per Django app with public API) ─────────
    # Rule: Keep in sync with LOCAL_APPS. One tag per domain app.
    "TAGS": [
        {"name": "Users", "description": "User account management"},
        {"name": "Auth", "description": "Authentication and token management"},
        {"name": "Orders", "description": "Order lifecycle management"},
        # Add new app tags here when creating a new app
    ],

    # RULE: Include ONLY the current environment's server.
    # Do NOT list all environments — it exposes staging/dev URLs in prod docs.
    "SERVERS": [
        {
            "url": env("API_BASE_URL", default="http://localhost:8000"),
            "description": env("ENVIRONMENT", default="development").capitalize() + " server",
        },
    ],

    # ── Security: Never expose schema publicly in production ────────
    # DECISION 1: Schema is only available to authenticated users in prod
    "SERVE_PERMISSIONS": (
        ["rest_framework.permissions.AllowAny"]
        if env.bool("DJANGO_DEBUG", default=False)
        else ["rest_framework.permissions.IsAuthenticated"]
    ),
    "SERVE_AUTHENTICATION": None,   # Inherits DEFAULT_AUTHENTICATION_CLASSES
    "SERVE_PUBLIC": env.bool("DJANGO_DEBUG", default=False),

    # DECISION 4 (CI): Never include the schema endpoint itself in the schema
    "SERVE_INCLUDE_SCHEMA": False,

    # ── Post-processing ────────────────────────────────────────────
    "POSTPROCESSING_HOOKS": [
        "drf_spectacular.hooks.postprocess_schema_enums",
        # Custom hook to inject standard pagination wrapper on list responses
        "apps.core.schema_hooks.add_pagination_envelope",
    ],

    # ── UI assets: sidecar in production, CDN in development ────────
    # DECISION 6: sidecar = self-hosted, no CDN dependency, CSP-safe
    "SWAGGER_UI_DIST": "SIDECAR" if _USE_SIDECAR else "https://cdn.jsdelivr.net/npm/swagger-ui-dist@latest",
    "SWAGGER_UI_FAVICON_HREF": "SIDECAR" if _USE_SIDECAR else "https://cdn.jsdelivr.net/npm/swagger-ui-dist@latest/favicon-32x32.png",
    "REDOC_DIST": "SIDECAR" if _USE_SIDECAR else "https://cdn.jsdelivr.net/npm/redoc@latest",

    # ── Swagger UI behaviour ───────────────────────────────────────
    "SWAGGER_UI_SETTINGS": {
        "deepLinking": True,
        # SECURITY: persistAuthorization stores JWT in localStorage.
        # Only enable in development — in production, XSS risk is real.
        "persistAuthorization": env.bool("DJANGO_DEBUG", default=False),
        "displayOperationId": False,
        "filter": True,                 # Enable search box
        "defaultModelsExpandDepth": 2,
        "defaultModelExpandDepth": 2,
        # SECURITY: Disabled in prod — prevents accidental live API calls from docs.
        "tryItOutEnabled": False,
    },
}
```

---

## 4. URL Configuration

```python
# config/urls.py
from django.urls import path, include
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
    SpectacularRedocView,
)

urlpatterns = [
    path("api/", include("config.api_router")),

    # Schema URLs are ALWAYS registered — access is controlled by
    # SERVE_PERMISSIONS in settings.py, not by URL presence.
    #
    # ✅ Production: SERVE_PERMISSIONS = ["IsAuthenticated"]
    #    → Unauthenticated access → 401. JWT-authenticated users see docs.
    # ✅ Development: SERVE_PERMISSIONS = ["AllowAny"]
    #    → Anyone on localhost sees docs.
    #
    # Keeping URLs always registered also means:
    #   - `manage.py spectacular --validate` always works in CI
    #   - Internal tooling (Postman, client generators) can always fetch schema
    path("api/schema/", SpectacularAPIView.as_view(), name="schema"),
    path(
        "api/schema/swagger-ui/",
        SpectacularSwaggerView.as_view(url_name="schema"),
        name="swagger-ui",
    ),
    path(
        "api/schema/redoc/",
        SpectacularRedocView.as_view(url_name="schema"),
        name="redoc",
    ),
]
```

---

## 5. JWT Security Extension — REQUIRED for Swagger "Authorize" Button

```python
# apps/core/schema_extensions.py
"""
DECISION 5: SimpleJWT is not natively known to drf-spectacular.
Without this, the OpenAPI schema has no security scheme, the Swagger
"Authorize" button never appears, and generated clients don't know
how to authenticate.

Import this module in AppConfig.ready() or at the bottom of settings.py
so it registers before schema generation runs.

RULE: Add one extension per authentication class your project uses.
"""
from drf_spectacular.extensions import OpenApiAuthenticationExtension
from drf_spectacular.plumbing import build_bearer_security_scheme_object


class SimpleJWTAuthenticationExtension(OpenApiAuthenticationExtension):
    """Register SimpleJWT as a Bearer/JWT security scheme."""
    target_class = "rest_framework_simplejwt.authentication.JWTAuthentication"
    name = "jwtAuth"
    priority = -1

    def get_security_definition(self, auto_schema: object) -> dict:
        return build_bearer_security_scheme_object(
            header_name="AUTHORIZATION",
            token_prefix="Bearer",
            bearer_format="JWT",
        )


# ── Generic templates — add whichever your project uses ──────────────

class KnoxTokenAuthenticationScheme(OpenApiAuthenticationExtension):
    """Knox multi-token auth (if using django-rest-knox)."""
    target_class = "knox.auth.TokenAuthentication"
    name = "knoxAuth"

    def get_security_definition(self, auto_schema: object) -> dict:
        return build_bearer_security_scheme_object(
            header_name="AUTHORIZATION",
            token_prefix="Token",
            bearer_format="Knox",
        )


class APIKeyAuthenticationScheme(OpenApiAuthenticationExtension):
    """Custom API key header auth (if using a custom auth class)."""
    target_class = "apps.integrations.authentication.APIKeyAuthentication"
    name = "apiKeyAuth"

    def get_security_definition(self, auto_schema: object) -> dict:
        return {
            "type": "apiKey",
            "in": "header",
            "name": "X-API-KEY",
        }


# apps/core/apps.py
from django.apps import AppConfig

class CoreConfig(AppConfig):
    name = "apps.core"

    def ready(self) -> None:
        import apps.core.schema_extensions  # noqa: F401 — registers all auth schemes
        import apps.core.signals             # noqa: F401
```

---

## 6. Standard Error Response Schemas

```python
# apps/core/schema_responses.py
"""
DECISION 3: Our custom_exception_handler returns a standard envelope:
  {"error": {"code": "...", "message": "...", "details": {...}}}

AutoSchema can NEVER infer this shape. Without explicit documentation
every endpoint shows generic 400 responses, and client generators
produce wrong error-handling code.

Use these pre-built OpenApiResponse objects on every ViewSet.
"""
from __future__ import annotations
from rest_framework import serializers
from drf_spectacular.utils import OpenApiResponse, inline_serializer
from drf_spectacular.types import OpenApiTypes


# ── Error envelope serializer (matches apps.core.exceptions handler) ──

class ErrorDetailSerializer(serializers.Serializer):
    code = serializers.CharField(help_text="Machine-readable error code")
    message = serializers.CharField(help_text="Human-readable error message")
    details = serializers.DictField(
        child=serializers.ListField(child=serializers.CharField()),
        required=False,
        help_text="Field-level validation errors",
    )


class ErrorEnvelopeSerializer(serializers.Serializer):
    error = ErrorDetailSerializer()


# ── Reusable response objects — use these in @extend_schema responses ──

RESPONSE_400 = OpenApiResponse(
    response=ErrorEnvelopeSerializer,
    description="Validation error — one or more input fields are invalid.",
)

RESPONSE_401 = OpenApiResponse(
    response=ErrorEnvelopeSerializer,
    description="Unauthorized — valid authentication credentials are required.",
)

RESPONSE_403 = OpenApiResponse(
    response=ErrorEnvelopeSerializer,
    description="Forbidden — you do not have permission to perform this action.",
)

RESPONSE_404 = OpenApiResponse(
    response=ErrorEnvelopeSerializer,
    description="Not found — the requested resource does not exist.",
)

RESPONSE_429 = OpenApiResponse(
    response=ErrorEnvelopeSerializer,
    description="Too many requests — rate limit exceeded.",
)

RESPONSE_500 = OpenApiResponse(
    response=ErrorEnvelopeSerializer,
    description="Internal server error — an unexpected error occurred.",
)

# ── Convenience dict for CRUD endpoints ───────────────────────────────

STANDARD_ERRORS: dict = {
    400: RESPONSE_400,
    401: RESPONSE_401,
    403: RESPONSE_403,
    404: RESPONSE_404,
    429: RESPONSE_429,
}

STANDARD_LIST_ERRORS: dict = {
    401: RESPONSE_401,
    403: RESPONSE_403,
    429: RESPONSE_429,
}
```

---

## 7. Custom Pagination Hook

```python
# apps/core/schema_hooks.py
"""
DECISION 10: Our StandardResultsPagination adds {page, total_pages}
fields that AutoSchema never discovers. This hook injects the correct
wrapper schema around all list responses.
"""
from __future__ import annotations
from typing import Any


def add_pagination_envelope(result: dict[str, Any], generator: Any, **kwargs: Any) -> dict[str, Any]:
    """
    Post-processing hook that wraps list operation responses in the
    standard pagination envelope matching StandardResultsPagination.
    """
    schemas = result.get("components", {}).get("schemas", {})

    for path_data in result.get("paths", {}).values():
        for operation in path_data.values():
            if not isinstance(operation, dict):
                continue
            # Only wrap list operations (GET with pagination)
            if operation.get("x-paginated"):
                for status_code, response in operation.get("responses", {}).items():
                    if status_code == "200" and "content" in response:
                        for media_type, media_data in response["content"].items():
                            original_schema = media_data.get("schema", {})
                            media_data["schema"] = {
                                "type": "object",
                                "properties": {
                                    "count": {"type": "integer", "description": "Total item count"},
                                    "next": {"type": "string", "nullable": True, "format": "uri"},
                                    "previous": {"type": "string", "nullable": True, "format": "uri"},
                                    "page": {"type": "integer", "description": "Current page number"},
                                    "total_pages": {"type": "integer"},
                                    "results": original_schema,
                                },
                                "required": ["count", "page", "total_pages", "results"],
                            }
    return result
```

---

## 8. The @extend_schema Decision Tree — MANDATORY

```
Before writing any view, run this decision tree for EACH HTTP method:

ViewSet or GenericViewSet?
  → Use @extend_schema_view(list=..., retrieve=..., create=...) at CLASS level
  → Use @extend_schema directly on individual @action methods

Plain APIView (no action names — only HTTP methods)?
  → Use @extend_schema on each HTTP method handler: get / post / put / patch / delete
  → Do NOT use @extend_schema_view — it only works for ViewSet action names

Function-based view with @api_view?
  → Place @extend_schema ABOVE the @api_view decorator

Is this a standard CRUD action on a ModelViewSet?
(list/retrieve/create/update/partial_update/destroy)
  AND does the serializer_class cover the full request AND response?
  AND are all response codes standard (200/201/204/400/401/403/404)?
  → AutoSchema handles it. Add standard error responses via @extend_schema_view.
    Still add summary= and tags= at the ViewSet level.

Is this a @action method? (custom endpoint)
  → ALWAYS annotate with @extend_schema.
    AutoSchema cannot infer response shape from a non-CRUD method name.

Does the endpoint use DIFFERENT serializers for request vs response?
(e.g., OrderCreateSerializer in, OrderDetailSerializer out)
  → Annotate: @extend_schema(request=InputSer, responses={201: OutputSer})

Does the endpoint return a non-serializer response?
(e.g., {"status": "ok"} or a plain string or a file download)
  → Annotate: use inline_serializer or OpenApiTypes or responses={200: None}
  → For binary files: use responses={(200, "application/pdf"): OpenApiResponse(...)}

Does the endpoint accept query parameters NOT on the serializer?
(filters, ordering, search terms)
  → Annotate: add OpenApiParameter entries

Are there multiple possible response shapes? (polymorphic)
  → Annotate: use PolymorphicProxySerializer

Is this a DELETE endpoint?
  → Use responses={204: None, 401: RESPONSE_401, 403: RESPONSE_403, 404: RESPONSE_404}
  → Never use 200 for a DELETE response

Is this a utility/internal endpoint?
(health check, readiness probe, admin-only diagnostic)
  → Exclude: @extend_schema(exclude=True)

RULE: When in doubt, annotate. Over-documentation is recoverable.
      Under-documentation silently breaks client generators.
```

---

## 9. ViewSet Annotation — Full Production Pattern

```python
# apps/orders/views.py
from __future__ import annotations
from rest_framework import viewsets, status, mixins
from rest_framework.decorators import action
from rest_framework.request import Request
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from drf_spectacular.utils import (
    extend_schema,
    extend_schema_view,
    OpenApiParameter,
    OpenApiResponse,
    OpenApiExample,
)
from drf_spectacular.types import OpenApiTypes
from apps.core.schema_responses import (
    RESPONSE_400, RESPONSE_401, RESPONSE_403, RESPONSE_404, RESPONSE_429,
)
from apps.orders.serializers import (
    OrderCreateSerializer,
    OrderDetailSerializer,
    OrderListSerializer,
)


# PATTERN: Use @extend_schema_view to annotate all standard actions
# at the class level. This keeps action logic clean.
@extend_schema_view(
    list=extend_schema(
        summary="List orders",
        description="Returns a paginated list of the authenticated user's orders.",
        tags=["Orders"],
        responses={
            200: OrderListSerializer,
            **{k: v for k, v in {401: RESPONSE_401, 403: RESPONSE_403, 429: RESPONSE_429}.items()},
        },
        # Mark as paginated so the pagination hook wraps the response
        extensions={"x-paginated": True},
        parameters=[
            OpenApiParameter(
                name="status",
                type=str,
                location=OpenApiParameter.QUERY,
                description="Filter by order status",
                enum=["pending", "processing", "completed", "cancelled"],
                required=False,
            ),
            OpenApiParameter(
                name="created_after",
                type=OpenApiTypes.DATETIME,
                location=OpenApiParameter.QUERY,
                description="ISO 8601 datetime — return orders created after this time",
                required=False,
            ),
            OpenApiParameter(
                name="ordering",
                type=str,
                location=OpenApiParameter.QUERY,
                description="Sort field. Prefix with `-` for descending.",
                enum=["created_at", "-created_at", "total", "-total"],
                required=False,
            ),
        ],
    ),
    retrieve=extend_schema(
        summary="Get order detail",
        description="Returns the full detail of a single order owned by the authenticated user.",
        tags=["Orders"],
        responses={
            200: OrderDetailSerializer,
            401: RESPONSE_401,
            403: RESPONSE_403,
            404: RESPONSE_404,
        },
    ),
    create=extend_schema(
        summary="Place order",
        description="Creates a new order. Returns the created order detail.",
        tags=["Orders"],
        request=OrderCreateSerializer,
        responses={
            201: OrderDetailSerializer,
            400: RESPONSE_400,
            401: RESPONSE_401,
            404: RESPONSE_404,
        },
        examples=[
            OpenApiExample(
                "Minimal order",
                request_only=True,
                value={
                    "shipping_address_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
                    "items": [{"product_id": "abc123", "quantity": 2, "unit_price": "9.99"}],
                },
            ),
        ],
    ),
)
class OrderViewSet(
    mixins.CreateModelMixin,
    mixins.RetrieveModelMixin,
    mixins.ListModelMixin,
    viewsets.GenericViewSet,
):
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        from apps.orders import selectors
        return selectors.get_user_orders(user=self.request.user)

    def get_serializer_class(self):
        if self.action == "create":
            return OrderCreateSerializer
        if self.action == "list":
            return OrderListSerializer
        return OrderDetailSerializer

    # DECISION TREE RESULT: @action → ALWAYS annotate
    @extend_schema(
        summary="Cancel order",
        description=(
            "Cancels a pending order. Only orders with status `pending` can be cancelled. "
            "Returns the updated order."
        ),
        tags=["Orders"],
        request=None,   # No request body for this action
        responses={
            200: OrderDetailSerializer,
            400: RESPONSE_400,   # If order is not in cancellable state
            401: RESPONSE_401,
            403: RESPONSE_403,
            404: RESPONSE_404,
        },
    )
    @action(detail=True, methods=["post"], url_path="cancel")
    def cancel(self, request: Request, pk: str | None = None) -> Response:
        from apps.orders import services
        from apps.core.exceptions import ValidationError, PermissionDeniedError
        order = self.get_object()
        try:
            updated = services.cancel_order(order=order, cancelled_by=request.user)
        except PermissionDeniedError as exc:
            return Response({"error": {"code": exc.code, "message": exc.message}}, status=403)
        except ValidationError as exc:
            return Response({"error": {"code": exc.code, "message": exc.message}}, status=400)
        return Response(OrderDetailSerializer(updated).data)
```

---

## 10. Annotating SerializerMethodField and Custom Fields

```python
# apps/orders/serializers.py
from __future__ import annotations
from rest_framework import serializers
from drf_spectacular.utils import extend_schema_field
from drf_spectacular.types import OpenApiTypes


class OrderDetailSerializer(serializers.ModelSerializer):
    status_display = serializers.SerializerMethodField()
    item_count = serializers.SerializerMethodField()
    is_cancellable = serializers.SerializerMethodField()
    coupon_code = serializers.SerializerMethodField()   # nullable example

    # RULE: Always annotate SerializerMethodField — AutoSchema infers only "any"
    @extend_schema_field(OpenApiTypes.STR)
    def get_status_display(self, obj: object) -> str:
        return obj.get_status_display()

    @extend_schema_field(OpenApiTypes.INT)
    def get_item_count(self, obj: object) -> int:
        return obj.items.count()

    @extend_schema_field(OpenApiTypes.BOOL)
    def get_is_cancellable(self, obj: object) -> bool:
        return obj.is_cancellable

    # T059 fix: nullable field — use {"type": "string", "nullable": True}
    # or a serializer field with allow_null=True
    @extend_schema_field({"type": "string", "nullable": True})
    def get_coupon_code(self, obj: object) -> str | None:
        return obj.coupon_code if hasattr(obj, "coupon_code") else None

    class Meta:
        from apps.orders.models import Order
        model = Order
        fields = [
            "id", "reference", "status", "status_display",
            "total", "item_count", "is_cancellable", "coupon_code",
            "created_at", "updated_at",
        ]
        read_only_fields = fields


# T032 fix: DELETE endpoint — always use 204, never 200
# Applied via @extend_schema_view:
@extend_schema_view(
    destroy=extend_schema(
        summary="Delete resource",
        tags=["Resources"],
        responses={
            204: None,           # No Content — the correct status for DELETE
            401: RESPONSE_401,
            403: RESPONSE_403,
            404: RESPONSE_404,
        },
    ),
)
class ResourceViewSet(viewsets.ModelViewSet):
    ...
```

---

## 11. inline_serializer — For One-Off Shapes

```python
# Use when the response shape doesn't warrant a full serializer class.
#
# NAMING RULE (T030 fix): inline_serializer names must be globally unique.
# Convention: {ViewName}{ActionName}{Request|Response}
# ❌ WRONG: inline_serializer(name="Response", ...)  ← name collision risk
# ✅ RIGHT: inline_serializer(name="TokenRefreshResponse", ...)

from drf_spectacular.utils import extend_schema, inline_serializer
from rest_framework import serializers


class TokenRefreshView(APIView):
    @extend_schema(
        summary="Refresh JWT access token",
        tags=["Auth"],
        request=inline_serializer(
            name="TokenRefreshRequest",   # ← {ViewName}{Direction}
            fields={"refresh": serializers.CharField(help_text="Valid refresh token")},
        ),
        responses={
            200: inline_serializer(
                name="TokenRefreshResponse",  # ← globally unique
                fields={"access": serializers.CharField(help_text="New access token")},
            ),
            401: RESPONSE_401,
        },
    )
    def post(self, request: Request) -> Response: ...


# ── APIView annotation (T036 fix) ─────────────────────────────────────
# @extend_schema_view is for ViewSets only.
# On plain APIView, annotate each HTTP method directly.

class UserProfileView(APIView):
    @extend_schema(
        summary="Get user profile",
        tags=["Users"],
        responses={200: UserProfileSerializer, 401: RESPONSE_401, 404: RESPONSE_404},
    )
    def get(self, request: Request) -> Response: ...

    @extend_schema(
        summary="Update user profile",
        tags=["Users"],
        request=UserProfileUpdateSerializer,
        responses={200: UserProfileSerializer, 400: RESPONSE_400, 401: RESPONSE_401},
    )
    def patch(self, request: Request) -> Response: ...


# ── FBV annotation (T036 fix) ─────────────────────────────────────────
# @extend_schema goes ABOVE @api_view, not below

@extend_schema(
    summary="List active items",
    tags=["Items"],
    responses={200: ItemSerializer(many=True), 401: RESPONSE_401},
)
@api_view(["GET"])
def list_items(request: Request) -> Response: ...


# ── Binary file response (T033 fix) ───────────────────────────────────
from drf_spectacular.types import OpenApiTypes

@extend_schema(
    summary="Download invoice PDF",
    tags=["Orders"],
    responses={
        (200, "application/pdf"): OpenApiResponse(
            description="PDF file download",
            response=OpenApiTypes.BINARY,
        ),
        401: RESPONSE_401,
        404: RESPONSE_404,
    },
)
@action(detail=True, methods=["get"], url_path="invoice")
def download_invoice(self, request: Request, pk: str | None = None) -> Response: ...
```

---

## 12. Exclude Internal / Non-API Endpoints

```python
# DECISION 8: Health checks, readiness probes, and internal views
# pollute the API schema. Exclude them explicitly.
from drf_spectacular.utils import extend_schema
from django.views.decorators.http import require_GET


@extend_schema(exclude=True)  # Does NOT appear in the OpenAPI schema
@require_GET
def health_check(request):
    ...


@extend_schema(exclude=True)
@require_GET
def readiness_check(request):
    ...
```

---

## 13. Versioned API — Prevent operation_id Collisions

```python
# DECISION 9: /api/v1/orders/ and /api/v2/orders/ auto-generate the
# same operation_id "orders_list". This silently breaks client generators.

# config/settings/base.py
# SCHEMA_PATH_PREFIX strips the version for tag extraction
# but operation_id still collides. Fix per endpoint:

@extend_schema_view(
    list=extend_schema(operation_id="v2_orders_list"),
    retrieve=extend_schema(operation_id="v2_orders_retrieve"),
)
class OrderViewSetV2(OrderViewSet):
    """Version 2 — extended order fields."""
    ...

# Or use SPECTACULAR_SETTINGS "SCHEMA_PATH_PREFIX" to generate a single
# unified schema that documents both versions via `versions=` on decorators:
@extend_schema(versions=["v1"], responses={200: OrderListSerializer})
@extend_schema(versions=["v2"], responses={200: OrderListSerializerV2})
def list(self, request, *args, **kwargs): ...
```

---

## 14. CI/CD — Schema Validation Step

```yaml
# .github/workflows/ci.yml — add this job step
# DECISION 4: Validate the schema on every PR. Catches missing annotations,
# broken serializer references, and spec violations before they reach clients.
# T020 fix: --fail-on-warn does not exist in the CLI — removed.
# Validation is via --validate (exits non-zero on errors) + Python check.

- name: Validate OpenAPI schema
  env:
    DJANGO_SETTINGS_MODULE: config.settings.testing
    DJANGO_SECRET_KEY: ci-secret-key
    JWT_SIGNING_KEY: ci-jwt-key
    DATABASE_URL: postgres://test_user:test_pass@localhost:5432/test_db
  run: |
    # Generate and validate the schema
    python manage.py spectacular \
      --color \
      --file /tmp/schema.yml \
      --validate

    # Additional checks: schema is non-empty and has documented paths
    python - << 'EOF'
    import yaml, sys
    with open("/tmp/schema.yml") as f:
        schema = yaml.safe_load(f)
    paths = schema.get("paths", {})
    if not paths:
        print("ERROR: Schema has no paths — check DEFAULT_SCHEMA_CLASS setting")
        sys.exit(1)
    # Check that security schemes are present (JWT extension registered)
    security_schemes = schema.get("components", {}).get("securitySchemes", {})
    if not security_schemes:
        print("WARNING: No security schemes found — JWT extension may not be registered")
    print(f"✅ Schema valid: {len(paths)} paths, {len(security_schemes)} security schemes")
    EOF
```

```python
# Also validate in tests — catches errors in the test suite
# apps/core/tests/test_schema.py
import pytest
from rest_framework.test import APIClient


@pytest.mark.django_db
def test_schema_generates_without_errors() -> None:
    """The schema endpoint must return 200 with valid OpenAPI content."""
    from django.test import override_settings
    from django.conf import settings as django_settings

    with override_settings(
        SPECTACULAR_SETTINGS={
            **django_settings.SPECTACULAR_SETTINGS,
            "SERVE_PUBLIC": True,
            "SERVE_PERMISSIONS": ["rest_framework.permissions.AllowAny"],
        }
    ):
        client = APIClient()
        response = client.get("/api/schema/?format=json")
        assert response.status_code == 200, f"Schema endpoint returned {response.status_code}"
        schema = response.json()
        assert "openapi" in schema, "Response is not a valid OpenAPI schema"
        assert "paths" in schema, "Schema has no paths section"
        assert len(schema["paths"]) > 0, "Schema has no documented paths"
        assert "components" in schema, "Schema has no components section"


@pytest.mark.django_db
def test_schema_has_security_schemes() -> None:
    """JWT security scheme must be registered via the auth extension."""
    from django.test import override_settings
    from django.conf import settings as django_settings

    with override_settings(
        SPECTACULAR_SETTINGS={
            **django_settings.SPECTACULAR_SETTINGS,
            "SERVE_PUBLIC": True,
            "SERVE_PERMISSIONS": ["rest_framework.permissions.AllowAny"],
        }
    ):
        client = APIClient()
        response = client.get("/api/schema/?format=json")
        schema = response.json()
        security_schemes = schema.get("components", {}).get("securitySchemes", {})
        assert "jwtAuth" in security_schemes, (
            "jwtAuth security scheme not found. "
            "Check that SimpleJWTAuthenticationExtension is imported in AppConfig.ready()"
        )
```

---

## 15. Polymorphic Responses

```python
# For endpoints that return different shapes based on type
from drf_spectacular.utils import extend_schema, PolymorphicProxySerializer


@extend_schema(
    request=PolymorphicProxySerializer(
        component_name="CreateNotification",
        serializers=[
            EmailNotificationSerializer,
            PushNotificationSerializer,
            SMSNotificationSerializer,
        ],
        resource_type_field_name="channel",  # The discriminator field
    ),
    responses={
        201: PolymorphicProxySerializer(
            component_name="NotificationCreated",
            serializers=[
                EmailNotificationSerializer,
                PushNotificationSerializer,
                SMSNotificationSerializer,
            ],
            resource_type_field_name="channel",
        )
    },
)
def create(self, request: Request) -> Response: ...
```

---

## 16. Schema Response Caching (Large Projects)

```python
# In projects with 50+ apps and hundreds of endpoints, schema generation
# can take 2–5 seconds per request — unacceptable for production doc loads.
# T060 fix: Cache the schema view response.

# config/urls.py
from django.views.decorators.cache import cache_page
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView, SpectacularRedocView
from django.conf import settings

# Cache duration: 5 minutes. Invalidated automatically on deploy
# (new process = empty cache). Use a shorter TTL during active development.
SCHEMA_CACHE_TTL = 0 if settings.DEBUG else 60 * 5

urlpatterns = [
    path(
        "api/schema/",
        cache_page(SCHEMA_CACHE_TTL)(SpectacularAPIView.as_view()),
        name="schema",
    ),
    path(
        "api/schema/swagger-ui/",
        SpectacularSwaggerView.as_view(url_name="schema"),
        name="swagger-ui",
    ),
    path(
        "api/schema/redoc/",
        SpectacularRedocView.as_view(url_name="schema"),
        name="redoc",
    ),
]
# Note: Cache only the raw schema (SpectacularAPIView), not the UI pages.
# Swagger UI and Redoc load the schema via XHR — they always get fresh UI.
```

---

## 17. Project-Wide Checklist — Applied Per App

```
When creating a new Django app with public API endpoints:
[ ] Add tag entry to SPECTACULAR_SETTINGS["TAGS"] (name + description)
[ ] Apply @extend_schema_view on every ViewSet for all standard CRUD actions
[ ] Apply @extend_schema on every @action method
[ ] Use @extend_schema per HTTP method on APIView classes
[ ] Place @extend_schema above @api_view on function-based views
[ ] Annotate ALL SerializerMethodField with @extend_schema_field
[ ] Annotate nullable SMF with {"type": "...", "nullable": True}
[ ] Import and attach STANDARD_ERRORS to response dicts
[ ] Use {204: None} on destroy endpoints — never 200 for DELETE
[ ] Use inline_serializer names: {ViewName}{Action}{Request|Response} (unique)
[ ] Use responses={(200, "application/pdf"): ...} for binary downloads
[ ] Exclude health/internal views with @extend_schema(exclude=True)
[ ] Verify the schema generates cleanly: manage.py spectacular --validate
[ ] Add schema validation + Python checks to CI pipeline
[ ] Confirm Swagger UI "Authorize" button appears (auth extension registered)
[ ] Confirm custom pagination wrapper is present on list responses
[ ] No operation_id collisions (--validate catches them)
[ ] persistAuthorization = False in production Swagger UI settings
[ ] Schema URLs always registered, gated by SERVE_PERMISSIONS only
[ ] Schema response caching enabled for projects with 30+ endpoints
```

---

## 18. Context7 Quick-Reference for This Skill

```
Library ID:  /tfranzel/drf-spectacular
Key queries to run before implementing:
  - "extend_schema decorator parameters"
  - "SPECTACULAR_SETTINGS configuration defaults"
  - "OpenApiAuthenticationExtension JWT security scheme"
  - "inline_serializer usage"
  - "postprocessing_hooks schema"
  - "PolymorphicProxySerializer"
  - "extend_schema_field SerializerMethodField"
  - "management command spectacular validate"
```
