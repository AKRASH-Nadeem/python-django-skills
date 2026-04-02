---
name: django-security
description: >
  Apply this skill for all security-related work: OWASP hardening, security headers,
  HTTPS enforcement, CORS configuration, input validation, rate limiting, SQL injection
  prevention, XSS/CSRF protection, secrets management, authentication hardening,
  and Django 6 security settings. Use whenever implementing auth flows, handling
  user input, configuring production settings, or auditing existing code for vulnerabilities.
---

# Django Security Skill

> FIRST: Use Context7 `/websites/djangoproject_en_6_0` for latest security settings.
> Security is non-negotiable. When in doubt, deny.

---

## 1. Production Security Settings

```python
# config/settings/production.py
from config.settings.base import *

# ── HTTPS ─────────────────────────────────────────────────────────
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")

# ── HSTS ──────────────────────────────────────────────────────────
SECURE_HSTS_SECONDS = 31_536_000          # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True                # Submit to browser preload list

# ── Content Security ──────────────────────────────────────────────
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_REFERRER_POLICY = "strict-origin-when-cross-origin"
SECURE_CROSS_ORIGIN_OPENER_POLICY = "same-origin"

# Django 6 CSP (native — no django-csp package needed)
from django.utils.csp import CSP
SECURE_CSP = {
    "default-src": [CSP.SELF],
    "script-src": [CSP.SELF, CSP.NONCE],  # Nonces for inline scripts
    "style-src": [CSP.SELF, CSP.NONCE],
    "img-src": [CSP.SELF, "https:", "data:"],
    "connect-src": [CSP.SELF, "wss:"],
    "frame-ancestors": [CSP.NONE],
    "base-uri": [CSP.SELF],
    "form-action": [CSP.SELF],
}

# ── Cookies ───────────────────────────────────────────────────────
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = "Lax"
SESSION_COOKIE_AGE = 60 * 60 * 24 * 14   # 2 weeks
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_HTTPONLY = True
CSRF_COOKIE_SAMESITE = "Strict"

# ── Permissions Policy ────────────────────────────────────────────
PERMISSIONS_POLICY = {
    "camera": [],
    "microphone": [],
    "geolocation": [],
    "payment": [],
}

# ── Debug & Internals ─────────────────────────────────────────────
DEBUG = False
ALLOWED_HOSTS = env.list("DJANGO_ALLOWED_HOSTS")
# Never expose sensitive info in 500 errors
SERVER_EMAIL = env("SERVER_EMAIL")
ADMINS = [("Ops Team", env("OPS_EMAIL"))]

# ── Password Validation ───────────────────────────────────────────
AUTH_PASSWORD_VALIDATORS = [
    {"NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator"},
    {"NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
     "OPTIONS": {"min_length": 12}},
    {"NAME": "django.contrib.auth.password_validation.CommonPasswordValidator"},
    {"NAME": "django.contrib.auth.password_validation.NumericPasswordValidator"},
]

# ── Upload Limits ─────────────────────────────────────────────────
DATA_UPLOAD_MAX_MEMORY_SIZE = 10 * 1024 * 1024   # 10MB
DATA_UPLOAD_MAX_NUMBER_FIELDS = 100
FILE_UPLOAD_MAX_MEMORY_SIZE = 10 * 1024 * 1024   # 10MB
```

---

## 2. CORS Configuration

```python
# config/settings/base.py
# django-cors-headers required
CORS_ALLOWED_ORIGINS = env.list("CORS_ALLOWED_ORIGINS", default=[])
CORS_ALLOW_CREDENTIALS = True
CORS_ALLOWED_METHODS = ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"]
CORS_ALLOWED_HEADERS = [
    "accept",
    "authorization",
    "content-type",
    "x-csrftoken",
    "x-request-id",
]
CORS_EXPOSE_HEADERS = ["x-request-id"]
CORS_PREFLIGHT_MAX_AGE = 86400  # 1 day

# NEVER use: CORS_ALLOW_ALL_ORIGINS = True in production
```

---

## 3. Input Validation — Defense in Depth

```python
# apps/core/validators.py
from __future__ import annotations
import re
import uuid
from django.core.exceptions import ValidationError
from django.core.validators import URLValidator


def validate_uuid(value: str) -> None:
    """Validate a string is a valid UUID."""
    try:
        uuid.UUID(str(value))
    except (ValueError, AttributeError):
        raise ValidationError(f"'{value}' is not a valid UUID.")


def validate_safe_string(value: str, max_length: int = 500) -> None:
    """Reject strings with common injection patterns."""
    dangerous_patterns = [
        r"<script",
        r"javascript:",
        r"vbscript:",
        r"onload=",
        r"onerror=",
        r"--",            # SQL comment
        r"union\s+select", # SQL UNION
        r"drop\s+table",
        r"\x00",          # Null byte
    ]
    lower = value.lower()
    for pattern in dangerous_patterns:
        if re.search(pattern, lower):
            raise ValidationError("Value contains disallowed content.")
    if len(value) > max_length:
        raise ValidationError(f"Value exceeds maximum length of {max_length}.")


def validate_url_safe(value: str) -> None:
    """Validate URL is safe for redirect (prevent open redirect)."""
    from urllib.parse import urlparse
    parsed = urlparse(value)
    # Reject absolute URLs (must be relative)
    if parsed.scheme or parsed.netloc:
        raise ValidationError("Only relative URLs are allowed.")
    # Reject protocol-relative URLs
    if value.startswith("//"):
        raise ValidationError("Protocol-relative URLs are not allowed.")
```

---

## 4. SQL Injection Prevention

```python
# ✅ CORRECT — Always use ORM or parameterized queries
users = User.objects.filter(email=email)

# ✅ CORRECT — When raw SQL is necessary, use params
from django.db import connection
with connection.cursor() as cursor:
    cursor.execute(
        "SELECT id, email FROM users_user WHERE status = %s AND created_at > %s",
        [status, cutoff_date]  # NEVER format these into the string
    )

# ✅ CORRECT — RawSQL annotation
from django.db.models.expressions import RawSQL
queryset = MyModel.objects.annotate(
    custom_val=RawSQL("EXTRACT(YEAR FROM created_at)", [])
)

# ❌ NEVER DO THIS
cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")
cursor.execute("SELECT * FROM users WHERE email = '" + email + "'")
User.objects.extra(where=[f"email = '{email}'"])  # extra() is deprecated AND dangerous
```

---

## 5. Authentication Security

```python
# apps/users/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.throttling import AnonRateThrottle
from django.contrib.auth import authenticate


class LoginThrottle(AnonRateThrottle):
    scope = "login"  # settings: "login": "5/minute"


class LoginView(APIView):
    """
    Security rules for login endpoints:
    1. Throttle aggressively
    2. Generic error messages (don't reveal if email exists)
    3. Log failed attempts with IP
    4. Use constant-time comparison (authenticate() does this)
    5. Issue short-lived tokens
    """
    throttle_classes = [LoginThrottle]
    permission_classes = []

    def post(self, request) -> Response:
        email = request.data.get("email", "")
        password = request.data.get("password", "")

        if not email or not password:
            return Response(
                {"error": {"code": "invalid_credentials", "message": "Invalid credentials."}},
                status=status.HTTP_401_UNAUTHORIZED,
            )

        import logging
        logger = logging.getLogger(__name__)
        ip = self._get_client_ip(request)

        user = authenticate(request, username=email, password=password)
        if user is None:
            # Generic message — don't reveal if email exists
            logger.warning("Failed login attempt", extra={"email": email, "ip": ip})
            return Response(
                {"error": {"code": "invalid_credentials", "message": "Invalid credentials."}},
                status=status.HTTP_401_UNAUTHORIZED,
            )

        if not user.is_active:
            return Response(
                {"error": {"code": "account_disabled", "message": "Account is disabled."}},
                status=status.HTTP_401_UNAUTHORIZED,
            )

        from rest_framework_simplejwt.tokens import RefreshToken
        refresh = RefreshToken.for_user(user)
        logger.info("Successful login", extra={"user_id": str(user.pk), "ip": ip})

        return Response({
            "access": str(refresh.access_token),
            "refresh": str(refresh),
        })

    def _get_client_ip(self, request) -> str:
        x_forwarded_for = request.META.get("HTTP_X_FORWARDED_FOR")
        if x_forwarded_for:
            return x_forwarded_for.split(",")[0].strip()
        return request.META.get("REMOTE_ADDR", "unknown")
```

---

## 6. File Upload Security

```python
# apps/core/validators.py
import magic  # python-magic
from django.core.exceptions import ValidationError

ALLOWED_MIME_TYPES = {
    "image": ["image/jpeg", "image/png", "image/gif", "image/webp"],
    "document": ["application/pdf", "text/plain"],
    "spreadsheet": ["application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"],
}


def validate_file_type(file, allowed_category: str) -> None:
    """
    Validate file type by MAGIC BYTES, not extension.
    Extensions can be faked. Magic bytes cannot (easily).
    """
    allowed = ALLOWED_MIME_TYPES.get(allowed_category, [])
    if not allowed:
        raise ValidationError(f"Unknown file category: {allowed_category}")

    file.seek(0)
    mime = magic.from_buffer(file.read(1024), mime=True)
    file.seek(0)

    if mime not in allowed:
        raise ValidationError(
            f"Invalid file type '{mime}'. Allowed: {', '.join(allowed)}"
        )


def validate_file_size(file, max_mb: int = 10) -> None:
    """Validate file does not exceed size limit."""
    max_bytes = max_mb * 1024 * 1024
    if file.size > max_bytes:
        raise ValidationError(f"File size exceeds {max_mb}MB limit.")


def sanitize_filename(filename: str) -> str:
    """Return a safe filename — strip path traversal attempts."""
    import os
    import re
    # Remove path components
    filename = os.path.basename(filename)
    # Allow only safe characters
    filename = re.sub(r"[^\w\.\-]", "_", filename)
    # Limit length
    name, ext = os.path.splitext(filename)
    return f"{name[:100]}{ext[:10]}"
```

---

## 7. Secrets Management

```python
# .env.example — commit this, NOT .env
# DJANGO_SECRET_KEY=your-secret-key-here
# DATABASE_URL=postgres://user:pass@localhost:5432/dbname
# REDIS_URL=redis://localhost:6379/0
# JWT_SIGNING_KEY=separate-key-for-jwt
# STRIPE_SECRET_KEY=sk_live_...
# STRIPE_WEBHOOK_SECRET=whsec_...

# config/settings/base.py
import environ
env = environ.Env()

# Secrets — from environment ONLY, no defaults for sensitive values
SECRET_KEY = env("DJANGO_SECRET_KEY")              # No default
JWT_SIGNING_KEY = env("JWT_SIGNING_KEY")           # No default
STRIPE_SECRET_KEY = env("STRIPE_SECRET_KEY")       # No default
STRIPE_WEBHOOK_SECRET = env("STRIPE_WEBHOOK_SECRET")  # No default

# Non-sensitive with safe defaults
DEBUG = env.bool("DJANGO_DEBUG", default=False)
LOG_LEVEL = env("LOG_LEVEL", default="INFO")

# NEVER:
# SECRET_KEY = "hardcoded-secret"
# SECRET_KEY = env("DJANGO_SECRET_KEY", default="dev-key-not-for-production")
```

---

## 8. Permission System Architecture

```python
# apps/core/permissions.py
from rest_framework.permissions import BasePermission
from rest_framework.request import Request


class HasAPIKey(BasePermission):
    """Allow service-to-service access via API key."""
    message = "Valid API key required."

    def has_permission(self, request: Request, view: object) -> bool:
        from apps.integrations.models import APIKey
        api_key = request.META.get("HTTP_X_API_KEY", "")
        if not api_key:
            return False
        return APIKey.objects.filter(key=api_key, is_active=True).exists()


class IsResourceOwnerOrAdmin(BasePermission):
    """Allow if user owns the resource OR is admin."""
    def has_object_permission(self, request, view, obj) -> bool:
        if request.user.is_staff:
            return True
        owner_field = getattr(view, "owner_field", "user")
        owner = getattr(obj, owner_field, None)
        if owner is None:
            return False
        if hasattr(owner, "pk"):
            return owner.pk == request.user.pk
        return owner == request.user.pk


class IsSuperAdmin(BasePermission):
    """Superuser-only access."""
    def has_permission(self, request, view) -> bool:
        return bool(request.user and request.user.is_superuser)
```

---

## 9. Security Middleware

```python
# apps/core/middleware.py
from __future__ import annotations
import logging
import time
import uuid
from typing import Callable
from django.http import HttpRequest, HttpResponse

logger = logging.getLogger(__name__)


class RequestIDMiddleware:
    """Attach unique request ID to every request for tracing."""

    def __init__(self, get_response: Callable) -> None:
        self.get_response = get_response

    def __call__(self, request: HttpRequest) -> HttpResponse:
        request_id = request.META.get("HTTP_X_REQUEST_ID") or str(uuid.uuid4())
        request.request_id = request_id
        response = self.get_response(request)
        response["X-Request-ID"] = request_id
        return response


class SecurityHeadersMiddleware:
    """Add additional security headers not covered by Django's SecurityMiddleware."""

    def __init__(self, get_response: Callable) -> None:
        self.get_response = get_response

    def __call__(self, request: HttpRequest) -> HttpResponse:
        response = self.get_response(request)
        response["X-Content-Type-Options"] = "nosniff"
        response["X-Frame-Options"] = "DENY"
        response["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()"
        return response


class RequestLoggingMiddleware:
    """Log all requests with timing. Use in production for observability."""

    def __init__(self, get_response: Callable) -> None:
        self.get_response = get_response

    def __call__(self, request: HttpRequest) -> HttpResponse:
        start = time.perf_counter()
        response = self.get_response(request)
        duration_ms = (time.perf_counter() - start) * 1000

        logger.info(
            "HTTP %s %s %d",
            request.method,
            request.path,
            response.status_code,
            extra={
                "method": request.method,
                "path": request.path,
                "status_code": response.status_code,
                "duration_ms": round(duration_ms, 2),
                "user_id": str(getattr(request.user, "pk", "anonymous")),
                "request_id": getattr(request, "request_id", ""),
            },
        )
        return response
```

---

## 10. Security Audit Checklist

```
Authentication & Authorization:
✅ Custom user model with email login
✅ JWT short-lived (15min access, 7day refresh with rotation)
✅ Password validators enforce 12+ char minimum
✅ Login endpoint rate-limited (5/minute)
✅ Generic error messages on auth failure
✅ Object-level permissions on all detail views
✅ Staff/superuser endpoints use IsSuperAdmin permission

Transport & Headers:
✅ HTTPS enforced (SECURE_SSL_REDIRECT)
✅ HSTS with 1-year max-age and preload
✅ CSP configured with nonces
✅ X-Frame-Options: DENY
✅ X-Content-Type-Options: nosniff
✅ Referrer-Policy: strict-origin-when-cross-origin
✅ CORS: explicit allowed origins list (no wildcard in prod)

Data:
✅ All secrets in environment variables (no defaults for sensitive values)
✅ No sensitive data in logs (no passwords, tokens, full card numbers)
✅ File uploads: magic-byte type checking + size limits
✅ Input sanitization: DRF serializers validate all user input
✅ SQL: ORM or parameterized queries only
✅ UUIDs as public IDs (no sequential integer PKs exposed)

Cookies & Sessions:
✅ SESSION_COOKIE_SECURE = True
✅ SESSION_COOKIE_HTTPONLY = True
✅ CSRF_COOKIE_SECURE = True
✅ Cookies have SameSite attribute

Dependency & Infrastructure:
✅ pip-audit in CI pipeline
✅ DEBUG = False in production
✅ ALLOWED_HOSTS explicitly set
✅ Admin URL is NOT /admin/ (rename in production)
✅ Error pages don't reveal stack traces
```
