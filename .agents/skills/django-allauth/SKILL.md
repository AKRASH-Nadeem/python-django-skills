---
name: django-allauth
description: >
  Apply this skill for all authentication and account management work using
  django-allauth: email/password auth, social OAuth2 (Google, GitHub, Apple, etc.),
  MFA/TOTP, headless REST API mode for SPAs/mobile, custom adapters, email
  verification, password policies, allauth signals, integration with custom User
  model, and SimpleJWT token bridge. Use when building login flows, social
  connect, sign-up, email verification, MFA, or any auth-related feature.
---

# Django Allauth Skill

> FIRST: Use Context7 with library ID `/pennersr/django-allauth` for the latest
> configuration API. Allauth has evolved significantly — settings names, headless
> API endpoints, and MFA features change between minor versions.
> ALWAYS verify setting names before using them.

---

## 1. When to Use Allauth

```
Use allauth when you need:
✅ Email/password registration with email verification
✅ Social login (Google, GitHub, Apple, Facebook, etc.)
✅ MFA/2FA with TOTP, WebAuthn/Passkeys, recovery codes
✅ Headless REST API mode (for React/Vue SPAs or mobile)
✅ Account management (email change, password change, delete)
✅ Username-less auth (email-only, our standard)

Do NOT use allauth alone for:
❌ JWT token issuance → bridge with SimpleJWT (see section 8)
❌ API-key auth → build custom with HasAPIKey permission
❌ Service-to-service auth → use API keys or mTLS
```

---

## 2. Installation

```
# requirements/base.txt
django-allauth[mfa]       # Includes MFA support
# For headless REST API:
django-allauth[headless]  # Includes REST API views
```

---

## 3. Settings — Complete Configuration

```python
# config/settings/base.py

INSTALLED_APPS = [
    # Django built-ins
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.sites",                     # Required by allauth

    # Allauth apps
    "allauth",
    "allauth.account",
    "allauth.socialaccount",
    "allauth.socialaccount.providers.google",
    "allauth.socialaccount.providers.github",
    # Add other providers as needed:
    # "allauth.socialaccount.providers.apple",
    # "allauth.socialaccount.providers.microsoft",
    "allauth.mfa",                              # MFA/2FA support
    "allauth.headless",                         # Headless REST API (for SPAs)

    # Local apps
    "apps.users",
]

SITE_ID = 1  # Required by allauth

# ── Middleware ────────────────────────────────────────────────────
MIDDLEWARE = [
    ...
    "allauth.account.middleware.AccountMiddleware",  # Required in allauth
    ...
]

# ── Authentication Backends ───────────────────────────────────────
AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",    # Admin login
    "allauth.account.auth_backends.AuthenticationBackend",
]

# ── Account Settings ──────────────────────────────────────────────
ACCOUNT_LOGIN_METHODS = {"email"}                   # Email-only, no username
ACCOUNT_EMAIL_REQUIRED = True
ACCOUNT_EMAIL_VERIFICATION = "mandatory"            # Must verify email before login
ACCOUNT_SIGNUP_FIELDS = ["email*", "password1*", "password2*"]
ACCOUNT_USER_MODEL_USERNAME_FIELD = None            # Custom user model: no username
ACCOUNT_UNIQUE_EMAIL = True
ACCOUNT_PASSWORD_MIN_LENGTH = 12                    # Matches AUTH_PASSWORD_VALIDATORS
ACCOUNT_PREVENT_ENUMERATION = True                  # Don't reveal if email exists
ACCOUNT_SESSION_REMEMBER = None                     # Ask user; True = always remember
ACCOUNT_LOGIN_ON_EMAIL_CONFIRMATION = True          # Auto-login after verification
ACCOUNT_CONFIRM_EMAIL_ON_GET = True                 # Confirm email via GET (simple UX)
ACCOUNT_EMAIL_CONFIRMATION_EXPIRE_DAYS = 3          # 3-day verification window

# ── Adapters ─────────────────────────────────────────────────────
ACCOUNT_ADAPTER = "apps.users.adapters.AccountAdapter"
SOCIALACCOUNT_ADAPTER = "apps.users.adapters.SocialAccountAdapter"

# ── Social Account ────────────────────────────────────────────────
SOCIALACCOUNT_AUTO_SIGNUP = True                    # Auto-create account on social login
SOCIALACCOUNT_EMAIL_REQUIRED = True                 # Always require verified email
SOCIALACCOUNT_EMAIL_VERIFICATION = "none"           # Provider already verified it
SOCIALACCOUNT_LOGIN_ON_GET = False                  # Require POST to initiate OAuth
SOCIALACCOUNT_STORE_TOKENS = True                   # Store OAuth tokens for API calls
SOCIALACCOUNT_QUERY_EMAIL = True                    # Always fetch email from provider

# ── Social Providers ──────────────────────────────────────────────
# IMPORTANT: Never hardcode client_id/secret — use environment variables
SOCIALACCOUNT_PROVIDERS = {
    "google": {
        "SCOPE": ["profile", "email"],
        "AUTH_PARAMS": {"access_type": "online"},
        "OAUTH_PKCE_ENABLED": True,                 # PKCE for added security
        "APPS": [
            {
                "client_id": env("GOOGLE_CLIENT_ID"),
                "secret": env("GOOGLE_CLIENT_SECRET"),
                "key": "",
            }
        ],
    },
    "github": {
        "SCOPE": ["user:email", "read:user"],
        "APPS": [
            {
                "client_id": env("GITHUB_CLIENT_ID"),
                "secret": env("GITHUB_CLIENT_SECRET"),
            }
        ],
    },
}

# ── MFA Settings ──────────────────────────────────────────────────
MFA_SUPPORTED_TYPES = ["totp", "recovery_codes", "webauthn"]
MFA_TOTP_ISSUER = env("MFA_TOTP_ISSUER", default="MyApp")
MFA_TOTP_PERIOD = 30
MFA_TOTP_DIGITS = 6
MFA_RECOVERY_CODE_COUNT = 10

# ── Headless API (for SPA/Mobile clients) ─────────────────────────
HEADLESS_ONLY = False                              # False: keep traditional views too
HEADLESS_FRONTEND_URLS = {
    "account_confirm_email": "/auth/verify-email/{key}",
    "account_reset_password_from_key": "/auth/reset-password/{uid}/{token}",
    "socialaccount_login_error": "/auth/social-error",
}

# ── URL Configuration ─────────────────────────────────────────────
# config/urls.py
urlpatterns = [
    path("accounts/", include("allauth.urls")),           # Traditional views
    path("_allauth/", include("allauth.headless.urls")),  # Headless REST API
]
```

---

## 4. Custom Adapters — The Right Extension Point

```python
# apps/users/adapters.py
"""
Adapters are the correct place to:
- Customize user creation during signup/social login
- Add validation to signup (e.g., block certain domains)
- Set custom fields on new accounts
- Control what social data is synced

DO NOT override views — override adapters.
"""
from __future__ import annotations
import logging
from typing import Any
from allauth.account.adapter import DefaultAccountAdapter
from allauth.socialaccount.adapter import DefaultSocialAccountAdapter
from django.contrib.auth import get_user_model
from django.http import HttpRequest

logger = logging.getLogger(__name__)
User = get_user_model()


class AccountAdapter(DefaultAccountAdapter):
    """
    Custom account adapter.
    Override methods here to customize allauth behavior.
    """

    def is_open_for_signup(self, request: HttpRequest) -> bool:
        """Control whether new signups are allowed."""
        from django.conf import settings
        return getattr(settings, "ACCOUNT_ALLOW_REGISTRATION", True)

    def save_user(
        self, request: HttpRequest, user: Any, form: Any, commit: bool = True
    ) -> Any:
        """Called when a user registers. Add custom field population here."""
        user = super().save_user(request, user, form, commit=False)
        # Example: set a default timezone from the form
        if hasattr(form, "cleaned_data"):
            user.timezone = form.cleaned_data.get("timezone", "UTC")
        if commit:
            user.save()
        return user

    def get_email_confirmation_url(self, request: HttpRequest, emailconfirmation: Any) -> str:
        """Return frontend URL for email confirmation (headless mode)."""
        from django.conf import settings
        key = emailconfirmation.key
        frontend_url = getattr(settings, "FRONTEND_URL", "")
        return f"{frontend_url}/auth/verify-email/{key}"

    def send_mail(self, template_prefix: str, email: str, context: dict[str, Any]) -> None:
        """
        Override to use your own email service instead of Django's default.
        Route through Celery for async delivery.
        """
        from apps.emails.tasks import send_allauth_email
        from django.db import transaction
        transaction.on_commit(
            lambda: send_allauth_email.delay(template_prefix, email, context)
        )


class SocialAccountAdapter(DefaultSocialAccountAdapter):
    """Custom social account adapter."""

    def pre_social_login(self, request: HttpRequest, sociallogin: Any) -> None:
        """
        Called after OAuth but before account creation/login.
        Use to:
        - Connect social account to existing account by email
        - Block signups from certain providers
        - Validate required scopes were granted
        """
        if sociallogin.is_existing:
            return  # Already connected — nothing to do

        # Auto-connect to existing account by verified email
        email = sociallogin.user.email
        if email:
            try:
                existing_user = User.objects.get(email=email, is_active=True)
                sociallogin.connect(request, existing_user)
                logger.info(
                    "Social account connected to existing user",
                    extra={
                        "user_id": str(existing_user.pk),
                        "provider": sociallogin.account.provider,
                    },
                )
            except User.DoesNotExist:
                pass  # New user — allauth will create the account

    def is_open_for_signup(self, request: HttpRequest, sociallogin: Any) -> bool:
        """Control whether social signup is allowed."""
        from django.conf import settings
        return getattr(settings, "ACCOUNT_ALLOW_REGISTRATION", True)

    def populate_user(
        self, request: HttpRequest, sociallogin: Any, data: dict[str, Any]
    ) -> Any:
        """
        Populate user fields from social data.
        Called when creating a new user via social signup.
        """
        user = super().populate_user(request, sociallogin, data)
        # Map provider-specific data to our custom fields
        extra_data = sociallogin.account.extra_data
        if not user.first_name and "given_name" in extra_data:
            user.first_name = extra_data["given_name"]
        if not user.last_name and "family_name" in extra_data:
            user.last_name = extra_data["family_name"]
        return user
```

---

## 5. Signals — Hook Into Auth Events

```python
# apps/users/signals.py
"""
Allauth signals — use for cross-domain side effects.
Rule: Always dispatch heavy work to Celery via transaction.on_commit().
"""
from __future__ import annotations
import logging
from django.db import transaction
from django.dispatch import receiver
from allauth.account.signals import (
    user_signed_up,
    user_logged_in,
    email_confirmed,
    password_changed,
    password_reset,
)
from allauth.socialaccount.signals import (
    social_account_added,
    social_account_removed,
)

logger = logging.getLogger(__name__)


@receiver(user_signed_up)
def on_user_signed_up(sender: object, request: object, user: object, **kwargs: object) -> None:
    """
    Fired when a user completes registration.
    Note: for social signup, also fires social_account_added.
    """
    logger.info("New user registered", extra={"user_id": str(user.pk), "email": user.email})
    transaction.on_commit(
        lambda: _dispatch_signup_tasks(str(user.pk))
    )


def _dispatch_signup_tasks(user_id: str) -> None:
    from apps.users.tasks import (
        send_welcome_email,
        create_default_preferences,
        notify_admins_of_new_signup,
    )
    send_welcome_email.delay(user_id)
    create_default_preferences.delay(user_id)


@receiver(email_confirmed)
def on_email_confirmed(sender: object, request: object, email_address: object, **kwargs: object) -> None:
    """
    Fired when a user verifies their email.
    At this point, `email_address.user` has a verified email.
    """
    logger.info(
        "Email verified",
        extra={"user_id": str(email_address.user.pk), "email": email_address.email}
    )
    transaction.on_commit(
        lambda: _on_email_verified(str(email_address.user.pk))
    )


def _on_email_verified(user_id: str) -> None:
    from apps.users.tasks import activate_pending_features
    activate_pending_features.delay(user_id)


@receiver(password_changed)
def on_password_changed(sender: object, request: object, user: object, **kwargs: object) -> None:
    """Security: notify user of password change and invalidate sessions."""
    logger.info("Password changed", extra={"user_id": str(user.pk)})
    transaction.on_commit(
        lambda: _notify_password_change(str(user.pk))
    )


def _notify_password_change(user_id: str) -> None:
    from apps.emails.tasks import send_security_alert
    send_security_alert.delay(user_id, alert_type="password_changed")


@receiver(social_account_added)
def on_social_account_connected(sender: object, request: object, sociallogin: object, **kwargs: object) -> None:
    """Fired when user explicitly connects a social account."""
    logger.info(
        "Social account connected",
        extra={
            "user_id": str(sociallogin.user.pk),
            "provider": sociallogin.account.provider,
        }
    )


@receiver(user_logged_in)
def on_user_logged_in(sender: object, request: object, user: object, **kwargs: object) -> None:
    """Track login events for security audit."""
    ip = request.META.get("HTTP_X_FORWARDED_FOR", request.META.get("REMOTE_ADDR", ""))
    logger.info("User logged in", extra={"user_id": str(user.pk), "ip": ip})
```

---

## 6. Headless REST API — Endpoints Reference

```
# Allauth Headless API endpoints (auto-provided when allauth.headless installed)
# Base: /_allauth/{client}/v1/
# client = "browser" (cookie auth) or "app" (token auth)

Authentication:
  POST /_allauth/app/v1/auth/login              → Login (email+password)
  POST /_allauth/app/v1/auth/logout             → Logout
  POST /_allauth/app/v1/auth/signup             → Register
  GET  /_allauth/app/v1/auth/session            → Get current session/user

Email:
  POST /_allauth/app/v1/auth/email/verify       → Submit verification code
  POST /_allauth/app/v1/auth/email/resend       → Resend verification

Password:
  POST /_allauth/app/v1/auth/password/change    → Change password (authenticated)
  POST /_allauth/app/v1/auth/password/request   → Request reset (unauthenticated)
  POST /_allauth/app/v1/auth/password/reset     → Submit new password + token

MFA:
  GET  /_allauth/app/v1/account/authenticators  → List MFA methods
  POST /_allauth/app/v1/auth/2fa/authenticate   → Complete MFA challenge
  GET/POST /_allauth/app/v1/account/authenticators/totp → Setup TOTP

Social:
  GET  /_allauth/app/v1/socialaccount/providers → List configured providers
  POST /_allauth/app/v1/auth/provider/token     → Authenticate with provider token
  POST /_allauth/app/v1/auth/provider/signup    → Complete social signup
```

---

## 7. Custom Headless User Serializer

```python
# apps/users/adapters.py (add to existing file)
from allauth.headless.adapter import DefaultHeadlessAdapter
from dataclasses import dataclass


class HeadlessAdapter(DefaultHeadlessAdapter):
    """
    Customize the user payload returned by headless API endpoints.
    Register in settings: HEADLESS_ADAPTER = "apps.users.adapters.HeadlessAdapter"
    """

    def get_user_dataclass(self) -> type:
        @dataclass
        class UserData:
            id: str
            email: str
            first_name: str
            last_name: str
            is_email_verified: bool
            has_mfa: bool
            avatar_url: str | None
        return UserData

    def user_as_dataclass(self, user: object) -> object:
        from allauth.account.models import EmailAddress
        UserData = self.get_user_dataclass()
        return UserData(
            id=str(user.pk),
            email=user.email,
            first_name=user.first_name,
            last_name=user.last_name,
            is_email_verified=EmailAddress.objects.filter(
                user=user, verified=True
            ).exists(),
            has_mfa=hasattr(user, "totpdevice_set") and user.totpdevice_set.exists(),
            avatar_url=getattr(user, "profile", None) and getattr(
                user.profile, "avatar_url", None
            ),
        )
```

---

## 8. JWT Bridge — Allauth + SimpleJWT

```python
# apps/users/views.py
"""
Pattern: Allauth handles authentication; SimpleJWT issues tokens.
Flow:
1. Client authenticates with allauth (social or email/password)
2. On success, call this endpoint to get JWT tokens
3. Client uses JWT for all subsequent API requests

This keeps allauth managing auth flows and SimpleJWT managing API tokens.
"""
from __future__ import annotations
from rest_framework.views import APIView
from rest_framework.request import Request
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from rest_framework_simplejwt.tokens import RefreshToken
from allauth.account.decorators import verified_email_required
from django.utils.decorators import method_decorator


class ObtainJWTFromSessionView(APIView):
    """
    Exchange an allauth session (just authenticated) for JWT tokens.
    Requires user to be logged in via allauth AND have a verified email.
    """
    permission_classes = [IsAuthenticated]

    @method_decorator(verified_email_required)
    def post(self, request: Request) -> Response:
        user = request.user
        refresh = RefreshToken.for_user(user)

        # Add custom claims
        refresh["email"] = user.email
        refresh["has_mfa"] = self._user_has_mfa(user)

        return Response({
            "access": str(refresh.access_token),
            "refresh": str(refresh),
        })

    def _user_has_mfa(self, user: object) -> bool:
        try:
            from allauth.mfa.models import Authenticator
            return Authenticator.objects.filter(user=user).exists()
        except ImportError:
            return False
```

---

## 9. Email Templates

```
# Allauth email templates — override by creating these files:
templates/
└── account/
    ├── email/
    │   ├── email_confirmation_subject.txt   ← Subject line
    │   ├── email_confirmation_message.txt   ← Plain text body
    │   ├── email_confirmation_message.html  ← HTML body (optional)
    │   ├── password_reset_key_subject.txt
    │   ├── password_reset_key_message.txt
    │   └── password_reset_key_message.html
    └── ...

# Example: templates/account/email/email_confirmation_message.txt
Hello {{ user.first_name }},

Please confirm your email address by clicking the link below:
{{ activate_url }}

This link expires in {{ expiration_days }} days.

Thanks,
The {{ site_name }} Team
```

---

## 10. Testing Allauth

```python
# apps/users/tests/test_auth.py
import pytest
from django.urls import reverse
from rest_framework.test import APIClient
from apps.users.tests.factories import UserFactory
from allauth.account.models import EmailAddress


@pytest.fixture
def verified_user():
    """A user with a verified email address."""
    user = UserFactory()
    EmailAddress.objects.create(
        user=user, email=user.email, verified=True, primary=True
    )
    return user


@pytest.fixture
def unverified_user():
    """A user with an unverified email address."""
    user = UserFactory()
    EmailAddress.objects.create(
        user=user, email=user.email, verified=False, primary=True
    )
    return user


@pytest.mark.django_db
class TestHeadlessLogin:
    url = "/_allauth/app/v1/auth/login"

    def test_login_with_verified_email(self, verified_user) -> None:
        client = APIClient()
        response = client.post(self.url, {
            "email": verified_user.email,
            "password": "SecurePass123!",
        }, format="json")
        assert response.status_code == 200

    def test_login_prevents_user_enumeration(self) -> None:
        client = APIClient()
        response = client.post(self.url, {
            "email": "nonexistent@example.com",
            "password": "wrongpassword",
        }, format="json")
        # Should return 400, NOT 404 (would reveal email doesn't exist)
        assert response.status_code in (400, 401)
        # Error message should be generic
        assert "nonexistent" not in str(response.content).lower()

    def test_unverified_user_cannot_login(self, unverified_user) -> None:
        client = APIClient()
        response = client.post(self.url, {
            "email": unverified_user.email,
            "password": "SecurePass123!",
        }, format="json")
        # Should fail or require verification
        assert response.status_code != 200
```

---

## 11. Security Checklist

```
Account Security:
✅ ACCOUNT_PREVENT_ENUMERATION = True (generic error on bad login)
✅ ACCOUNT_EMAIL_VERIFICATION = "mandatory"
✅ ACCOUNT_PASSWORD_MIN_LENGTH = 12
✅ MFA available and encouraged for all users
✅ Social login connects to existing account by email (pre_social_login)
✅ Rate limiting on login endpoint (5/minute throttle)
✅ Password reset links are single-use and expire (3 days)

Social OAuth:
✅ OAUTH_PKCE_ENABLED = True for all providers (prevents CSRF)
✅ SOCIALACCOUNT_LOGIN_ON_GET = False (requires POST to initiate)
✅ Client ID/secrets in environment variables, never in code
✅ SOCIALACCOUNT_EMAIL_REQUIRED = True
✅ Minimum required scopes for each provider

Email:
✅ Email confirmation expire = 3 days
✅ Email templates are custom branded
✅ All allauth emails routed through Celery (non-blocking)
✅ Email change notifies old email address

JWT Bridge:
✅ JWT issued ONLY after allauth session + verified email
✅ MFA status included in JWT claims
✅ Separate signing key for JWT (not DJANGO_SECRET_KEY)
```
