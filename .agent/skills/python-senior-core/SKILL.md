---
name: python-senior-core
description: >
  Apply this skill when writing, reviewing, or refactoring Python code. Covers senior-level
  Python patterns: type system, async/await, dataclasses, protocols, context managers,
  generators, descriptors, metaclasses, memory management, performance idioms, and
  Pythonic code style. Use when asked about Python best practices, code quality,
  refactoring, or any general Python implementation question.
---

# Senior Python Core Skill

> FIRST: Use Context7 to look up the latest Python documentation for any feature used.
> Python evolves rapidly — do not assume training-data knowledge is current.

---

## 1. Type System — Production Standards

### Always Use Type Hints
```python
from __future__ import annotations
from typing import TYPE_CHECKING, Any, TypeVar, Generic, Protocol, TypedDict
from collections.abc import Callable, Iterator, Generator, Sequence, Mapping
import typing

if TYPE_CHECKING:
    from myapp.models import User  # avoid circular imports in type hints

T = TypeVar("T")
ModelT = TypeVar("ModelT", bound="models.Model")
```

### TypedDict for Structured Dicts
```python
from typing import TypedDict, Required, NotRequired

class UserContextDict(TypedDict):
    user_id: int
    email: str
    role: str
    extra: NotRequired[dict[str, Any]]
```

### Protocol for Duck Typing (prefer over ABC for external types)
```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Notifiable(Protocol):
    def send_notification(self, message: str, recipient: str) -> bool: ...
    def get_channel_name(self) -> str: ...
```

### Generic Classes
```python
from typing import Generic

class Repository(Generic[ModelT]):
    def __init__(self, model_class: type[ModelT]) -> None:
        self.model_class = model_class

    def get_by_id(self, pk: int) -> ModelT | None:
        ...
```

---

## 2. Data Classes & Frozen Data

### Use `dataclasses` for Value Objects
```python
from dataclasses import dataclass, field
from decimal import Decimal

@dataclass(frozen=True, slots=True)
class Money:
    amount: Decimal
    currency: str

    def __post_init__(self) -> None:
        if self.amount < 0:
            raise ValueError("Amount cannot be negative")
        if len(self.currency) != 3:
            raise ValueError("Currency must be a 3-letter ISO code")

    def __add__(self, other: Money) -> Money:
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)
```

### Prefer `slots=True` for Memory Efficiency
```python
@dataclass(slots=True)
class EventPayload:
    event_type: str
    user_id: int
    timestamp: float
    data: dict[str, Any] = field(default_factory=dict)
```

---

## 3. Async/Await — Django Context

### Async Function Rules
```python
import asyncio
from asgiref.sync import sync_to_async, async_to_sync
from typing import AsyncIterator

# RULE: Mark coroutines clearly, document if CPU or IO bound
async def fetch_external_data(url: str, timeout: float = 30.0) -> dict[str, Any]:
    """Fetch data from external API. IO-bound. Requires ASGI server."""
    import httpx
    async with httpx.AsyncClient(timeout=timeout) as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()

# RULE: Use sync_to_async for Django ORM in async context
async def get_user_async(user_id: int) -> "User | None":
    from django.contrib.auth import get_user_model
    User = get_user_model()
    # thread_sensitive=True (default) shares thread with other sync code
    return await sync_to_async(User.objects.filter(pk=user_id).first)()
```

### Async Generators for Streaming
```python
async def stream_queryset(queryset: Any, batch_size: int = 100) -> AsyncIterator[Any]:
    """Memory-efficient async iteration over large querysets."""
    offset = 0
    while True:
        batch = await sync_to_async(list)(queryset[offset:offset + batch_size])
        if not batch:
            break
        for item in batch:
            yield item
        offset += batch_size
```

### Concurrent Task Execution
```python
async def fetch_multiple(urls: list[str]) -> list[dict[str, Any]]:
    """Fetch multiple URLs concurrently — NOT sequentially."""
    tasks = [fetch_external_data(url) for url in urls]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    # Filter out exceptions, log them
    return [r for r in results if not isinstance(r, Exception)]
```

---

## 4. Context Managers

### Custom Context Manager
```python
from contextlib import contextmanager, asynccontextmanager
from typing import Generator

@contextmanager
def measure_time(label: str) -> Generator[None, None, None]:
    import time
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        import logging
        logging.getLogger(__name__).debug("%s took %.3fs", label, elapsed)

# Usage
with measure_time("user lookup"):
    user = User.objects.get(pk=user_id)
```

### Resource Management with `__enter__`/`__exit__`
```python
class DatabaseConnectionPool:
    def __enter__(self) -> "DatabaseConnectionPool":
        self._acquire()
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc_val: BaseException | None,
        exc_tb: Any,
    ) -> bool:
        self._release()
        return False  # Do NOT suppress exceptions
```

---

## 5. Generators & Itertools for Efficiency

```python
from itertools import islice, chain, groupby
from collections.abc import Iterator, Iterable

def chunked(iterable: Iterable[T], size: int) -> Iterator[list[T]]:
    """Split iterable into chunks of `size`. Memory efficient."""
    it = iter(iterable)
    while chunk := list(islice(it, size)):
        yield chunk

# Usage — batch process DB records without loading all into memory
for batch in chunked(MyModel.objects.iterator(), 500):
    process_batch(batch)
```

---

## 6. Descriptors & Properties

```python
class ValidatedField:
    """Descriptor for validated model-like fields."""
    def __set_name__(self, owner: type, name: str) -> None:
        self.name = name
        self.private_name = f"_{name}"

    def __get__(self, obj: Any, objtype: type | None = None) -> Any:
        if obj is None:
            return self
        return getattr(obj, self.private_name, None)

    def __set__(self, obj: Any, value: Any) -> None:
        self._validate(value)
        setattr(obj, self.private_name, value)

    def _validate(self, value: Any) -> None:
        raise NotImplementedError
```

---

## 7. Exception Hierarchy

```python
# apps/core/exceptions.py
class AppBaseError(Exception):
    """Root exception for this application. Never catch Exception — catch this."""
    default_message: str = "An unexpected error occurred."
    default_code: str = "internal_error"

    def __init__(self, message: str | None = None, code: str | None = None) -> None:
        self.message = message or self.default_message
        self.code = code or self.default_code
        super().__init__(self.message)

class NotFoundError(AppBaseError):
    default_message = "Resource not found."
    default_code = "not_found"

class ValidationError(AppBaseError):
    default_message = "Validation failed."
    default_code = "validation_error"

class PermissionDeniedError(AppBaseError):
    default_message = "You do not have permission to perform this action."
    default_code = "permission_denied"

class ServiceUnavailableError(AppBaseError):
    default_message = "An external service is currently unavailable."
    default_code = "service_unavailable"
```

---

## 8. Logging — Production Patterns

```python
import logging
from typing import Any

logger = logging.getLogger(__name__)

# CORRECT: Structured logging with context
def process_order(order_id: int, user_id: int) -> None:
    logger.info(
        "Processing order",
        extra={"order_id": order_id, "user_id": user_id}
    )
    try:
        result = _do_process(order_id)
        logger.info("Order processed successfully", extra={"order_id": order_id})
    except PaymentError as exc:
        logger.error(
            "Payment failed for order",
            extra={"order_id": order_id, "error_code": exc.code},
            exc_info=True,  # includes traceback
        )
        raise
    except Exception:
        logger.exception("Unexpected error processing order", extra={"order_id": order_id})
        raise

# WRONG: Never do this
# print(f"Processing order {order_id}")
# logger.error(f"Error: {str(e)}")  # don't format in the string
```

---

## 9. Functional Patterns

```python
from functools import wraps, lru_cache, cached_property, partial
from typing import Callable, TypeVar

FuncT = TypeVar("FuncT", bound=Callable[..., Any])

def retry(max_attempts: int = 3, exceptions: tuple[type[Exception], ...] = (Exception,)):
    """Decorator for retrying a function on exception."""
    def decorator(func: FuncT) -> FuncT:
        @wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            last_exception: Exception | None = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as exc:
                    last_exception = exc
                    logger.warning(
                        "Attempt %d/%d failed for %s: %s",
                        attempt, max_attempts, func.__name__, exc
                    )
            raise last_exception  # type: ignore[misc]
        return wrapper  # type: ignore[return-value]
    return decorator
```

---

## 10. Python 3.13+ Features to Use

```python
# Match statements (3.10+) — use for type dispatch
match event.type:
    case "user.created":
        handle_user_created(event)
    case "order.completed":
        handle_order_completed(event)
    case _:
        logger.warning("Unknown event type: %s", event.type)

# Walrus operator for conditional assignment
if result := cache.get(cache_key):
    return result

# f-string = debugging (3.8+)
logger.debug(f"{user_id=}, {order_id=}")

# Exception groups (3.11+) for concurrent error handling
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(send_email(user))
        tg.create_task(update_record(user))
except* ValueError as eg:
    for exc in eg.exceptions:
        logger.error("Validation error: %s", exc)

# TypeAlias (3.12+)
type UserID = int
type OrderID = int
```

---

## 11. Code Quality Checklist

Before delivering any Python code, verify:

- [ ] All functions have type hints (parameters + return type)
- [ ] All public functions/classes have docstrings
- [ ] No `print()` statements — use `logging`
- [ ] No bare `except:` clauses
- [ ] No mutable default arguments (`def f(x=[])` — NEVER)
- [ ] No global mutable state
- [ ] Resource-acquiring code uses context managers
- [ ] Async functions are not called from sync context without `async_to_sync`
- [ ] Large iterables use generators, not list comprehensions loaded into memory
- [ ] Magic numbers have named constants
- [ ] Complex expressions have explaining variables
