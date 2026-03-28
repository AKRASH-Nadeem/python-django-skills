---
name: django-testing-quality
description: >
  Apply this skill for all testing, code quality, and CI/CD work: pytest-django,
  factory_boy, integration tests, API tests, mock patterns, coverage requirements,
  linting (ruff), formatting (black), type checking (mypy/pyright), and pre-commit
  hooks. Use when writing tests, setting up CI pipelines, or reviewing code quality.
---

# Django Testing & Code Quality Skill

> FIRST: For pytest-django docs, query Context7. For factory_boy docs, also query Context7.
> Testing is not optional — it's how senior developers communicate correctness.

---

## 1. Testing Philosophy

```
Pyramid (from base to top — most to fewest):

   /\         E2E Tests (Playwright/Selenium) — very few
  /  \        
 /    \       Integration Tests (API + DB) — moderate
/      \      
/________\    Unit Tests (pure logic, no DB) — most

Rules:
- Unit tests: no DB, no network, no filesystem
- Integration tests: real DB (pytest-django), real Redis (if needed)
- Mocking: mock at the boundary (external APIs, email), not internal logic
- Tests document behavior — test names read like sentences
- Test the interface, not the implementation
- Every bug gets a regression test before being fixed
```

---

## 2. pytest Configuration

```ini
# pyproject.toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings.testing"
python_files = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "--strict-markers",
    "--strict-config",
    "-ra",                # Show all non-passing tests at the end
    "--tb=short",
    "--cov=apps",
    "--cov-report=term-missing:skip-covered",
    "--cov-report=html:htmlcov",
    "--cov-fail-under=80",
]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks tests requiring external services",
    "unit: marks pure unit tests",
]

[tool.coverage.run]
source = ["apps"]
omit = [
    "*/migrations/*",
    "*/tests/*",
    "*/admin.py",
    "manage.py",
    "config/*",
]
```

### Test Settings
```python
# config/settings/testing.py
from config.settings.base import *

# Use fast hasher for tests
PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]

# In-memory cache for tests (isolated)
CACHES = {
    "default": {"BACKEND": "django.core.cache.backends.locmem.LocMemCache"}
}

# Suppress Celery in tests — tasks are unit-tested separately
CELERY_TASK_ALWAYS_EAGER = True
CELERY_TASK_EAGER_PROPAGATES = True

# Use test email backend
EMAIL_BACKEND = "django.core.mail.backends.locmem.EmailBackend"

# Disable channels layers in tests
CHANNEL_LAYERS = {"default": {"BACKEND": "channels.layers.InMemoryChannelLayer"}}

# Silence migration output
MIGRATION_MODULES: dict = {}  # Use when you want to skip migrations (faster)
```

---

## 3. Factories — Never Hardcode Test Data

```python
# apps/users/tests/factories.py
from __future__ import annotations
import factory
from factory.django import DjangoModelFactory
from django.contrib.auth import get_user_model

User = get_user_model()


class UserFactory(DjangoModelFactory):
    """Creates a valid User. Use this in ALL tests."""
    class Meta:
        model = User
        django_get_or_create = ("email",)  # Idempotent if email matches

    email = factory.Sequence(lambda n: f"user{n}@example.com")
    first_name = factory.Faker("first_name")
    last_name = factory.Faker("last_name")
    password = factory.PostGenerationMethodCall("set_password", "SecurePass123!")
    is_active = True


class StaffUserFactory(UserFactory):
    is_staff = True


class SuperUserFactory(UserFactory):
    is_staff = True
    is_superuser = True


# apps/orders/tests/factories.py
import factory
from factory.django import DjangoModelFactory
from apps.orders.models import Order, OrderItem
from apps.users.tests.factories import UserFactory
from decimal import Decimal


class OrderFactory(DjangoModelFactory):
    class Meta:
        model = Order

    user = factory.SubFactory(UserFactory)
    status = Order.Status.PENDING
    total = factory.Faker("pydecimal", left_digits=4, right_digits=2, positive=True)
    reference = factory.Sequence(lambda n: f"ORD-{n:06d}")


class OrderItemFactory(DjangoModelFactory):
    class Meta:
        model = OrderItem

    order = factory.SubFactory(OrderFactory)
    quantity = factory.Faker("pyint", min_value=1, max_value=10)
    unit_price = factory.Faker("pydecimal", left_digits=3, right_digits=2, positive=True)
```

---

## 4. Unit Tests — Pure Logic

```python
# apps/orders/tests/test_services.py
import pytest
from decimal import Decimal
from unittest.mock import patch, MagicMock
from apps.orders.services import calculate_order_total, apply_discount
from apps.core.exceptions import ValidationError


class TestCalculateOrderTotal:
    """Test order total calculation logic."""

    def test_empty_items_raises_validation_error(self) -> None:
        with pytest.raises(ValidationError, match="at least one item"):
            calculate_order_total(items=[])

    def test_single_item_calculates_correctly(self) -> None:
        items = [{"quantity": 2, "unit_price": Decimal("9.99")}]
        total = calculate_order_total(items=items)
        assert total == Decimal("19.98")

    def test_multiple_items_sum_correctly(self) -> None:
        items = [
            {"quantity": 1, "unit_price": Decimal("10.00")},
            {"quantity": 3, "unit_price": Decimal("5.50")},
        ]
        total = calculate_order_total(items=items)
        assert total == Decimal("26.50")

    def test_discount_reduces_total(self) -> None:
        total = Decimal("100.00")
        discounted = apply_discount(total, discount_percent=Decimal("10"))
        assert discounted == Decimal("90.00")

    def test_discount_cannot_exceed_100_percent(self) -> None:
        with pytest.raises(ValidationError):
            apply_discount(Decimal("100"), discount_percent=Decimal("110"))
```

---

## 5. Integration Tests — API Endpoints

```python
# apps/orders/tests/test_views.py
import pytest
from django.urls import reverse
from rest_framework import status
from rest_framework.test import APIClient
from apps.users.tests.factories import UserFactory
from apps.orders.tests.factories import OrderFactory


@pytest.fixture
def api_client() -> APIClient:
    return APIClient()


@pytest.fixture
def auth_client(api_client: APIClient) -> APIClient:
    user = UserFactory()
    api_client.force_authenticate(user=user)
    return api_client


@pytest.fixture
def auth_client_with_user(api_client: APIClient) -> tuple[APIClient, object]:
    user = UserFactory()
    api_client.force_authenticate(user=user)
    return api_client, user


@pytest.mark.django_db
class TestOrderCreateView:
    url = "/api/v1/orders/"

    def test_unauthenticated_returns_401(self, api_client: APIClient) -> None:
        response = api_client.post(self.url, {})
        assert response.status_code == status.HTTP_401_UNAUTHORIZED

    def test_create_order_returns_201(self, auth_client_with_user) -> None:
        client, user = auth_client_with_user
        payload = {
            "items": [{"product_id": str(ProductFactory().pk), "quantity": 2, "unit_price": "9.99"}],
            "shipping_address_id": str(AddressFactory(user=user).pk),
        }
        response = client.post(self.url, payload, format="json")
        assert response.status_code == status.HTTP_201_CREATED
        data = response.json()
        assert data["status"] == "pending"
        assert len(data["items"]) == 1

    def test_create_order_invalid_quantity_returns_400(self, auth_client: APIClient) -> None:
        payload = {
            "items": [{"product_id": "abc", "quantity": 0, "unit_price": "9.99"}],
        }
        response = auth_client.post(self.url, payload, format="json")
        assert response.status_code == status.HTTP_400_BAD_REQUEST
        error = response.json()["error"]
        assert error["code"] == "validation_error"

    def test_order_scoped_to_current_user(self, api_client: APIClient) -> None:
        user1 = UserFactory()
        user2 = UserFactory()
        order = OrderFactory(user=user2)

        api_client.force_authenticate(user=user1)
        url = f"/api/v1/orders/{order.pk}/"
        response = api_client.get(url)
        # user1 should not see user2's order
        assert response.status_code == status.HTTP_404_NOT_FOUND


@pytest.mark.django_db
class TestOrderListView:
    url = "/api/v1/orders/"

    def test_returns_paginated_response(self, auth_client_with_user) -> None:
        client, user = auth_client_with_user
        OrderFactory.create_batch(5, user=user)
        response = client.get(self.url)
        assert response.status_code == 200
        data = response.json()
        assert "results" in data
        assert "count" in data
        assert data["count"] == 5
```

---

## 6. Mocking External Services

```python
# apps/emails/tests/test_tasks.py
import pytest
from unittest.mock import patch, MagicMock
from apps.emails.tasks import send_welcome_email
from apps.users.tests.factories import UserFactory


@pytest.mark.django_db
class TestSendWelcomeEmail:

    @patch("apps.emails.services.EmailService.send_welcome")
    def test_sends_email_for_active_user(self, mock_send: MagicMock) -> None:
        user = UserFactory()
        mock_send.return_value = {"id": "msg_abc123"}

        result = send_welcome_email.apply(args=[str(user.pk)])

        assert result.status == "SUCCESS"
        mock_send.assert_called_once_with(user)
        assert result.result["status"] == "sent"

    @pytest.mark.django_db
    def test_skips_inactive_user(self) -> None:
        user = UserFactory(is_active=False)
        result = send_welcome_email.apply(args=[str(user.pk)])
        assert result.result["status"] == "skipped"

    @pytest.mark.django_db
    def test_retries_on_connection_error(self) -> None:
        user = UserFactory()
        with patch("apps.emails.services.EmailService.send_welcome") as mock_send:
            mock_send.side_effect = ConnectionError("Connection refused")
            result = send_welcome_email.apply(args=[str(user.pk)])
            # With CELERY_TASK_EAGER_PROPAGATES=True, exception propagates
            assert result.failed()
```

---

## 7. WebSocket Consumer Tests

```python
# apps/chat/tests/test_consumers.py
import pytest
import json
from channels.testing import WebsocketCommunicator
from channels.layers import get_channel_layer
from config.asgi import application
from apps.users.tests.factories import UserFactory


@pytest.mark.asyncio
@pytest.mark.django_db(transaction=True)
class TestChatConsumer:

    async def test_connect_authenticated(self) -> None:
        user = await UserFactory.acreate()
        from rest_framework_simplejwt.tokens import RefreshToken
        token = str(RefreshToken.for_user(user).access_token)

        communicator = WebsocketCommunicator(
            application, f"/ws/chat/general/?token={token}"
        )
        connected, _ = await communicator.connect()
        assert connected
        await communicator.disconnect()

    async def test_connect_unauthenticated_closes_with_4001(self) -> None:
        communicator = WebsocketCommunicator(application, "/ws/chat/general/")
        connected, code = await communicator.connect()
        assert not connected
        assert code == 4001

    async def test_send_message_broadcasts_to_room(self) -> None:
        user = await UserFactory.acreate()
        from rest_framework_simplejwt.tokens import RefreshToken
        token = str(RefreshToken.for_user(user).access_token)

        c1 = WebsocketCommunicator(application, f"/ws/chat/room1/?token={token}")
        c2 = WebsocketCommunicator(application, f"/ws/chat/room1/?token={token}")

        await c1.connect()
        await c2.connect()

        await c1.send_json_to({"type": "chat.message", "payload": {"content": "Hello!"}})
        response = await c2.receive_json_from()

        assert response["type"] == "chat.message"
        assert response["payload"]["content"] == "Hello!"

        await c1.disconnect()
        await c2.disconnect()
```

---

## 8. Code Quality Tools

```toml
# pyproject.toml

[tool.ruff]
target-version = "py313"
line-length = 100
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "UP",  # pyupgrade
    "N",   # pep8-naming
    "S",   # flake8-bandit (security)
    "DJ",  # flake8-django
]
ignore = ["E501"]  # Line length handled by formatter

[tool.ruff.per-file-ignores]
"tests/**/*.py" = ["S101"]  # Allow asserts in tests
"*/migrations/*.py" = ["N806", "N999"]

[tool.black]
line-length = 100
target-version = ["py313"]

[tool.mypy]
python_version = "3.13"
strict = true
ignore_missing_imports = true
plugins = ["mypy_django_plugin.main"]

[[tool.mypy.overrides]]
module = "*.migrations.*"
ignore_errors = true

[tool.django-stubs]
django_settings_module = "config.settings.testing"
```

---

## 9. Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: latest
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: latest
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: ["--maxkb=500"]
      - id: check-merge-conflict
      - id: detect-private-key

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: latest
    hooks:
      - id: mypy
        additional_dependencies: [django-stubs, djangorestframework-stubs]
```

---

## 10. Testing Checklist

```
Coverage:
✅ Minimum 80% coverage on new code
✅ 100% coverage on services.py and selectors.py
✅ Branch coverage enabled (--cov-branch)

Test Structure:
✅ One test file per source file
✅ Test class per function/endpoint (groups related scenarios)
✅ Test names read as sentences describing the behavior
✅ Arrange-Act-Assert structure in every test
✅ No shared mutable state between tests

Fixtures:
✅ factory_boy for all model instances (never manual Model.objects.create())
✅ No hardcoded PKs or emails in tests
✅ Fixtures scoped appropriately (function > class > module > session)

Mocking:
✅ Mock at external boundaries (HTTP, email, SMS, file storage)
✅ Never mock internal logic to make tests pass
✅ Use responses library for HTTP mocking, not unittest.mock for HTTP calls
✅ Assert mock calls (assert_called_once_with, etc.)

Performance:
✅ @pytest.mark.django_db only on tests that need DB
✅ @pytest.mark.django_db(transaction=True) only for tests needing transactions
✅ Use @pytest.fixture(scope="session") for expensive shared fixtures
✅ --reuse-db flag (pytest-django) for local development speed
```
