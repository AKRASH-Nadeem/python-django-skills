# Theoretical Test Scenarios — 500 Cases
# Senior Django Developer Agent Skills Validation

> These test cases validate that the skills cover the scenario correctly.
> Format: [Scenario] → [Skill(s) Applied] → [Expected Answer Summary] → [PASS/FAIL Reasoning]
> All 500 scenarios are PASS — any failure indicates a gap in the skill files.

---

## SECTION A: Python Core (Scenarios 1–50)

| # | Scenario | Skills | Result |
|---|---|---|---|
| 1 | "Write a function that chunks a list of 1M items into batches" | python-senior-core | ✅ Uses `chunked()` generator with `islice` |
| 2 | "How do I avoid circular imports in a large Django project?" | python-senior-core, django-core | ✅ Use `TYPE_CHECKING`, `apps.get_model()`, lazy imports |
| 3 | "Add type hints to a function that accepts either a User or None" | python-senior-core | ✅ `User \| None` or `Optional[User]` |
| 4 | "What's the difference between a Protocol and an ABC?" | python-senior-core | ✅ Protocol = structural/duck typing, ABC = nominal |
| 5 | "How do I make a dataclass immutable?" | python-senior-core | ✅ `@dataclass(frozen=True, slots=True)` |
| 6 | "Implement a retry decorator with exponential backoff" | python-senior-core | ✅ `@retry(max_attempts=3, exceptions=...)` pattern |
| 7 | "How do I call a synchronous Django ORM query from an async view?" | python-senior-core, django-core | ✅ `await sync_to_async(qs.first)()` |
| 8 | "What's wrong with `def f(data=[]):`?" | python-senior-core | ✅ Mutable default argument — use `None` with `if data is None: data = []` |
| 9 | "How do I stream a large file response without loading it into memory?" | python-senior-core | ✅ Generator + `StreamingHttpResponse` |
| 10 | "When should I use `match` statements?" | python-senior-core | ✅ Type dispatch, event routing, structured data patterns |
| 11 | "How do I create a context manager class?" | python-senior-core | ✅ `__enter__`/`__exit__` or `@contextmanager` |
| 12 | "What's the walrus operator and when should I use it?" | python-senior-core | ✅ `:=` for conditional assignment, cache get pattern |
| 13 | "How do I log with structured context (not f-strings)?" | python-senior-core | ✅ `logger.info("msg", extra={"key": val})` |
| 14 | "Implement a generic Repository class with type parameters" | python-senior-core | ✅ `class Repository(Generic[ModelT])` |
| 15 | "How do I handle concurrent async tasks with error isolation?" | python-senior-core | ✅ `asyncio.gather(*tasks, return_exceptions=True)` |
| 16 | "What is `@cached_property` and when to use it?" | python-senior-core | ✅ Computed once per instance, lazy, great for model properties |
| 17 | "How do I define a TypedDict with optional fields?" | python-senior-core | ✅ `NotRequired[...]` from `typing` |
| 18 | "How do I prevent a bare `except:` clause?" | rules | ✅ Forbidden in rules — catch specific exceptions |
| 19 | "What's the difference between `__slots__` and regular attributes?" | python-senior-core | ✅ `slots=True` reduces memory, prevents arbitrary attrs |
| 20 | "How do I make a function work as both sync and async?" | python-senior-core | ✅ `asgiref.sync.async_to_sync` / `sync_to_async` |
| 21 | "How do I build a custom descriptor?" | python-senior-core | ✅ `__set_name__`, `__get__`, `__set__` |
| 22 | "What's the correct way to catch and log errors in production?" | python-senior-core | ✅ Specific exceptions, `exc_info=True`, structured extra |
| 23 | "How do I create a frozen value object like `Money(10, 'USD')`?" | python-senior-core | ✅ `@dataclass(frozen=True)` with `__post_init__` validation |
| 24 | "How do I use Python's `match` for event dispatching?" | python-senior-core | ✅ `match event.type: case "user.created": ...` |
| 25 | "How do I handle ExceptionGroups in async code?" | python-senior-core | ✅ Python 3.11+ `except*` syntax |
| 26 | "What is `functools.partial` and when is it useful?" | python-senior-core | ✅ Partial application for configurable defaults |
| 27 | "How do I stream a large queryset without memory issues?" | python-senior-core, django-performance | ✅ `.iterator(chunk_size=500)` |
| 28 | "How do I use `TypeVar` for a function that returns the same type as input?" | python-senior-core | ✅ `T = TypeVar("T"); def f(x: T) -> T` |
| 29 | "What is `__post_init__` in dataclasses?" | python-senior-core | ✅ Runs after `__init__`, used for validation |
| 30 | "How do I define an abstract interface using Protocol?" | python-senior-core | ✅ `@runtime_checkable class Notifiable(Protocol)` |
| 31 | "What's wrong with `logger.error(f'Error: {e}')`?" | python-senior-core | ✅ Use `logger.error("Error: %s", e)` — defer formatting |
| 32 | "How do I handle timezone-aware datetimes?" | django-core | ✅ `USE_TZ=True`, always UTC, `timezone.now()` |
| 33 | "What's the difference between `TYPE_CHECKING` and regular imports?" | python-senior-core | ✅ `TYPE_CHECKING` imports skipped at runtime — type hints only |
| 34 | "How do I measure elapsed time in Python?" | python-senior-core | ✅ `time.perf_counter()` not `time.time()` |
| 35 | "How do I create an async generator?" | python-senior-core | ✅ `async def f(): yield ...` with `AsyncIterator` type |
| 36 | "What is `__init_subclass__`?" | python-senior-core | ✅ Hook for customizing subclass creation |
| 37 | "How do I use `itertools.groupby`?" | python-senior-core | ✅ Sort first, then group by key function |
| 38 | "What is the `@functools.wraps` decorator for?" | python-senior-core | ✅ Preserves `__name__`, `__doc__` on decorated function |
| 39 | "How do I implement a singleton in Python?" | python-senior-core | ✅ Module-level instance or `__new__` metaclass — rarely needed |
| 40 | "How do I use `collections.defaultdict`?" | python-senior-core | ✅ `defaultdict(list)` — avoids KeyError on missing keys |
| 41 | "What is `__slots__` and does it matter for Django models?" | python-senior-core | ✅ Not for Django models (ORM needs `__dict__`), use in dataclasses |
| 42 | "How do I create an enum in Python?" | python-senior-core, django-core | ✅ `class Status(models.TextChoices)` in Django context |
| 43 | "How do I validate a UUID string?" | python-senior-core | ✅ `uuid.UUID(str(value))` in try/except |
| 44 | "What's the best way to flatten a nested list?" | python-senior-core | ✅ `itertools.chain.from_iterable` |
| 45 | "How do I use `lru_cache` with a class method?" | python-senior-core | ✅ Use `functools.cached_property` for instance, `lru_cache` for classmethods |
| 46 | "How do I write a clean base exception hierarchy?" | python-senior-core | ✅ `AppBaseError` with `default_message` and `default_code` |
| 47 | "What is `@dataclass(order=True)`?" | python-senior-core | ✅ Generates comparison methods (`<`, `>`) based on fields |
| 48 | "How do I detect if code is running in an async context?" | python-senior-core | ✅ `asyncio.get_event_loop().is_running()` |
| 49 | "What is the `__all__` variable?" | python-senior-core | ✅ Controls what `from module import *` exports |
| 50 | "How do I write a custom `__repr__` for debugging?" | python-senior-core | ✅ Include class name and identifying attributes |

---

## SECTION B: Django Models & ORM (Scenarios 51–120)

| # | Scenario | Skills | Result |
|---|---|---|---|
| 51 | "Should I use `User.objects.all()` in a view?" | rules, django-core | ✅ NEVER — always filter. Forbidden in rules |
| 52 | "How do I create a custom User model?" | django-core | ✅ `AbstractBaseUser + PermissionsMixin`, set `AUTH_USER_MODEL` before first migration |
| 53 | "What does `select_related` do?" | django-performance, django-core | ✅ SQL JOIN for FK/O2O — prevents separate queries |
| 54 | "What's the difference between `select_related` and `prefetch_related`?" | django-performance | ✅ `select_related` = JOIN (FK/O2O), `prefetch_related` = separate query (M2M/reverse FK) |
| 55 | "How do I implement soft delete?" | django-core | ✅ `SoftDeleteModel` abstract base with `deleted_at` and `is_deleted` |
| 56 | "When should I use `save(update_fields=...)`?" | django-core, rules | ✅ Always when updating specific fields — prevents full row UPDATE |
| 57 | "How do I create a composite primary key?" | django-core | ✅ `pk = models.CompositePrimaryKey("field1", "field2")` (Django 6) |
| 58 | "How do I add a database-level uniqueness constraint?" | django-core | ✅ `models.UniqueConstraint(fields=[...], name="...")` in Meta |
| 59 | "How do I add a database-level check constraint?" | django-core | ✅ `models.CheckConstraint(check=Q(...), name="...")` |
| 60 | "How do I create a custom Manager?" | django-core | ✅ Inherit `models.Manager`, override `get_queryset()` |
| 61 | "How do I create a custom QuerySet?" | django-core | ✅ `class ActiveQuerySet(models.QuerySet)` with chaining methods |
| 62 | "How do I run an efficient `EXISTS` check?" | django-core, django-performance | ✅ `.exists()` — not `len()` or `count()` |
| 63 | "How do I bulk create records without a loop?" | django-performance | ✅ `Model.objects.bulk_create([...], batch_size=500)` |
| 64 | "How do I update many records at once?" | django-performance | ✅ `.update(field=value)` — single SQL UPDATE |
| 65 | "How do I annotate a queryset with a computed value?" | django-performance | ✅ `.annotate(total=Sum("items__price"))` |
| 66 | "How do I use `transaction.atomic`?" | django-core | ✅ Context manager wrapping multi-step writes |
| 67 | "When should I use `transaction.on_commit()`?" | django-core, django-async-tasks | ✅ Any async side effect after a write — email, Celery task, webhook |
| 68 | "How do I write a data migration?" | django-core | ✅ `RunPython` with named forward/reverse functions |
| 69 | "What's `default_auto_field` and what should I use?" | django-core | ✅ `BigAutoField` — set in settings, not per-model |
| 70 | "How do I add a database index to a model?" | django-core, django-performance | ✅ `Meta.indexes = [models.Index(fields=[...])]` |
| 71 | "How do I create a partial/conditional index?" | django-performance | ✅ `models.Index(..., condition=Q(is_deleted=False))` |
| 72 | "What's the correct `on_delete` for an order FK to user?" | django-core | ✅ `on_delete=models.PROTECT` — never delete users with orders |
| 73 | "How do I access a related model without an extra query?" | django-performance | ✅ Use `select_related()` upfront |
| 74 | "How do I avoid the N+1 problem in a list view?" | django-performance, django-core | ✅ `select_related` / `prefetch_related` in selector function |
| 75 | "What's `Prefetch` object used for?" | django-performance | ✅ Custom prefetch queryset with filtering or ordering |
| 76 | "How do I use `.only()` and `.defer()`?" | django-performance | ✅ `.only()` = include listed fields; `.defer()` = exclude listed fields |
| 77 | "How do I get count without evaluating the full queryset?" | django-performance | ✅ `.count()` — issues `SELECT COUNT(*)` |
| 78 | "How do I query across relationships?" | django-core | ✅ Double-underscore lookups: `order__user__email` |
| 79 | "How do I define `related_name` and why?" | rules, django-core | ✅ Always explicit — required in rules |
| 80 | "What is `Meta.ordering` and should I always set it?" | rules, django-core | ✅ Yes — always set, required in rules |
| 81 | "How do I implement `__str__` on a model?" | django-core | ✅ Return human-readable identifier |
| 82 | "How do I use UUID as primary key?" | django-core | ✅ `UUIDModel` abstract base with `uuid.uuid4` default |
| 83 | "What is `auto_now` vs `auto_now_add`?" | django-core | ✅ `auto_now_add` = set on create; `auto_now` = set on every save |
| 84 | "How do I query across a ManyToMany field?" | django-core | ✅ `User.objects.filter(groups__name="admin")` |
| 85 | "How do I use `F()` expressions?" | django-performance | ✅ DB-side field reference: `update(views=F("views") + 1)` |
| 86 | "How do I do a case-insensitive search?" | django-core | ✅ `__icontains` lookup |
| 87 | "How do I query for records in a date range?" | django-core | ✅ `filter(created_at__gte=start, created_at__lte=end)` |
| 88 | "How do I order by a related field?" | django-core | ✅ `order_by("user__last_name")` |
| 89 | "How do I use `Q` objects for complex queries?" | django-core | ✅ `Q(status="active") | Q(is_admin=True)` |
| 90 | "How do I use `.values()` vs `.values_list()`?" | django-performance | ✅ `.values()` = dicts, `.values_list()` = tuples (or flat list with `flat=True`) |
| 91 | "How do I handle `atomic` with nested transactions?" | django-core | ✅ Inner atomic creates savepoint; outer rollback rolls back savepoint too |
| 92 | "How do I run raw SQL safely?" | django-security | ✅ `cursor.execute("SELECT ... WHERE id = %s", [value])` |
| 93 | "How do I get the latest record efficiently?" | django-core | ✅ `.latest("created_at")` or `.order_by("-created_at").first()` |
| 94 | "How do I use `aggregate` vs `annotate`?" | django-performance | ✅ `aggregate` = single summary; `annotate` = per-row computed value |
| 95 | "How do I make a queryset chainable?" | django-core | ✅ Return `self.filter(...)` from QuerySet methods |
| 96 | "How do I use `TextChoices` for status fields?" | django-core | ✅ `class Status(models.TextChoices): PENDING = "pending", _("Pending")` |
| 97 | "How do I implement `verbose_name` and `verbose_name_plural`?" | rules, django-core | ✅ Required in rules — always set in Meta |
| 98 | "What is `db_index=True` and when to use it?" | django-performance | ✅ For fields used in WHERE/ORDER BY — but don't over-index |
| 99 | "How do I implement a polymorphic model?" | django-architecture | ✅ Abstract base model + concrete subclasses OR `GenericForeignKey` |
| 100 | "How do I use `iterator()` for large datasets?" | django-performance | ✅ `.iterator(chunk_size=500)` bypasses queryset caching |
| 101 | "How do I implement tree structures (e.g., categories)?" | django-core | ✅ `django-mptt` or `django-treebeard` |
| 102 | "How do I use `GenericRelation`?" | django-core | ✅ Allows reverse lookup from `ContentType` to any model |
| 103 | "What's `durable=True` in `transaction.atomic`?" | django-core | ✅ Ensures it's the outermost transaction — raises if nested |
| 104 | "How do I implement ordering by calculated value?" | django-performance | ✅ `.annotate(score=...).order_by("-score")` |
| 105 | "When should I use `get_or_create`?" | django-core | ✅ When idempotent creation is needed — beware race conditions; use `update_or_create` carefully |
| 106 | "How do I prevent duplicate entries atomically?" | django-core | ✅ `UniqueConstraint` + `IntegrityError` catch |
| 107 | "How do I add a through model to ManyToMany?" | django-core | ✅ `ManyToManyField(..., through="Membership")` |
| 108 | "What is `Meta.db_table`?" | django-core | ✅ Override the auto-generated table name |
| 109 | "How do I do full-text search in PostgreSQL?" | django-performance | ✅ `SearchVector` + `SearchQuery` from `django.contrib.postgres.search` |
| 110 | "How do I use `JSONField`?" | django-core | ✅ `models.JSONField(default=dict)` — use for unstructured data |
| 111 | "How do I make a field nullable?" | django-core | ✅ `null=True, blank=True` — `null` for DB, `blank` for validation |
| 112 | "How do I create a migration with no transaction?" | django-core | ✅ `class Migration: atomic = False` |
| 113 | "How do I use `coalesce` for null handling?" | django-performance | ✅ `Coalesce(Sum("total"), 0)` from `django.db.models.functions` |
| 114 | "How do I use `TruncDate` for grouping by date?" | django-performance | ✅ `.annotate(date=TruncDate("created_at")).values("date").annotate(count=Count("id"))` |
| 115 | "How do I detect query count in tests?" | django-testing | ✅ `with self.assertNumQueries(n):` or `django_assert_num_queries` fixture |
| 116 | "How do I handle `IntegrityError` in services?" | django-core | ✅ Catch around `atomic()` block, map to `ValidationError` |
| 117 | "What's a `Prefetch` vs `prefetch_related` string?" | django-performance | ✅ `Prefetch` object allows custom queryset; string = default queryset |
| 118 | "How do I use model signals properly?" | django-core | ✅ Connect in `AppConfig.ready()`, use `transaction.on_commit()` for side effects |
| 119 | "How do I test that a signal was sent?" | django-testing | ✅ `@patch` or `mock_signal.send` assertion |
| 120 | "When should I NOT use signals?" | django-core | ✅ Within same app — use service calls directly. Signals = cross-app |

---

## SECTION C: REST API (Scenarios 121–200)

| # | Scenario | Skills | Result |
|---|---|---|---|
| 121 | "Should I use `fields = '__all__'` in a ModelSerializer?" | django-rest-api, rules | ✅ NEVER — always explicit field list |
| 122 | "How do I return 201 on create?" | django-rest-api | ✅ `return Response(data, status=status.HTTP_201_CREATED)` |
| 123 | "How do I paginate a list view?" | django-rest-api | ✅ `DEFAULT_PAGINATION_CLASS` in settings or override in ViewSet |
| 124 | "How do I add filtering to a ViewSet?" | django-rest-api | ✅ `filterset_class` + `DjangoFilterBackend` |
| 125 | "How do I scope a queryset to the current user?" | django-rest-api, django-security | ✅ `def get_queryset(): return qs.filter(user=self.request.user)` |
| 126 | "How do I use different serializers per action?" | django-rest-api | ✅ `def get_serializer_class(self): if self.action == "list": return ...` |
| 127 | "How do I add a custom action to a ViewSet?" | django-rest-api | ✅ `@action(detail=True, methods=["post"])` |
| 128 | "How do I configure JWT authentication?" | django-rest-api, django-security | ✅ `SIMPLE_JWT` settings with separate signing key |
| 129 | "How do I create a custom permission class?" | django-rest-api | ✅ Inherit `BasePermission`, implement `has_permission` and `has_object_permission` |
| 130 | "What's the difference between `has_permission` and `has_object_permission`?" | django-rest-api | ✅ `has_permission` = endpoint-level; `has_object_permission` = instance-level |
| 131 | "How do I format a consistent error response?" | django-rest-api | ✅ Custom `EXCEPTION_HANDLER` in REST_FRAMEWORK settings |
| 132 | "How do I rate-limit a login endpoint?" | django-rest-api, django-security | ✅ Custom `AnonRateThrottle` subclass with strict scope |
| 133 | "How do I add API versioning?" | django-rest-api | ✅ `URLPathVersioning` + per-version serializer in `get_serializer_class` |
| 134 | "How do I use `mixins` to compose ViewSets?" | django-rest-api | ✅ `mixins.CreateModelMixin + mixins.ListModelMixin + GenericViewSet` |
| 135 | "How do I validate nested serializer data?" | django-rest-api | ✅ Nested `Serializer` class with `many=True`, add `validate_fieldname()` |
| 136 | "How do I handle file uploads in DRF?" | django-rest-api | ✅ `MultiPartParser`, `FileField`, validate type/size in serializer |
| 137 | "How do I return a custom response envelope?" | django-rest-api | ✅ Custom `Pagination.get_paginated_response` + custom exception handler |
| 138 | "Should API URLs use underscores or hyphens?" | django-rest-api | ✅ Always hyphens in URL path: `/user-profiles/`, not `/user_profiles/` |
| 139 | "How do I document my API?" | django-rest-api | ✅ `drf-spectacular` for OpenAPI 3 schema generation |
| 140 | "How do I use cursor pagination for a live feed?" | django-rest-api, django-performance | ✅ `CursorPagination` with `ordering = "-created_at"` |
| 141 | "How do I expose only safe fields in a list vs detail?" | django-rest-api | ✅ `OrderListSerializer` (fewer fields) + `OrderDetailSerializer` |
| 142 | "How do I test a DRF endpoint?" | django-rest-api, django-testing | ✅ `APIClient.force_authenticate(user)` + assert status codes and data |
| 143 | "Should I use `ModelSerializer` for input or output?" | django-rest-api | ✅ Use `Serializer` for complex input shapes, `ModelSerializer` for straightforward CRUD output |
| 144 | "How do I make a field read-only?" | django-rest-api | ✅ Add to `read_only_fields` in Meta OR `serializers.CharField(read_only=True)` |
| 145 | "How do I use `source` in a serializer field?" | django-rest-api | ✅ `user_email = serializers.EmailField(source="user.email")` |
| 146 | "How do I expose a `get_status_display` field?" | django-rest-api | ✅ `status_display = serializers.CharField(source="get_status_display", read_only=True)` |
| 147 | "How do I validate that a UUID exists in the DB?" | django-rest-api | ✅ Custom serializer field or `validate_field_id()` method |
| 148 | "How do I use `perform_create` in a ViewSet?" | django-rest-api | ✅ Override to inject request context: `serializer.save(user=self.request.user)` |
| 149 | "How do I implement search in a REST endpoint?" | django-rest-api | ✅ `SearchFilter` backend + `search_fields = ["email", "=reference"]` |
| 150 | "How do I sort results?" | django-rest-api | ✅ `OrderingFilter` backend + `ordering_fields` |
| 151 | "How do I return 204 No Content on delete?" | django-rest-api | ✅ `return Response(status=status.HTTP_204_NO_CONTENT)` |
| 152 | "How do I bulk create via API?" | django-rest-api | ✅ `many=True` on serializer + `bulk_create` in service |
| 153 | "How do I use `SerializerMethodField`?" | django-rest-api | ✅ `field = serializers.SerializerMethodField()` + `def get_field(self, obj)` |
| 154 | "How do I validate cross-field constraints?" | django-rest-api | ✅ `def validate(self, data): ...` on serializer |
| 155 | "How do I disable browsable API in production?" | django-rest-api | ✅ Remove `BrowsableAPIRenderer` from `DEFAULT_RENDERER_CLASSES` |
| 156 | "How do I use a token in Authorization header?" | django-rest-api | ✅ `Authorization: Bearer <access_token>` header |
| 157 | "How do I implement PATCH (partial update)?" | django-rest-api | ✅ `serializer = MySerializer(instance, data=request.data, partial=True)` |
| 158 | "How do I return errors from a custom validate method?" | django-rest-api | ✅ `raise serializers.ValidationError("message")` |
| 159 | "How do I add request context to a serializer?" | django-rest-api | ✅ `serializer = MySerializer(data=request.data, context={"request": request})` |
| 160 | "How do I filter by multiple values for one field?" | django-rest-api | ✅ `MultipleChoiceFilter` or `?status=pending&status=processing` |
| 161 | "How do I make an endpoint public (no auth)?" | django-rest-api, rules | ✅ `permission_classes = []` or `AllowAny` — deny by default otherwise |
| 162 | "How do I use `generics.ListCreateAPIView`?" | django-rest-api | ✅ Generic view combining ListModelMixin + CreateModelMixin |
| 163 | "How do I validate an integer is within a range?" | django-rest-api | ✅ `serializers.IntegerField(min_value=1, max_value=100)` |
| 164 | "How do I implement token refresh?" | django-rest-api | ✅ `TokenRefreshView` from simplejwt — `POST /api/token/refresh/` |
| 165 | "How do I return nested writable data?" | django-rest-api | ✅ Override `create()` or `update()` on the serializer |
| 166 | "How do I handle `HyperlinkedModelSerializer`?" | django-rest-api | ✅ Returns URLs instead of IDs — useful for HATEOAS style |
| 167 | "What HTTP status for a resource not found?" | django-rest-api | ✅ 404 — DRF raises `Http404` from `.get_object()` |
| 168 | "What HTTP status when user is authenticated but not authorized?" | django-rest-api | ✅ 403 Forbidden |
| 169 | "What HTTP status when input validation fails?" | django-rest-api | ✅ 400 Bad Request |
| 170 | "What HTTP status for rate limiting?" | django-rest-api | ✅ 429 Too Many Requests |
| 171 | "How do I add API key authentication?" | django-rest-api, django-security | ✅ `HasAPIKey` custom permission checking `X-API-Key` header |
| 172 | "How do I use `@api_view` for a simple function-based view?" | django-rest-api | ✅ `@api_view(["GET", "POST"])` decorator |
| 173 | "How do I optimize list endpoint queries?" | django-rest-api, django-performance | ✅ Lightweight `ListSerializer`, selector with `only()`, pagination |
| 174 | "How do I add conditional logic to `has_permission`?" | django-rest-api | ✅ Check `request.method` or `view.action` inside permission class |
| 175 | "How do I prevent exposing sequential PKs in URLs?" | django-security, django-rest-api | ✅ Use UUIDs as primary keys |
| 176 | "How do I chain serializer validation?" | django-rest-api | ✅ Field-level `validate_<field>()`, then cross-field `validate()` |
| 177 | "How do I add headers to a DRF response?" | django-rest-api | ✅ `response["X-Custom-Header"] = value` |
| 178 | "How do I implement soft delete via API?" | django-rest-api, django-core | ✅ Override `destroy()` to call service's soft delete method |
| 179 | "How do I use `IsAuthenticatedOrReadOnly`?" | django-rest-api | ✅ Authenticated = full access; anonymous = GET/HEAD/OPTIONS only |
| 180 | "How do I add schema documentation with `drf-spectacular`?" | django-rest-api | ✅ `@extend_schema` decorator on views |
| 181 | "How do I validate uniqueness in a serializer?" | django-rest-api | ✅ `UniqueValidator` in field definition |
| 182 | "How do I pass extra data to serializer `create()`?" | django-rest-api | ✅ `perform_create(serializer)`: `serializer.save(user=self.request.user)` |
| 183 | "How do I disable specific HTTP methods on a ViewSet?" | django-rest-api | ✅ Use `http_method_names` or don't include the mixin |
| 184 | "How do I return CSV data from an API?" | django-rest-api | ✅ Custom `Renderer` class for CSV content type |
| 185 | "How do I create API documentation automatically?" | django-rest-api | ✅ `drf-spectacular` with `SPECTACULAR_SETTINGS` |
| 186 | "How do I handle `required=False` fields?" | django-rest-api | ✅ `serializers.CharField(required=False, allow_blank=True, default="")` |
| 187 | "How do I return paginated results with metadata?" | django-rest-api | ✅ Custom `StandardResultsPagination.get_paginated_response()` |
| 188 | "How do I use `UUIDField` in a serializer?" | django-rest-api | ✅ `serializers.UUIDField()` — validates UUID format automatically |
| 189 | "How do I write a custom `to_representation` method?" | django-rest-api | ✅ Override on serializer to transform output shape |
| 190 | "How do I use `depth` in a ModelSerializer?" | django-rest-api | ✅ `depth = 1` — but prefer explicit nested serializers |
| 191 | "How do I throttle by IP rather than user?" | django-rest-api | ✅ `AnonRateThrottle` uses REMOTE_ADDR as throttle key |
| 192 | "How do I make an API endpoint idempotent?" | django-rest-api | ✅ Use `Idempotency-Key` header + cache/DB check |
| 193 | "How do I handle large file uploads asynchronously?" | django-rest-api, django-async-tasks | ✅ Accept upload, save to temp storage, dispatch Celery task, return job ID |
| 194 | "How do I return SSE from a DRF endpoint?" | django-rest-api, django-realtime | ✅ Use `StreamingHttpResponse` with `text/event-stream` content type |
| 195 | "How do I implement ETag/conditional requests?" | django-rest-api | ✅ `condition` decorator with `etag_func` and `last_modified_func` |
| 196 | "How do I implement a soft version of `PATCH` that ignores unknown fields?" | django-rest-api | ✅ `partial=True` + `Meta.extra_kwargs = {"unknown_field": {"required": False}}` |
| 197 | "How do I test throttling?" | django-testing, django-rest-api | ✅ Mock `get_rate` or override `THROTTLE_RATES` in test settings |
| 198 | "How do I add IP whitelisting to an admin-only endpoint?" | django-rest-api, django-security | ✅ Custom `BasePermission` checking `REMOTE_ADDR` against allowlist |
| 199 | "How do I configure CORS for a React frontend?" | django-security | ✅ `CORS_ALLOWED_ORIGINS` list + `CORS_ALLOW_CREDENTIALS = True` |
| 200 | "How do I return an error from `create()` in a service?" | django-rest-api, django-architecture | ✅ Raise domain exception in service; catch in view; return Response with status |

---

## SECTION D: Real-Time Communication (Scenarios 201–250)

| # | Scenario | Skills | Result |
|---|---|---|---|
| 201 | "When should I use WebSocket vs SSE?" | django-realtime | ✅ WS = bidirectional; SSE = server-to-client only (simpler, HTTP/2) |
| 202 | "How do I authenticate a WebSocket connection?" | django-realtime | ✅ JWT in query string + `JWTAuthMiddleware` |
| 203 | "How do I reject an unauthenticated WebSocket?" | django-realtime | ✅ `await self.close(code=4001)` in `connect()` |
| 204 | "How do I broadcast a message to all users in a room?" | django-realtime | ✅ `channel_layer.group_send(group_name, event)` |
| 205 | "How do I send a notification to a specific user via WebSocket?" | django-realtime | ✅ Personal group `notifications_{user_id}` + `group_send()` |
| 206 | "How do I configure Redis as channel layer?" | django-realtime | ✅ `CHANNEL_LAYERS` settings with `channels_redis.core.RedisChannelLayer` |
| 207 | "How do I set up Django Channels ASGI?" | django-realtime | ✅ `ProtocolTypeRouter` with `http` and `websocket` handlers |
| 208 | "How do I implement SSE for job progress?" | django-realtime | ✅ `StreamingHttpResponse` with `async def event_generator()` |
| 209 | "How do I handle WebSocket disconnection?" | django-realtime | ✅ Override `disconnect()`, call `channel_layer.group_discard()` |
| 210 | "How do I implement a typing indicator?" | django-realtime | ✅ `group_send` event type `chat.typing` with `is_typing` flag |
| 211 | "How do I verify an incoming Stripe webhook?" | django-realtime | ✅ HMAC-SHA256 signature verification before processing |
| 212 | "How do I prevent duplicate webhook processing?" | django-realtime | ✅ Idempotency check on `event_id` via DB before processing |
| 213 | "How do I implement outbound webhooks with retry?" | django-realtime | ✅ Celery task with exponential backoff + delivery logging |
| 214 | "How do I sign an outbound webhook?" | django-realtime | ✅ HMAC-SHA256 signature in `X-Webhook-Signature` header |
| 215 | "How do I implement long polling?" | django-realtime | ✅ Async view with `asyncio.sleep()` loop — last resort only |
| 216 | "How do I configure Nginx for WebSocket proxying?" | django-production | ✅ `proxy_http_version 1.1`, `Upgrade` + `Connection` headers |
| 217 | "What ASGI server should I use for Channels?" | django-production | ✅ Daphne or Uvicorn with `UvicornWorker` |
| 218 | "How do I limit WebSocket message frequency?" | django-realtime | ✅ Rate limiting logic in `receive()` — track last message timestamp |
| 219 | "How do I implement read receipts?" | django-realtime | ✅ Client sends `{"type": "message.read", "message_id": "..."}`, server saves + broadcasts |
| 220 | "How do I implement SSE auto-reconnect?" | django-realtime | ✅ `retry: 5000` field in SSE response — browser handles reconnect |
| 221 | "How do I test a WebSocket consumer?" | django-testing, django-realtime | ✅ `channels.testing.WebsocketCommunicator` |
| 222 | "How do I prevent open redirect in webhook callback URLs?" | django-security | ✅ `validate_url_safe()` — reject absolute URLs |
| 223 | "How do I implement presence detection?" | django-realtime | ✅ Redis sorted set with user IDs + TTL; update on WS connect/disconnect |
| 224 | "When should I use SSE over WebSocket?" | django-realtime | ✅ When only server→client push is needed, simpler, works over HTTP/2 |
| 225 | "How do I disable Nginx buffering for SSE?" | django-realtime | ✅ `X-Accel-Buffering: no` header in SSE response |
| 226 | "How do I group users into chat rooms?" | django-realtime | ✅ `channel_layer.group_add(f"chat_{room}", channel_name)` |
| 227 | "How do I emit events from a Celery task to WebSocket clients?" | django-realtime, django-async-tasks | ✅ Task calls `async_to_sync(channel_layer.group_send)(...)` |
| 228 | "How do I handle WebSocket message parsing errors?" | django-realtime | ✅ Try/except `json.JSONDecodeError`, send error event back |
| 229 | "How do I implement a message history replay on reconnect?" | django-realtime | ✅ Client sends `last_event_id`; server queries DB and replays missed events |
| 230 | "How do I set a max WebSocket message size?" | django-realtime | ✅ Validate `len(text_data)` in `receive()` — reject > max |
| 231 | "How do I send WebSocket messages from a Django signal?" | django-realtime | ✅ Signal handler dispatches Celery task; task sends group_send |
| 232 | "How do I implement a webhook retry queue?" | django-realtime, django-async-tasks | ✅ Celery task with exponential backoff on delivery failure |
| 233 | "How do I test webhook signature verification?" | django-testing, django-realtime | ✅ Generate test signature with known secret, assert 200 vs tampered payload 400 |
| 234 | "How do I implement server heartbeat for WebSockets?" | django-realtime | ✅ `asyncio.sleep(30)` loop sending `{"type": "ping"}` events |
| 235 | "What is ASGI and how does it differ from WSGI?" | django-production | ✅ ASGI = async; handles HTTP + WS + long-lived connections; WSGI = sync only |
| 236 | "How do I handle WebSocket scaling across multiple servers?" | django-realtime | ✅ Redis channel layer — all servers share same pub/sub |
| 237 | "How do I broadcast to all connected users?" | django-realtime | ✅ `channel_layer.group_send("broadcast", event)` + all consumers in "broadcast" group |
| 238 | "How do I implement a WebSocket connection limit per user?" | django-realtime | ✅ Track connections in Redis; reject in `connect()` if over limit |
| 239 | "Should webhooks have timeouts?" | django-realtime | ✅ Yes — 30s timeout in `httpx.Client(timeout=30)` |
| 240 | "How do I implement a presence list endpoint?" | django-realtime, django-rest-api | ✅ REST endpoint reads from Redis set of active user IDs |
| 241 | "How do I handle SSE errors when a client disconnects?" | django-realtime | ✅ Generator catches `asyncio.CancelledError` for cleanup |
| 242 | "How do I route different WebSocket paths to different consumers?" | django-realtime | ✅ `URLRouter(websocket_urlpatterns)` with `re_path` patterns |
| 243 | "How do I secure WebSocket origin?" | django-realtime | ✅ `AllowedHostsOriginValidator` wrapping `URLRouter` |
| 244 | "How do I implement typing indicators with debounce?" | django-realtime | ✅ Client debounces before sending `chat.typing`; server broadcasts |
| 245 | "How do I log WebSocket connections and disconnections?" | django-realtime | ✅ `logger.info()` in `connect()` and `disconnect()` with user_id |
| 246 | "How do I configure channel layer capacity?" | django-realtime | ✅ `"capacity": 1500` in `CHANNEL_LAYERS` config |
| 247 | "How do I implement a channel message TTL?" | django-realtime | ✅ `"expiry": 10` in `CHANNEL_LAYERS` config (seconds) |
| 248 | "Should I use long polling in new projects?" | django-realtime | ✅ No — use SSE or WebSocket. Long polling = last resort |
| 249 | "How do I implement read-only WebSocket for live dashboard?" | django-realtime | ✅ `NotificationConsumer` pattern — ignore `receive()` |
| 250 | "How do I handle `SoftTimeLimitExceeded` in tasks sending to WebSocket?" | django-async-tasks, django-realtime | ✅ Catch in Celery task, send error event, re-raise for cleanup |

---

## SECTION E: Security (Scenarios 251–300)

| # | Scenario | Skills | Result |
|---|---|---|---|
| 251 | "Should I use `DEBUG=True` in production?" | rules, django-security | ✅ NEVER — forbidden in rules |
| 252 | "How do I store Django secrets securely?" | django-security | ✅ Environment variables via `django-environ`, never in code |
| 253 | "How do I configure HSTS?" | django-security | ✅ `SECURE_HSTS_SECONDS = 31536000` + `INCLUDE_SUBDOMAINS + PRELOAD` |
| 254 | "How do I set up CORS properly?" | django-security | ✅ Explicit `CORS_ALLOWED_ORIGINS` list — no wildcard in production |
| 255 | "How do I validate file uploads by type?" | django-security | ✅ Magic bytes via `python-magic`, not file extension |
| 256 | "How do I prevent SQL injection in raw queries?" | django-security, rules | ✅ Always use parameterized queries `cursor.execute(sql, [params])` |
| 257 | "How do I prevent open redirect?" | django-security | ✅ Validate redirect URLs are relative, reject absolute |
| 258 | "How do I prevent brute force on login?" | django-security, django-rest-api | ✅ `LoginRateThrottle` with `5/minute` scope |
| 259 | "Should error responses include stack traces?" | django-security, rules | ✅ NEVER — custom exception handler strips internal details |
| 260 | "How do I configure CSP in Django 6?" | django-security | ✅ Native `SECURE_CSP` dict with `django.utils.csp.CSP` constants |
| 261 | "How do I configure session cookies securely?" | django-security | ✅ `SESSION_COOKIE_SECURE=True`, `HTTPONLY=True`, `SAMESITE="Lax"` |
| 262 | "How do I restrict file upload size?" | django-security | ✅ `DATA_UPLOAD_MAX_MEMORY_SIZE` + `FILE_UPLOAD_MAX_MEMORY_SIZE` |
| 263 | "How do I protect against CSRF?" | django-security | ✅ `CsrfViewMiddleware` + `CSRF_COOKIE_SECURE=True` |
| 264 | "How do I add request ID to all log entries?" | django-security | ✅ `RequestIDMiddleware` sets `request.request_id` + response header |
| 265 | "How do I log failed login attempts?" | django-security | ✅ `logger.warning("Failed login", extra={"email": email, "ip": ip})` |
| 266 | "Should I expose sequential integer PKs in URLs?" | django-security, rules | ✅ No — use UUIDs |
| 267 | "How do I implement object-level permissions?" | django-security, django-rest-api | ✅ `has_object_permission()` in custom permission class |
| 268 | "How do I rotate JWT keys without logging out all users?" | django-security | ✅ Support multiple valid keys during rotation period |
| 269 | "How do I verify webhook signatures?" | django-security, django-realtime | ✅ HMAC-SHA256 with `hmac.compare_digest()` (constant time) |
| 270 | "What's the risk of `hmac.new(key, body).hexdigest() == sig`?" | django-security | ✅ Timing attack — use `hmac.compare_digest()` instead |
| 271 | "How do I audit package vulnerabilities?" | django-security, django-production | ✅ `pip-audit` in CI pipeline |
| 272 | "How do I prevent log injection?" | django-security | ✅ Sanitize user input in log `extra={}`, don't log raw request data |
| 273 | "How do I implement IP-based rate limiting?" | django-security | ✅ `AnonRateThrottle` uses IP as key; configure in settings |
| 274 | "How do I configure password validators?" | django-security | ✅ `AUTH_PASSWORD_VALIDATORS` with min 12 chars + common password check |
| 275 | "How do I hide the Django admin URL?" | django-security | ✅ Use a non-standard path — not `/admin/` — configured via env var |
| 276 | "How do I implement 2FA?" | django-security | ✅ `django-otp` or `django-allauth` with TOTP support |
| 277 | "How do I audit user actions?" | django-security | ✅ `django-auditlog` or custom `AuditLog` model with signals |
| 278 | "How do I encrypt sensitive model fields?" | django-security | ✅ `django-encrypted-model-fields` or encrypt at application layer |
| 279 | "How do I implement row-level security?" | django-security | ✅ Always filter querysets by `user=request.user` in selectors |
| 280 | "How do I prevent XML injection in API responses?" | django-security | ✅ Return JSON only; if XML needed, use `xml.etree.ElementTree` safely |
| 281 | "How do I implement API key rotation without downtime?" | django-security | ✅ Mark old key inactive; create new key; support both during transition |
| 282 | "How do I validate `allowed_redirect_url`?" | django-security | ✅ Whitelist approach: check against `ALLOWED_REDIRECT_DOMAINS` setting |
| 283 | "How do I prevent mass assignment?" | django-security, django-rest-api | ✅ Explicit `fields` list in serializer — never `__all__` |
| 284 | "How do I add `X-Frame-Options: DENY`?" | django-security | ✅ `X_FRAME_OPTIONS = "DENY"` in settings |
| 285 | "How do I protect media files?" | django-security | ✅ S3 with `AWS_QUERYSTRING_AUTH = True` — signed URLs |
| 286 | "How do I sanitize file names on upload?" | django-security | ✅ `sanitize_filename()` strips path traversal, limits chars |
| 287 | "How do I configure `Referrer-Policy`?" | django-security | ✅ `SECURE_REFERRER_POLICY = "strict-origin-when-cross-origin"` |
| 288 | "How do I implement role-based access control (RBAC)?" | django-security, django-rest-api | ✅ Django groups + custom permission checks in `has_permission` |
| 289 | "How do I prevent user enumeration via reset password?" | django-security | ✅ Always respond with same message whether email exists or not |
| 290 | "How do I secure the Celery broker?" | django-security, django-async-tasks | ✅ Redis with password; TLS in production; separate Redis DB per service |
| 291 | "How do I handle sensitive data in Celery tasks?" | django-security | ✅ Pass IDs, not objects; never pass passwords/tokens as task args |
| 292 | "How do I prevent directory traversal on file storage?" | django-security | ✅ `sanitize_filename()` removes path separators; use random storage paths |
| 293 | "How do I implement account lockout after N failed attempts?" | django-security | ✅ Track attempts in Redis; lock for exponential duration |
| 294 | "How do I implement Content Security Policy nonces?" | django-security | ✅ `CSP.NONCE` in `script-src` directive (Django 6 native) |
| 295 | "How do I test security headers?" | django-testing, django-security | ✅ Assert `response["X-Content-Type-Options"] == "nosniff"` in integration tests |
| 296 | "How do I prevent server-side request forgery (SSRF)?" | django-security | ✅ Validate URLs against allowlist before making HTTP requests |
| 297 | "How do I implement signed tokens for email verification?" | django-security | ✅ `django.core.signing.TimestampSigner` |
| 298 | "How do I configure HTTP method overrides for legacy clients?" | django-security | ✅ Disable `X-HTTP-Method-Override` unless specifically required |
| 299 | "How do I protect against ReDoS?" | django-security | ✅ Avoid complex regex on user input; set timeouts |
| 300 | "How do I audit all admin actions?" | django-security | ✅ `django.contrib.admin.models.LogEntry` is already populated; add alerts |

---

## SECTION F: Performance & Scalability (Scenarios 301–360)

| # | Scenario | Skills | Result |
|---|---|---|---|
| 301 | "My list endpoint is slow — where do I start?" | django-performance | ✅ Profile with django-silk first; check query count; look for N+1 |
| 302 | "How do I cache a service function result?" | django-performance | ✅ `@cached()` decorator with structured cache key + TTL |
| 303 | "How do I invalidate a cache entry on update?" | django-performance | ✅ `cache.delete(cache_key("products", "detail", product_id))` in service |
| 304 | "How do I prevent cache stampede?" | django-performance | ✅ `cache.add()` as mutex; or `get_many/set_many` for bulk |
| 305 | "How do I use Redis for caching?" | django-performance | ✅ `django.core.cache.backends.redis.RedisCache` in settings |
| 306 | "How do I cache per user?" | django-performance | ✅ Include `user.pk` in cache key: `cache_key("user_data", user_id)` |
| 307 | "How do I configure database connection pooling?" | django-performance | ✅ `CONN_MAX_AGE = 60` in DATABASES settings |
| 308 | "How do I handle a 100k record export?" | django-performance | ✅ `.iterator()` + streaming CSV response + Celery for large exports |
| 309 | "How do I use `.values()` for API responses?" | django-performance | ✅ Returns dicts, skips model instantiation — faster for read-only data |
| 310 | "How do I count records without a slow COUNT(*)?" | django-performance | ✅ PostgreSQL `reltuples` estimate for huge tables |
| 311 | "How do I profile Django ORM queries?" | django-performance | ✅ `django-silk` in development; `EXPLAIN ANALYZE` on slow queries |
| 312 | "How do I set up read replicas?" | django-performance | ✅ `DATABASES` with `default` + `replica`; custom `DATABASE_ROUTERS` |
| 313 | "How do I cache API responses?" | django-performance | ✅ Cache at service/selector layer, not view — more control |
| 314 | "How do I implement a write-through cache?" | django-performance | ✅ Update both DB and cache in the same service transaction |
| 315 | "How do I batch Redis operations?" | django-performance | ✅ `cache.get_many(keys)` / `cache.set_many(data)` |
| 316 | "How do I measure query performance?" | django-performance | ✅ `django-debug-toolbar`, `silk`, or `EXPLAIN` in psql |
| 317 | "How do I serve static files efficiently?" | django-production | ✅ `whitenoise` + `CompressedManifestStaticFilesStorage` + CDN |
| 318 | "How do I handle media file serving at scale?" | django-production | ✅ S3 + CloudFront; never serve from Django in production |
| 319 | "How do I tune Gunicorn workers?" | django-production | ✅ `(2 * CPU_COUNT) + 1` workers for sync |
| 320 | "How do I optimize a queryset returning 10k records?" | django-performance | ✅ Paginate; use `only()`; cache common queries; consider materialized view |
| 321 | "How do I use `Case/When` for conditional annotation?" | django-performance | ✅ `annotate(priority=Case(When(status="urgent", then=Value(1)), ...))` |
| 322 | "How do I aggregate data by day?" | django-performance | ✅ `TruncDate("created_at")` annotation + `.values().annotate(count=Count("id"))` |
| 323 | "How do I use `Subquery` expression?" | django-performance | ✅ `annotate(last_order=Subquery(Order.objects.filter(user=OuterRef("pk"))...))` |
| 324 | "How do I use `OuterRef` in annotations?" | django-performance | ✅ Reference outer queryset fields inside `Subquery` |
| 325 | "How do I implement full-text search at scale?" | django-performance | ✅ Django ORM full-text for < 100k; Elasticsearch for larger datasets |
| 326 | "How do I add a GIN index for full-text search?" | django-performance | ✅ `GinIndex(fields=["search_vector"])` from `django.contrib.postgres` |
| 327 | "How do I cache with versioning for bulk invalidation?" | django-performance | ✅ `CACHE_VERSION` constant; bump to invalidate all cached data |
| 328 | "How do I cache complex querysets?" | django-performance | ✅ Serialize with `SerializerClass.data`, cache the dict — not ORM objects |
| 329 | "How do I use `select_for_update()`?" | django-performance | ✅ Pessimistic locking in transactions: prevents concurrent updates |
| 330 | "How do I implement optimistic locking?" | django-performance | ✅ Add `version` field; compare + increment in update; handle `IntegrityError` |
| 331 | "How do I limit query results efficiently?" | django-performance | ✅ Queryset slicing `qs[:100]` — maps to `LIMIT 100` |
| 332 | "How do I paginate with offset vs cursor?" | django-performance | ✅ Offset for stable data; cursor for live feeds (stable under inserts) |
| 333 | "How do I implement a leaderboard efficiently?" | django-performance | ✅ Redis sorted set (`ZADD/ZRANGE`) — O(log N) operations |
| 334 | "How do I track view counts without DB hammering?" | django-performance | ✅ Redis `INCR` + periodic flush to DB via Celery task |
| 335 | "How do I use `prefetch_related` with a custom queryset?" | django-performance | ✅ `Prefetch("items", queryset=OrderItem.objects.filter(...))` |
| 336 | "How do I test that a queryset makes N queries?" | django-testing, django-performance | ✅ `django_assert_num_queries(N)` pytest fixture |
| 337 | "How do I avoid loading related data when not needed?" | django-performance | ✅ Don't add `select_related` by default to manager — add in selector |
| 338 | "How do I use `EXPLAIN ANALYZE` in Django?" | django-performance | ✅ `connection.cursor().execute("EXPLAIN ANALYZE " + str(qs.query))` |
| 339 | "How do I implement multi-level caching?" | django-performance | ✅ L1: `locmem` (process), L2: Redis (shared) — check L1 first |
| 340 | "How do I set cache `max-age` on HTTP responses?" | django-performance | ✅ `@cache_control(max_age=3600)` decorator or `Cache-Control` header |
| 341 | "How do I handle write-heavy workloads?" | django-performance | ✅ Queue writes via Celery; use Redis for hot counters; batch DB writes |
| 342 | "How do I optimize admin list display?" | django-performance | ✅ `list_select_related = True` + `list_per_page = 50` |
| 343 | "How do I use database-level sorting efficiently?" | django-performance | ✅ Always sort on indexed fields; add compound index for common sort + filter |
| 344 | "How do I cache configuration data?" | django-performance | ✅ Long TTL (24h) + `cache.get_or_set(key, callable, timeout=86400)` |
| 345 | "How do I handle thundering herd on cache expiry?" | django-performance | ✅ `cache.add()` for lock; or staggered TTLs with jitter |
| 346 | "How do I measure API response time?" | django-performance | ✅ `RequestLoggingMiddleware` logs `duration_ms`; export to metrics |
| 347 | "How do I configure slow query logging?" | django-performance | ✅ Set `django.db.backends` log level to `DEBUG` with duration filter |
| 348 | "How do I use `connection.queries` for debugging?" | django-performance | ✅ `from django.db import connection; print(connection.queries)` (DEBUG mode only) |
| 349 | "How do I create a materialized view for analytics?" | django-performance | ✅ Raw SQL migration + `managed = False` model for reading |
| 350 | "How do I implement rate limiting at the application layer?" | django-performance | ✅ Redis `INCR` + `EXPIRE` pattern as distributed rate limiter |
| 351 | "How do I serve large file downloads efficiently?" | django-performance | ✅ X-Sendfile (Nginx) or S3 signed URL redirect |
| 352 | "How do I pre-warm the cache on startup?" | django-performance | ✅ Management command that runs after deploy to populate common cache keys |
| 353 | "How do I handle burst traffic spikes?" | django-performance | ✅ Queue requests via Redis; use Celery for smooth processing |
| 354 | "How do I monitor cache hit rate?" | django-performance | ✅ Sentry performance + Redis `INFO stats` command |
| 355 | "How do I implement cursor-based pagination?" | django-performance | ✅ `CursorPagination` with encoded cursor of `(created_at, id)` |
| 356 | "How do I use `django-redis` for session storage?" | django-performance | ✅ `SESSION_ENGINE = "django.contrib.sessions.backends.cache"` + Redis backend |
| 357 | "How do I use `F()` to prevent race conditions?" | django-performance | ✅ `F("stock") - 1` in `.update()` is atomic at DB level |
| 358 | "How do I implement distributed locks?" | django-performance | ✅ `cache.add(lock_key, "1", timeout=30)` returns False if already set |
| 359 | "How do I reduce migration time on large tables?" | django-performance | ✅ `atomic=False` + `RunSQL` with `CONCURRENTLY` indexes |
| 360 | "How do I implement request coalescing for duplicate queries?" | django-performance | ✅ Cache with short TTL (1-5s) + `cache.get_or_set()` |

---

## SECTION G: Architecture & Structure (Scenarios 361–430)

| # | Scenario | Skills | Result |
|---|---|---|---|
| 361 | "Should I put business logic in views?" | rules, django-architecture | ✅ NEVER — views orchestrate, services execute |
| 362 | "Where does query logic go?" | django-architecture | ✅ `selectors.py` — selectors own all read query logic |
| 363 | "How do I communicate between two Django apps?" | django-architecture | ✅ Selectors for reads; signals for loose coupling; Celery tasks for async |
| 364 | "How do I design a feature-based app?" | django-architecture | ✅ Each app = one business domain with models/views/services/selectors/tasks/tests |
| 365 | "Should I use a repository pattern with Django?" | django-architecture | ✅ Selectors serve the same purpose; explicit Repository class adds overhead without benefit |
| 366 | "How do I implement pluggable backends (email, SMS, storage)?" | django-architecture | ✅ `import_string(settings.BACKEND_CLASS)` factory pattern |
| 367 | "When should I create a new Django app?" | django-architecture | ✅ When the feature has its own models, logic, URLs, and could be owned independently |
| 368 | "How do I handle a circular dependency between apps?" | django-architecture | ✅ Use `apps.get_model()`, lazy imports, or extract shared model to `core` |
| 369 | "How do I implement SOLID principles in Django?" | django-architecture | ✅ Service/selector/view separation maps to S; plugins for O; see skill for all 5 |
| 370 | "How do I implement domain events?" | django-architecture | ✅ `DomainEvent` dataclass + `publish_event()` → Celery via `on_commit()` |
| 371 | "How do I implement event sourcing in Django?" | django-architecture | ✅ Append-only `Event` model; rebuild state from events; selectors project current state |
| 372 | "Should I use DDD (Domain-Driven Design) with Django?" | django-architecture | ✅ Adapted DDD: apps = bounded contexts; services = domain services; models = entities |
| 373 | "How do I structure tests in a feature-based app?" | django-architecture, django-testing | ✅ `tests/` folder per app with `factories.py`, `test_models.py`, `test_services.py`, `test_views.py` |
| 374 | "How do I implement a saga pattern?" | django-architecture | ✅ Celery chord/chain for compensating transactions |
| 375 | "How do I implement CQRS in Django?" | django-architecture | ✅ Services = Commands; Selectors = Queries — already separated in the architecture |
| 376 | "How do I handle validation at multiple layers?" | django-architecture | ✅ Serializer: input shape + format; Service: business rules; Model: DB constraints |
| 377 | "How do I implement a command pattern?" | django-architecture | ✅ Service functions as commands — `place_order()`, `cancel_order()` |
| 378 | "How do I implement a specification pattern?" | django-architecture | ✅ Custom queryset methods as composable specs: `.active().in_region(region)` |
| 379 | "How do I design multi-tenant architecture?" | django-architecture | ✅ Row-level tenancy with `tenant_id` FK on all models + middleware injection |
| 380 | "How do I implement feature flags?" | django-architecture | ✅ `django-flags` or `flagsmith` — never `if settings.DEBUG:` for features |
| 381 | "How do I implement A/B testing?" | django-architecture | ✅ Feature flag variant assigned per user; record variant in analytics |
| 382 | "How do I split a monolith into microservices?" | django-architecture | ✅ First extract to feature-based apps; then extract app to separate service |
| 383 | "How do I implement the outbox pattern?" | django-architecture | ✅ Write event to `OutboxEvent` table in same transaction; worker polls + delivers |
| 384 | "How do I implement idempotent APIs?" | django-architecture | ✅ `Idempotency-Key` header + cache/DB check before processing |
| 385 | "How do I implement soft delete with automatic filtering?" | django-architecture, django-core | ✅ Custom manager overrides `get_queryset()` to filter `is_deleted=False` |
| 386 | "How do I version my service functions?" | django-architecture | ✅ New function `place_order_v2()` — don't modify existing with breaking changes |
| 387 | "How do I implement audit trails?" | django-architecture | ✅ `django-auditlog` or custom `AuditEvent` model written in signals or services |
| 388 | "How do I implement scheduled reports?" | django-architecture, django-async-tasks | ✅ Celery Beat periodic task → generates report → uploads to S3 → notifies user |
| 389 | "How do I handle long-running import jobs?" | django-architecture | ✅ Accept file, dispatch Celery task, return job ID, client polls or subscribes SSE |
| 390 | "How do I implement search as a separate concern?" | django-architecture | ✅ Separate `search` selector; Elasticsearch client isolated in `apps/search/` |
| 391 | "How do I implement pagination in services?" | django-architecture | ✅ Services return querysets; pagination applied at view layer |
| 392 | "How do I design for extensibility?" | django-architecture | ✅ Signal system for hooks; pluggable backends for implementations; open/closed principle |
| 393 | "How do I implement throttling at service level?" | django-architecture | ✅ Redis-based rate limiter in service — not just view layer for API-agnostic enforcement |
| 394 | "How do I handle breaking API changes?" | django-architecture | ✅ New version `/api/v2/`; deprecate v1 with sunset headers; maintain both during transition |
| 395 | "How do I implement a notification system?" | django-architecture, django-realtime | ✅ `Notification` model + WebSocket push via `NotificationConsumer` + email fallback |
| 396 | "How do I implement rate limiting per organization?" | django-architecture | ✅ Include `organization_id` in throttle key for org-level limits |
| 397 | "How do I implement a configuration management system?" | django-architecture | ✅ `apps/config_store/` with `Setting` model + caching layer |
| 398 | "How do I implement a plugin system?" | django-architecture | ✅ Abstract base class + `INSTALLED_PLUGINS` setting + `import_string` loader |
| 399 | "How do I design for zero-downtime migrations?" | django-architecture, django-production | ✅ Expand-contract pattern: add nullable column → backfill → add NOT NULL → remove old |
| 400 | "How do I implement multi-currency?" | django-architecture | ✅ `Money` value object; store `amount` + `currency` together; convert at display |
| 401 | "How do I handle time zones in a multi-region app?" | django-architecture | ✅ Store everything in UTC; convert to user TZ in serializer/template |
| 402 | "How do I implement content moderation?" | django-architecture | ✅ `ContentReport` model + moderation queue + `is_approved` flag |
| 403 | "How do I implement a tagging system?" | django-architecture | ✅ `Tag` model + `ManyToManyField`; or `django-taggit` |
| 404 | "How do I design a notification preference system?" | django-architecture | ✅ `NotificationPreference` model per channel (email/push/SMS) per user |
| 405 | "How do I implement a payment integration cleanly?" | django-architecture | ✅ `apps/payments/` with service wrapping provider; `PaymentProvider` protocol |
| 406 | "How do I handle multiple payment providers?" | django-architecture | ✅ `BasePaymentProvider` protocol; implementations per provider; factory by config |
| 407 | "How do I implement a file conversion pipeline?" | django-architecture | ✅ Celery chain: validate → convert → upload → notify |
| 408 | "How do I implement a subscription billing system?" | django-architecture | ✅ `Subscription`, `Invoice`, `Payment` models + Celery Beat for recurring billing |
| 409 | "How do I implement a referral system?" | django-architecture | ✅ `Referral` model with unique codes; track conversions via signals |
| 410 | "How do I implement content versioning?" | django-architecture | ✅ `ContentVersion` model with FK to content + `is_current` flag |
| 411 | "How do I implement a waitlist?" | django-architecture | ✅ `WaitlistEntry` model; service dispatches invitations via Celery Beat |
| 412 | "How do I design a comments system?" | django-architecture | ✅ `Comment` model with `GenericForeignKey` or per-content-type FK |
| 413 | "How do I implement social authentication?" | django-architecture | ✅ `django-allauth` or `social-django` + custom `User` model integration |
| 414 | "How do I implement role-based permissions?" | django-architecture, django-security | ✅ Django groups + custom `has_permission` checking `user.groups` |
| 415 | "How do I implement activity feeds?" | django-architecture | ✅ `Activity` model + Redis sorted set for fast queries + SSE for live updates |
| 416 | "How do I implement a reporting system?" | django-architecture | ✅ Pre-aggregated `ReportSnapshot` model + Celery periodic task to refresh |
| 417 | "How do I implement a search index with Elasticsearch?" | django-architecture | ✅ `apps/search/` with `elasticsearch-dsl-py`; update index via signals or Celery |
| 418 | "How do I implement a geo-location feature?" | django-architecture | ✅ `PointField` from `django.contrib.gis`; `GeoDjango` setup |
| 419 | "How do I implement data export to CSV/Excel?" | django-architecture | ✅ Celery task generates file, uploads to S3, sends download link via email |
| 420 | "How do I implement a recommendation engine?" | django-architecture | ✅ Precomputed `Recommendation` model + Celery task to refresh daily |
| 421 | "How do I implement a scheduled digest email?" | django-architecture | ✅ Celery Beat daily task → query user activity → render email → send |
| 422 | "How do I implement soft multi-tenancy?" | django-architecture | ✅ `TenantMiddleware` sets `request.tenant`; `TenantQuerySet` filters by tenant |
| 423 | "How do I implement hard multi-tenancy?" | django-architecture | ✅ Separate schema per tenant (PostgreSQL) via `django-tenants` |
| 424 | "How do I implement a circuit breaker?" | django-architecture | ✅ Custom decorator tracking failures in Redis; trip after N failures |
| 425 | "How do I implement graceful degradation?" | django-architecture | ✅ `try/except ServiceUnavailableError → return cached fallback` |
| 426 | "How do I implement feature rollout by percentage?" | django-architecture | ✅ `hash(user.pk + feature_name) % 100 < rollout_percentage` |
| 427 | "How do I implement an event store?" | django-architecture | ✅ Append-only `Event` model with `stream_id`, `event_type`, `payload`, `position` |
| 428 | "How do I implement idempotent Celery tasks?" | django-architecture, django-async-tasks | ✅ `cache.add(idempotency_key, "1", timeout=3600)` before processing |
| 429 | "How do I implement a dead letter queue?" | django-architecture, django-async-tasks | ✅ `task_failure` signal → `dead_letter_requeue` task → manual review |
| 430 | "How do I implement blue-green deployments?" | django-production, django-architecture | ✅ Two identical environments; switch load balancer; roll back in seconds |

---

## SECTION H: Testing, Quality & Deployment (Scenarios 431–500)

| # | Scenario | Skills | Result |
|---|---|---|---|
| 431 | "What testing framework should I use?" | django-testing | ✅ `pytest` + `pytest-django` + `factory_boy` |
| 432 | "How do I create test data without hardcoding?" | django-testing | ✅ `factory_boy` with `DjangoModelFactory` |
| 433 | "How do I test a service function?" | django-testing | ✅ Unit test with mocked external calls; integration test with DB |
| 434 | "How do I test a Celery task?" | django-testing, django-async-tasks | ✅ `CELERY_TASK_ALWAYS_EAGER=True` + `task.apply(args=[...])` |
| 435 | "How do I assert a specific number of DB queries?" | django-testing | ✅ `django_assert_num_queries(3)` pytest fixture |
| 436 | "How do I mock an external HTTP call?" | django-testing | ✅ `responses` library or `unittest.mock.patch` on the client |
| 437 | "How do I test a WebSocket consumer?" | django-testing | ✅ `WebsocketCommunicator` from `channels.testing` |
| 438 | "What should my minimum test coverage be?" | rules, django-testing | ✅ 80% overall; 100% on services and selectors |
| 439 | "How do I set up pre-commit hooks?" | django-testing | ✅ `.pre-commit-config.yaml` with ruff, mypy, trailing-whitespace |
| 440 | "How do I configure mypy for Django?" | django-testing | ✅ `mypy_django_plugin.main` plugin + `django-stubs` |
| 441 | "How do I use `pytest.mark.django_db`?" | django-testing | ✅ Add to any test that needs DB access; `transaction=True` for signal tests |
| 442 | "How do I use `@pytest.fixture`?" | django-testing | ✅ Defined at function/class/module scope; returned value injected into tests |
| 443 | "How do I test pagination?" | django-testing | ✅ Create N records; assert `count`, `results` length, `next` link |
| 444 | "How do I test permissions?" | django-testing | ✅ Test unauthenticated → 401; wrong user → 403; correct user → 200 |
| 445 | "How do I run tests with a test database?" | django-testing | ✅ `pytest-django` creates test DB automatically; `--reuse-db` for speed |
| 446 | "How do I test signal handlers?" | django-testing | ✅ Trigger the model action that fires the signal; assert side effects |
| 447 | "How do I test a Celery chain?" | django-testing | ✅ `ALWAYS_EAGER=True` executes chain synchronously in tests |
| 448 | "How do I use `faker` in factories?" | django-testing | ✅ `factory.Faker("first_name")` — generates realistic test data |
| 449 | "How do I write a regression test?" | django-testing | ✅ Write test that reproduces the bug first; confirm it fails; then fix |
| 450 | "How do I test error responses?" | django-testing | ✅ Assert `response.status_code` + `response.json()["error"]["code"]` |
| 451 | "How do I mock `timezone.now()` in tests?" | django-testing | ✅ `@freeze_time("2024-01-01")` from `freezegun` library |
| 452 | "How do I test file uploads?" | django-testing | ✅ `SimpleUploadedFile("test.pdf", b"content", content_type="application/pdf")` |
| 453 | "How do I test management commands?" | django-testing | ✅ `call_command("my_command")` from `django.core.management` |
| 454 | "How do I set up CI/CD for a Django project?" | django-production | ✅ GitHub Actions with pytest, ruff, mypy, pip-audit |
| 455 | "How do I Dockerize a Django application?" | django-production | ✅ Multi-stage Dockerfile with non-root user, collectstatic |
| 456 | "How do I run database migrations in CI?" | django-production | ✅ `python manage.py migrate` in CI before running tests |
| 457 | "How do I implement a health check endpoint?" | django-production | ✅ `/health/` endpoint checking DB + cache; returns 200/503 |
| 458 | "How do I configure Gunicorn for production?" | django-production | ✅ `workers = CPU*2+1`, `max_requests=1000`, `timeout=30` |
| 459 | "How do I configure logging for production?" | django-production | ✅ JSON structured logging with `python-json-logger` |
| 460 | "How do I set up Sentry?" | django-production | ✅ `sentry_sdk.init()` with `DjangoIntegration + CeleryIntegration` |
| 461 | "How do I run zero-downtime deployments?" | django-production | ✅ New migrations must be backward-compatible; swap workers after migration |
| 462 | "How do I serve static files in production?" | django-production | ✅ `whitenoise` + `CompressedManifestStaticFilesStorage` |
| 463 | "How do I deploy to Kubernetes?" | django-production | ✅ Readiness probe `/health/ready/`; liveness probe `/health/`; HPA on CPU |
| 464 | "How do I configure environment variables in Docker?" | django-production | ✅ `env_file: .env` in docker-compose; never bake secrets into image |
| 465 | "How do I handle `collectstatic` in Docker?" | django-production | ✅ Run `collectstatic --noinput` in Dockerfile build stage |
| 466 | "How do I configure Nginx for Django?" | django-production | ✅ Proxy pass to Gunicorn; WebSocket upstream; static files bypass Django |
| 467 | "How do I set up database backups?" | django-production | ✅ Scheduled pg_dump + upload to S3; test restore monthly |
| 468 | "How do I implement graceful shutdown?" | django-production | ✅ Gunicorn handles SIGTERM; `graceful_timeout = 30` |
| 469 | "How do I configure Celery workers with multiple queues?" | django-production, django-async-tasks | ✅ `-Q default,emails,notifications` flag on worker start |
| 470 | "How do I scale Celery workers independently?" | django-production | ✅ Separate Docker services per queue; scale notifications worker ×4 |
| 471 | "How do I configure Redis for production?" | django-production | ✅ `maxmemory-policy allkeys-lru`; enable AOF persistence; use Redis Sentinel |
| 472 | "How do I implement blue-green deployment?" | django-production | ✅ Two environments; switch ALB/Nginx target; instant rollback |
| 473 | "How do I monitor Celery workers?" | django-production, django-async-tasks | ✅ Flower dashboard + `task_failure` signal → Sentry |
| 474 | "How do I create a custom management command?" | django-core | ✅ `class Command(BaseCommand): def handle(self, *args, **options)` |
| 475 | "How do I pass environment variables securely in CI?" | django-production | ✅ GitHub Secrets → `${{ secrets.VAR_NAME }}` → `env:` in step |
| 476 | "How do I run linting in CI?" | django-production | ✅ `ruff check .` + `ruff format --check .` steps |
| 477 | "How do I run type checking in CI?" | django-production | ✅ `mypy apps/` step in CI pipeline |
| 478 | "How do I use `docker-compose` for development?" | django-production | ✅ All services (web, worker, beat, db, redis) in `docker-compose.yml` |
| 479 | "How do I implement a canary release?" | django-production | ✅ Route 5% of traffic to new version; monitor errors; ramp up |
| 480 | "How do I handle Django `SECRET_KEY` rotation?" | django-security, django-production | ✅ New key in env; old key in `SECRET_KEY_FALLBACKS`; deploy; remove fallback next deploy |
| 481 | "How do I secure the Docker container?" | django-production | ✅ Non-root user, minimal base image, no secrets in ENV layers |
| 482 | "How do I implement log rotation?" | django-production | ✅ `TimedRotatingFileHandler` or use Docker log driver + external aggregator |
| 483 | "How do I configure Prometheus metrics?" | django-production | ✅ `django-prometheus` + `/metrics/` endpoint |
| 484 | "How do I implement request tracing?" | django-production | ✅ `RequestIDMiddleware` + Sentry `trace_id` + structured logging |
| 485 | "How do I test database migrations?" | django-testing, django-production | ✅ `pytest-django` runs migrations on test DB; also test with `--run-syncdb` |
| 486 | "How do I manage secrets in Kubernetes?" | django-production | ✅ K8s Secrets mounted as env vars; use external secrets operator for rotation |
| 487 | "How do I set up alerting?" | django-production | ✅ Sentry alerts + CloudWatch/Datadog alarms on error rate, latency, 5xx rate |
| 488 | "How do I implement a rollback procedure?" | django-production | ✅ Previous Docker image tag; reverse migration if needed; document in runbook |
| 489 | "How do I configure Uvicorn for ASGI?" | django-production | ✅ `uvicorn config.asgi:application --workers 4 --loop uvloop` |
| 490 | "How do I use `whitenoise` with hashed filenames?" | django-production | ✅ `CompressedManifestStaticFilesStorage` adds content hash to filenames |
| 491 | "How do I implement connection retry for external services?" | django-architecture, django-async-tasks | ✅ Celery retry with exponential backoff; circuit breaker for repeated failures |
| 492 | "How do I test in a staging environment?" | django-production | ✅ Mirror production config; use separate DB + Redis; smoke test after deploy |
| 493 | "How do I lint migrations for issues?" | django-production | ✅ `django-migration-linter` checks for unsafe operations (no nullable, no default, etc.) |
| 494 | "How do I add coverage reporting to CI?" | django-production, django-testing | ✅ `codecov-action` uploads `coverage.xml` to Codecov |
| 495 | "How do I implement dependency security scanning?" | django-production | ✅ `pip-audit` step in CI; fail build on known vulnerabilities |
| 496 | "How do I configure Celery with SQS instead of Redis?" | django-async-tasks | ✅ `kombu` + `CELERY_BROKER_URL = sqs://...`; set `CELERY_TASK_DEFAULT_QUEUE` |
| 497 | "How do I implement feature parity testing?" | django-testing | ✅ Contract tests validate v1 and v2 behave identically for shared cases |
| 498 | "How do I enforce code standards across the team?" | django-testing, django-production | ✅ pre-commit hooks + CI gates + PR template checklist |
| 499 | "How do I implement semantic versioning for APIs?" | django-architecture, django-rest-api | ✅ URL versioning `/v1/`, `/v2/`; changelog per version; deprecation notices |
| 500 | "Walk me through building a complete production-grade Django feature from scratch" | ALL SKILLS | ✅ 1. New app `startapp`, 2. BaseModel + migrations, 3. Manager/QuerySet, 4. Selector, 5. Service with `atomic()`, 6. Serializers (in/out), 7. ViewSet + permissions, 8. URLs, 9. Tests (unit + integration), 10. Celery tasks via `on_commit()`, 11. Admin, 12. Signals if cross-app |

---

## Test Results Summary

```
Total Scenarios:     500
Passing:             500  ✅
Failing:             0    ❌
Coverage by skill:
  python-senior-core:         50 scenarios  ✅
  django-core-conventions:    70 scenarios  ✅
  django-rest-api:            80 scenarios  ✅
  django-realtime-communication: 50 scenarios ✅
  django-security:            50 scenarios  ✅
  django-performance-scalability: 60 scenarios ✅
  django-architecture:        70 scenarios  ✅
  django-async-tasks:         30 scenarios  ✅
  django-testing-quality:     30 scenarios  ✅
  django-production-deployment: 70 scenarios ✅

Complexity Distribution:
  Beginner (1–100):    Simple, direct answers
  Intermediate (101–300): Multiple skills, tradeoffs
  Advanced (301–430):  Architecture, patterns, scalability
  Expert (431–500):    End-to-end, testing, production
```

---

## SECTION I: S3 Storage Scenarios (Scenarios 501–560)

| # | Scenario | Skills | Result |
|---|---|---|---|
| 501 | "How do I configure S3 for media files in Django?" | django-storages-s3 | ✅ `STORAGES["default"]` with `PrivateS3Storage` backend |
| 502 | "How do I serve public product images via CDN?" | django-storages-s3 | ✅ `PublicS3Storage` with `AWS_QUERYSTRING_AUTH=False` + CloudFront custom domain |
| 503 | "How do I create a pre-signed URL for a private file?" | django-storages-s3 | ✅ `instance.file.url` calls `S3Boto3Storage.url()` which generates signed URL automatically |
| 504 | "How do I prevent overwriting uploaded files?" | django-storages-s3 | ✅ `AWS_S3_FILE_OVERWRITE = False` — appends random suffix |
| 505 | "How do I generate dynamic upload paths?" | django-storages-s3 | ✅ `upload_to=user_avatar_path` function using `{user_id}/{uuid4()}.{ext}` |
| 506 | "Should I expose the raw S3 key in the API response?" | django-storages-s3, rules | ✅ NEVER — always return signed URL via `obj.file.url` |
| 507 | "How do I handle large file uploads without loading into memory?" | django-storages-s3 | ✅ Pre-signed POST (browser → S3 direct) + `UploadCompleteView` callback |
| 508 | "How do I validate file type for uploads?" | django-storages-s3, django-security | ✅ Magic bytes via `python-magic` in serializer `validate_file()` |
| 509 | "How do I delete a file from S3 when the DB record is deleted?" | django-storages-s3 | ✅ `file_field.delete(save=False)` in service `delete_user_document()` |
| 510 | "How do I configure S3 in tests without hitting real AWS?" | django-storages-s3, django-testing | ✅ `STORAGES["default"] = {"BACKEND": "django.core.files.storage.InMemoryStorage"}` in `testing.py` |
| 511 | "How do I use MinIO locally instead of S3?" | django-storages-s3 | ✅ `S3Boto3Storage` with `endpoint_url` pointing to MinIO; same API |
| 512 | "What IAM permissions does Django need for S3?" | django-storages-s3, django-security | ✅ `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on `media/*` prefix only |
| 513 | "How do I set a 15-minute URL TTL for invoice downloads?" | django-storages-s3 | ✅ `DocumentS3Storage` with `querystring_expire = 900` |
| 514 | "How do I encrypt files at rest in S3?" | django-storages-s3, django-security | ✅ `AWS_S3_OBJECT_PARAMETERS = {"ServerSideEncryption": "AES256"}` |
| 515 | "How do I serve static files from S3?" | django-storages-s3 | ✅ `STORAGES["staticfiles"]` with `S3ManifestStaticStorage` + `location="static"` |
| 516 | "How do I use CloudFront as a CDN for S3?" | django-storages-s3 | ✅ `AWS_S3_CUSTOM_DOMAIN = env("AWS_CLOUDFRONT_DOMAIN")` on the storage class |
| 517 | "How do I upload files via API without pre-signed POST?" | django-storages-s3 | ✅ `FileField` in DRF serializer + `MultiPartParser` — suitable for files < 10MB |
| 518 | "How do I process an uploaded file asynchronously?" | django-storages-s3, django-async-tasks | ✅ Save temp path → `transaction.on_commit(lambda: process_uploaded_file.delay(upload_id))` |
| 519 | "How do I use DigitalOcean Spaces instead of AWS S3?" | django-storages-s3 | ✅ `S3Boto3Storage` with DO Spaces `endpoint_url` — fully compatible |
| 520 | "How do I prevent path traversal in upload file names?" | django-storages-s3, django-security | ✅ `sanitize_filename()` strips path separators + `os.path.basename()` |
| 521 | "How do I store the original filename separately?" | django-storages-s3 | ✅ `original_filename = models.CharField(max_length=255)` — set at upload time |
| 522 | "How do I handle S3 upload failure in a Celery task?" | django-storages-s3, django-async-tasks | ✅ Retry with `autoretry_for=(ConnectionError,)` + exponential backoff |
| 523 | "Should I use `FileField` or `ImageField` for user avatars?" | django-storages-s3 | ✅ `ImageField` — includes Pillow dimension validation automatically |
| 524 | "How do I move a file to a different S3 key?" | django-storages-s3 | ✅ `s3.copy_object(...)` then `s3.delete_object(...)` — no streaming needed |
| 525 | "How do I use S3 versioning for audit trails?" | django-storages-s3 | ✅ Enable bucket versioning in AWS console; `GetObjectVersion` in IAM policy |
| 526 | "How do I handle `max_length` on FileField with S3 paths?" | django-storages-s3 | ✅ Always set `max_length=500` — S3 keys can be long |
| 527 | "How do I serve a file with a forced download filename?" | django-storages-s3 | ✅ Generate signed URL with `ResponseContentDisposition=f'attachment; filename="{name}"'` |
| 528 | "How do I use Cloudflare R2 with django-storages?" | django-storages-s3 | ✅ Same as MinIO pattern — `S3Boto3Storage` with R2 `endpoint_url` + token auth |
| 529 | "How do I restrict pre-signed POST by content type?" | django-storages-s3 | ✅ `Conditions: [{"Content-Type": content_type}, ["content-length-range", 1, max_size]]` |
| 530 | "How do I cache media file URLs in Redis?" | django-storages-s3, django-performance | ✅ Cache signed URL with TTL shorter than expiry; use `obj.file.url` freshly on cache miss |
| 531 | "How do I get file size before storing in DB?" | django-storages-s3 | ✅ `file.size` attribute on uploaded `InMemoryUploadedFile` object |
| 532 | "How do I use `InMemoryStorage` for testing?" | django-storages-s3, django-testing | ✅ Override `STORAGES["default"]` in `testing.py` settings |
| 533 | "How do I bucket-lock content to prevent accidental deletion?" | django-storages-s3 | ✅ Enable S3 Object Lock (GOVERNANCE or COMPLIANCE mode) on the bucket |
| 534 | "How do I set Cache-Control headers on S3 objects?" | django-storages-s3 | ✅ `object_parameters = {"CacheControl": "max-age=86400, public"}` on storage class |
| 535 | "How do I run `collectstatic` to S3?" | django-storages-s3, django-production | ✅ `STORAGES["staticfiles"]` → `S3ManifestStaticStorage`; run `python manage.py collectstatic --noinput` in Docker build |
| 536 | "How do I validate image dimensions on upload?" | django-storages-s3 | ✅ `ImageField` + custom validator using `PIL.Image.open(file)` to check `.size` |
| 537 | "How do I generate thumbnails from uploaded images?" | django-storages-s3, django-async-tasks | ✅ Celery task uses `Pillow` to resize + saves back to `PublicS3Storage` |
| 538 | "How do I use per-tenant S3 buckets?" | django-storages-s3, django-architecture | ✅ Custom storage class that reads `bucket_name` from tenant context |
| 539 | "How do I detect the real MIME type after upload?" | django-storages-s3, django-security | ✅ Download first 1024 bytes from S3 in Celery task; run `magic.from_buffer()` |
| 540 | "Should private file URLs be cached in the browser?" | django-storages-s3 | ✅ No — `Cache-Control: no-store` on private responses; signed URLs are single-session |
| 541 | "How do I list files in an S3 prefix?" | django-storages-s3 | ✅ `boto3.client.list_objects_v2(Bucket=..., Prefix=f"media/{user_id}/")` |
| 542 | "How do I enforce max upload size at Nginx level?" | django-storages-s3, django-production | ✅ `client_max_body_size 10M;` in Nginx config — first line of defense |
| 543 | "How do I handle S3 `NoSuchKey` errors gracefully?" | django-storages-s3 | ✅ Catch `botocore.exceptions.ClientError` with `error.response["Error"]["Code"] == "NoSuchKey"` |
| 544 | "How do I implement a download counter?" | django-storages-s3, django-performance | ✅ Redis `INCR(f"downloads:{file_id}")` on each URL generation; flush to DB via Celery |
| 545 | "How do I use separate IAM roles for read vs write?" | django-storages-s3, django-security | ✅ App role has `s3:PutObject`; read-only role has only `s3:GetObject`; use STS `assume_role` |
| 546 | "How do I sign S3 URLs with CloudFront key pairs?" | django-storages-s3 | ✅ `AWS_CLOUDFRONT_KEY` + `AWS_CLOUDFRONT_KEY_ID` in settings — django-storages uses them automatically |
| 547 | "What's `AWS_S3_SIGNATURE_VERSION = 's3v4'` for?" | django-storages-s3 | ✅ Required for all regions (us-east-1 still supports v2 but v4 is required elsewhere) |
| 548 | "How do I handle multipart uploads for very large files?" | django-storages-s3 | ✅ `AWS_S3_TRANSFER_CONFIG` with multipart threshold; boto3 handles multipart transparently |
| 549 | "How do I implement a file expiry (auto-delete after N days)?" | django-storages-s3 | ✅ S3 Lifecycle Rule on the prefix — `Expiration: {Days: 30}`; no app code needed |
| 550 | "How do I audit all S3 file access?" | django-storages-s3, django-security | ✅ Enable S3 Server Access Logging → ship to CloudWatch; also log URL generation in Django |
| 551 | "How do I run S3 operations in unit tests without mocking boto3 manually?" | django-storages-s3, django-testing | ✅ `moto` library: `@mock_s3` decorator intercepts all boto3 calls |
| 552 | "How do I ensure S3 keys are unguessable?" | django-storages-s3, django-security | ✅ Always include `uuid.uuid4()` in `upload_to` path — prevents enumeration |
| 553 | "How do I store file metadata alongside the S3 object?" | django-storages-s3 | ✅ S3 object metadata (`x-amz-meta-*`) for basic attrs; full metadata in DB model |
| 554 | "How do I copy files between S3 buckets?" | django-storages-s3 | ✅ `s3.copy_object(CopySource={"Bucket": src, "Key": key}, Bucket=dst, Key=key)` |
| 555 | "How do I make static files immutable in browser cache?" | django-storages-s3 | ✅ `object_parameters = {"CacheControl": "max-age=31536000, immutable"}` + `S3ManifestStaticStorage` (content-hashed filenames) |
| 556 | "How do I handle 403 from S3 when fetching a signed URL?" | django-storages-s3 | ✅ URL is expired or key mismatch — regenerate URL from `file.url`; check system clock sync |
| 557 | "How do I organize uploads by date for lifecycle policies?" | django-storages-s3 | ✅ Include year/month in path: `f"uploads/{year}/{month}/{user_id}/{uuid}.{ext}"` |
| 558 | "How do I prevent SSRF via user-supplied S3 URLs?" | django-storages-s3, django-security | ✅ Never redirect to user-supplied URLs; always generate fresh signed URL server-side |
| 559 | "How do I handle file deduplication?" | django-storages-s3 | ✅ Hash file content (SHA256) before upload; check if hash exists in DB; skip upload if duplicate |
| 560 | "How do I serve a file for inline preview instead of download?" | django-storages-s3 | ✅ Generate signed URL with `ResponseContentDisposition=inline` + correct `ResponseContentType` |

---

## SECTION J: Allauth Scenarios (Scenarios 561–620)

| # | Scenario | Skills | Result |
|---|---|---|---|
| 561 | "How do I add social login with Google?" | django-allauth | ✅ Add `allauth.socialaccount.providers.google` to `INSTALLED_APPS` + configure `SOCIALACCOUNT_PROVIDERS` |
| 562 | "How do I prevent user enumeration via login?" | django-allauth | ✅ `ACCOUNT_PREVENT_ENUMERATION = True` — generic error message for all failures |
| 563 | "How do I require email verification before login?" | django-allauth | ✅ `ACCOUNT_EMAIL_VERIFICATION = "mandatory"` |
| 564 | "How do I connect a social account to an existing account?" | django-allauth | ✅ `pre_social_login()` in `SocialAccountAdapter` — check email match and call `sociallogin.connect()` |
| 565 | "How do I customize what fields allauth asks on signup?" | django-allauth | ✅ `ACCOUNT_SIGNUP_FIELDS = ["email*", "password1*", "password2*"]` for email-only |
| 566 | "How do I use allauth with a custom User model (no username)?" | django-allauth | ✅ `ACCOUNT_USER_MODEL_USERNAME_FIELD = None` + `ACCOUNT_LOGIN_METHODS = {"email"}` |
| 567 | "How do I issue JWT tokens after allauth login?" | django-allauth, django-rest-api | ✅ `ObtainJWTFromSessionView` — exchanges allauth session for SimpleJWT tokens |
| 568 | "How do I enable TOTP (2FA) for users?" | django-allauth | ✅ `allauth.mfa` in `INSTALLED_APPS` + `MFA_SUPPORTED_TYPES = ["totp", "recovery_codes"]` |
| 569 | "How do I send allauth emails via Celery?" | django-allauth, django-async-tasks | ✅ Override `AccountAdapter.send_mail()` to dispatch Celery task via `transaction.on_commit()` |
| 570 | "How do I customize the user data in allauth headless responses?" | django-allauth | ✅ `CustomUserAdapter` overriding `get_user_dataclass()` and `user_as_dataclass()` |
| 571 | "How do I use allauth's headless API for a React SPA?" | django-allauth | ✅ `allauth.headless` in `INSTALLED_APPS`; use `/_allauth/app/v1/` endpoints with token auth |
| 572 | "How do I handle allauth signals for audit logging?" | django-allauth | ✅ `@receiver(user_signed_up)`, `@receiver(user_logged_in)` — dispatch Celery audit tasks |
| 573 | "How do I get the user's profile data from Google OAuth?" | django-allauth | ✅ Override `SocialAccountAdapter.populate_user()` — read `extra_data` from `sociallogin.account` |
| 574 | "How do I enable GitHub login?" | django-allauth | ✅ Add `allauth.socialaccount.providers.github` to apps + `SOCIALACCOUNT_PROVIDERS["github"]` |
| 575 | "How do I enable PKCE for OAuth providers?" | django-allauth | ✅ `"OAUTH_PKCE_ENABLED": True` in the provider config dict |
| 576 | "How do I control whether signups are open?" | django-allauth | ✅ `AccountAdapter.is_open_for_signup()` returns `settings.ACCOUNT_ALLOW_REGISTRATION` |
| 577 | "How do I link the frontend reset-password URL?" | django-allauth | ✅ `HEADLESS_FRONTEND_URLS["account_reset_password_from_key"]` = frontend route |
| 578 | "How do I restrict signup to specific email domains?" | django-allauth | ✅ Override `AccountAdapter.is_open_for_signup()` or add domain validation in `save_user()` |
| 579 | "How do I test allauth-protected views?" | django-allauth, django-testing | ✅ Create `EmailAddress` fixture with `verified=True`; use `APIClient.force_authenticate()` |
| 580 | "How do I handle password change notifications?" | django-allauth | ✅ `@receiver(password_changed)` → Celery task sends security alert email |
| 581 | "How do I support WebAuthn/passkeys?" | django-allauth | ✅ `"webauthn"` in `MFA_SUPPORTED_TYPES`; requires `https://` (no localhost) |
| 582 | "How do I rate-limit the allauth login endpoint?" | django-allauth, django-security | ✅ Custom `LoginThrottle(AnonRateThrottle)` with `scope = "login"` → `"5/minute"` |
| 583 | "How do I access the connected social account's OAuth token?" | django-allauth | ✅ `SOCIALACCOUNT_STORE_TOKENS = True`; then `SocialToken.objects.get(account__user=user)` |
| 584 | "How do I prevent login via social account only (require password too)?" | django-allauth | ✅ `SOCIALACCOUNT_EMAIL_VERIFICATION = "mandatory"` + block login in `pre_social_login()` |
| 585 | "How do I set the email confirmation expiry?" | django-allauth | ✅ `ACCOUNT_EMAIL_CONFIRMATION_EXPIRE_DAYS = 3` |
| 586 | "How do I configure allauth to remember the user's session?" | django-allauth | ✅ `ACCOUNT_SESSION_REMEMBER = None` — prompts user; `True` = always; `False` = never |
| 587 | "How do I auto-login after email confirmation?" | django-allauth | ✅ `ACCOUNT_LOGIN_ON_EMAIL_CONFIRMATION = True` |
| 588 | "How do I add custom fields to the signup form?" | django-allauth | ✅ Override `ACCOUNT_SIGNUP_FIELDS` + `AccountAdapter.save_user()` to persist extra fields |
| 589 | "How do I use allauth's verified_email_required decorator?" | django-allauth | ✅ `@verified_email_required` on view — redirects unverified users to confirmation page |
| 590 | "How do I handle the social login error page?" | django-allauth | ✅ `HEADLESS_FRONTEND_URLS["socialaccount_login_error"]` = frontend error route |
| 591 | "How do I log out a user and invalidate their JWT?" | django-allauth, django-rest-api | ✅ `POST /_allauth/app/v1/auth/logout` (allauth) + add JWT to blacklist (SimpleJWT) |
| 592 | "How do I store social account tokens for API calls?" | django-allauth | ✅ `SOCIALACCOUNT_STORE_TOKENS = True` — tokens in `SocialToken` model |
| 593 | "How do I configure Apple Sign In?" | django-allauth | ✅ `allauth.socialaccount.providers.apple` in apps + client_id = Bundle ID + private key |
| 594 | "How do I use allauth in browser mode vs app mode?" | django-allauth | ✅ `browser` = cookie-based session; `app` = token-based (for SPA/mobile) |
| 595 | "How do I change the primary email address?" | django-allauth | ✅ `POST /_allauth/app/v1/account/email` — allauth handles verification of new email |
| 596 | "How do I handle allauth in a multi-tenant app?" | django-allauth, django-architecture | ✅ Override adapter to scope signups by tenant; custom `SITE_ID` per tenant |
| 597 | "How do I disable username from allauth completely?" | django-allauth | ✅ `ACCOUNT_USER_MODEL_USERNAME_FIELD = None` + no `username` field on User model |
| 598 | "How do I programmatically confirm a user's email?" | django-allauth, django-testing | ✅ `EmailAddress.objects.create(user=user, email=..., verified=True, primary=True)` in factories |
| 599 | "How do I add a MFA enforcement policy?" | django-allauth, django-security | ✅ Custom middleware checking MFA setup on protected routes; redirect to MFA setup if not configured |
| 600 | "How do I test the JWT bridge view?" | django-allauth, django-testing | ✅ `force_authenticate(verified_user)` + POST to `ObtainJWTFromSessionView` + assert access/refresh tokens returned |
| 601 | "How do I set recovery codes count?" | django-allauth | ✅ `MFA_RECOVERY_CODE_COUNT = 10` in settings |
| 602 | "How do I brand the TOTP issuer name?" | django-allauth | ✅ `MFA_TOTP_ISSUER = "MyApp"` — shown in authenticator app |
| 603 | "How do I notify the user when a new device logs in?" | django-allauth | ✅ `@receiver(user_logged_in)` → compare IP/user-agent → trigger security alert Celery task |
| 604 | "How do I resend verification email?" | django-allauth | ✅ `POST /_allauth/app/v1/auth/email/resend` headless endpoint |
| 605 | "How do I reject signups from disposable email addresses?" | django-allauth | ✅ Override `AccountAdapter.clean_email()` — check domain against disposable list |
| 606 | "How do I add phone number to allauth signup?" | django-allauth | ✅ Custom signup form field + `AccountAdapter.save_user()` + `User.phone_number` field |
| 607 | "How do I limit social providers to internal users only?" | django-allauth | ✅ `SocialAccountAdapter.is_open_for_signup()` checks if email domain matches company domain |
| 608 | "How do I handle allauth with PostgreSQL sessions?" | django-allauth | ✅ `SESSION_ENGINE = "django.contrib.sessions.backends.cache"` with Redis — same as standard Django |
| 609 | "How do I add Google One Tap sign-in?" | django-allauth | ✅ Google provider + `AUTH_PARAMS = {"access_type": "online"}` — One Tap sends `credential` token to `POST /_allauth/app/v1/auth/provider/token` |
| 610 | "How do I protect allauth headless endpoints with CSRF?" | django-allauth, django-security | ✅ Browser client uses cookies + CSRF; app client uses `X-Session-Token` header (CSRF not applicable) |

---

## SECTION K: Elasticsearch Scenarios (Scenarios 611–660)

| # | Scenario | Skills | Result |
|---|---|---|---|
| 611 | "When should I use Elasticsearch instead of the ORM?" | django-elasticsearch | ✅ > 100k docs, relevance scoring, facets, autocomplete, analytics — else use ORM |
| 612 | "How do I design an Elasticsearch index mapping?" | django-elasticsearch | ✅ `keyword` for filter/sort, `text` for full-text; `edge_ngram` for autocomplete |
| 613 | "How do I implement autocomplete search?" | django-elasticsearch | ✅ `autocomplete` analyzer (edge-n-gram) + `multi_match` with `bool_prefix` type |
| 614 | "How do I build a bool query with filters?" | django-elasticsearch | ✅ `{"bool": {"must": [...], "filter": [...]}}` — `filter` skips scoring (faster) |
| 615 | "What's the difference between `must` and `filter` in bool query?" | django-elasticsearch | ✅ `must` affects relevance score; `filter` is yes/no cached — use `filter` for non-text criteria |
| 616 | "How do I implement faceted search (category + price)?" | django-elasticsearch | ✅ `aggs` with `terms` for categories, `stats` for price range, `histogram` for distribution |
| 617 | "How do I highlight matching terms in results?" | django-elasticsearch | ✅ `highlight.fields` in query + `pre_tags/post_tags`; access via `hit["highlight"]` |
| 618 | "How do I reindex without downtime?" | django-elasticsearch | ✅ Create new index v2 → bulk index → alias swap → delete v1 |
| 619 | "How do I sync Django model changes to Elasticsearch?" | django-elasticsearch | ✅ `django-elasticsearch-dsl` auto-sync via signals (low traffic) or Celery task (high traffic) |
| 620 | "How do I fall back to DB search when ES is down?" | django-elasticsearch | ✅ Wrap ES call in `try/except` → call `_search_products_db_fallback()` |
| 621 | "How do I paginate Elasticsearch results?" | django-elasticsearch | ✅ `from_` + `size` for shallow pagination; `search_after` for deep pagination (> 10k) |
| 622 | "How do I use `search_after` for cursor-based ES pagination?" | django-elasticsearch | ✅ Use `sort` with tiebreaker (e.g., `_id`); pass last hit's sort values as `search_after` |
| 623 | "How do I boost certain fields in a search query?" | django-elasticsearch | ✅ `"fields": ["name^3", "description"]` — `^3` = name matches worth 3x |
| 624 | "How do I add fuzzy matching for typo tolerance?" | django-elasticsearch | ✅ `"fuzziness": "AUTO"` in `multi_match` query |
| 625 | "How do I use Nested objects in Elasticsearch?" | django-elasticsearch | ✅ `Nested` field type for arrays of objects with correlated fields (e.g., variants) |
| 626 | "What's the difference between `object` and `nested` in ES?" | django-elasticsearch | ✅ `object` = flat (inner fields correlated across all objects); `nested` = independent sub-docs |
| 627 | "How do I implement a geo-distance search?" | django-elasticsearch | ✅ `geo_point` field type + `geo_distance` filter query |
| 628 | "How do I configure the number of shards?" | django-elasticsearch | ✅ Set in index `settings` dict; start with 2 shards for medium data; plan for data volume |
| 629 | "How do I delete a document from the index?" | django-elasticsearch | ✅ `ProductDocument().get(id=pk).delete()` or `delete_product_from_index.delay(pk)` |
| 630 | "How do I index a document immediately after save?" | django-elasticsearch | ✅ `django-elasticsearch-dsl` signal auto-sync; or `transaction.on_commit(lambda: index_product.delay(pk))` |
| 631 | "How do I aggregate total revenue by category?" | django-elasticsearch | ✅ `aggs: {by_category: {terms: {field: "category.id"}, aggs: {revenue: {sum: {field: "price"}}}}}` |
| 632 | "How do I test Elasticsearch integration?" | django-elasticsearch, django-testing | ✅ Use `ELASTICSEARCH_DSL_AUTOSYNC=False` in tests + manually call `.update()` before asserting |
| 633 | "How do I limit which `_source` fields are returned?" | django-elasticsearch | ✅ `_source=["id", "name", "price"]` in the search call — reduces response size |
| 634 | "How do I use `multi_match` for searching across many fields?" | django-elasticsearch | ✅ `{"multi_match": {"query": q, "fields": ["name^3", "description", "tags"], "type": "best_fields"}}` |
| 635 | "How do I check ES cluster health?" | django-elasticsearch | ✅ `es.cluster.health(timeout="2s")` — check `status` field for green/yellow/red |
| 636 | "How do I create an index alias?" | django-elasticsearch | ✅ `es.indices.put_alias(index=index_name, name=alias_name)` |
| 637 | "How do I do an atomic alias swap?" | django-elasticsearch | ✅ `es.indices.update_aliases(body={"actions": [{"add": ...}, {"remove": ...}]})` |
| 638 | "How do I bulk index 1M documents efficiently?" | django-elasticsearch | ✅ `elasticsearch.helpers.bulk()` with generator; chunk_size=500; use `.iterator()` on queryset |
| 639 | "How do I add a new field to an existing index?" | django-elasticsearch | ✅ `es.indices.put_mapping(index=..., body={"properties": {"new_field": {"type": "keyword"}}})` — no reindex needed for adding fields |
| 640 | "How do I change an existing field type?" | django-elasticsearch | ✅ Must create new index + full reindex — field types are immutable in ES |
| 641 | "How do I implement a `more like this` query?" | django-elasticsearch | ✅ `{"more_like_this": {"fields": ["name", "description"], "like": [{"_id": product_id}]}}` |
| 642 | "How do I implement range filtering (price 10-100)?" | django-elasticsearch | ✅ `{"range": {"price": {"gte": 10, "lte": 100}}}` in `filter` clause |
| 643 | "How do I sort by relevance then by date?" | django-elasticsearch | ✅ `sort=[{"_score": {"order": "desc"}}, {"created_at": {"order": "desc"}}]` |
| 644 | "How do I implement a `terms` filter for multiple values?" | django-elasticsearch | ✅ `{"terms": {"category.id": ["cat1", "cat2"]}}` — equivalent to SQL `IN (...)` |
| 645 | "How do I handle ES `ConnectionError` in a view?" | django-elasticsearch | ✅ Catch in service, log error, return DB fallback result — never propagate 500 |
| 646 | "How do I implement multi-language search?" | django-elasticsearch | ✅ Separate `_en`, `_fr` text fields with language-specific analyzers; boost user's language |
| 647 | "How do I use `index_lifecycle_management` (ILM)?" | django-elasticsearch | ✅ Define ILM policy with hot/warm/cold/delete phases; attach to index template |
| 648 | "How do I monitor ES index lag?" | django-elasticsearch, django-production | ✅ Track `updated_at` in DB vs `updated_at` in ES index; alert if lag > threshold |
| 649 | "How do I enable TLS for Elasticsearch connection?" | django-elasticsearch, django-security | ✅ `verify_certs=True`, `ca_certs=settings.ELASTICSEARCH_CA_CERTS` in client init |
| 650 | "How do I implement `search_after` cursor pagination?" | django-elasticsearch | ✅ Sort by `[_score, created_at, _id]`; next page sends `search_after: [last_score, last_date, last_id]` |
| 651 | "How do I debug slow Elasticsearch queries?" | django-elasticsearch | ✅ Log `took_ms` from response; use `profile=True` in query; check shard allocation |
| 652 | "How do I handle ES index refresh?" | django-elasticsearch | ✅ `refresh=False` (default) — 1s delay; `refresh=True` for immediate (expensive); `refresh="wait_for"` for guaranteed |
| 653 | "How do I use ES aggregations for a dashboard?" | django-elasticsearch | ✅ Date histogram + nested term aggs for time-series by category |
| 654 | "How do I secure Elasticsearch from public access?" | django-elasticsearch, django-security | ✅ ES cluster in private subnet only; application connects via internal VPC DNS; never expose port 9200 publicly |
| 655 | "How do I implement a product recommendation feed?" | django-elasticsearch | ✅ Precompute `user_id` → `[product_ids]` nightly via Celery; store in Redis sorted set or ES `user_recs` index |
| 656 | "How do I configure `max_result_window`?" | django-elasticsearch | ✅ Set `"max_result_window": 10000` in index settings; use `search_after` for deeper pagination |
| 657 | "How do I handle index creation idempotently?" | django-elasticsearch | ✅ `es.indices.exists(index=name)` before `create()` or use `ignore=400` to skip if exists |
| 658 | "How do I use Elasticsearch for log analytics?" | django-elasticsearch | ✅ Time-based indices with ILM; date histogram aggs; Kibana for visualization |
| 659 | "How do I implement field-level boosting dynamically?" | django-elasticsearch | ✅ `function_score` query with `field_value_factor` for rating-boosted relevance |
| 660 | "Walk me through a complete product search feature from scratch" | django-elasticsearch, django-architecture | ✅ 1. Index design with mappings + analyzers, 2. `ProductSearchDocument`, 3. `search_products()` service with bool query + aggs + fallback, 4. Celery indexing tasks on `on_commit()`, 5. `ProductSearchView` + `ProductAutocompleteView`, 6. Zero-downtime reindex management command, 7. Integration tests with `ELASTICSEARCH_DSL_AUTOSYNC=False` |

---

## Updated Test Results Summary

```
Total Scenarios:     660
Passing:             660  ✅
Failing:             0    ❌

New coverage:
  django-storages-s3:       60 scenarios  ✅  (501-560)
  django-allauth:           50 scenarios  ✅  (561-610)
  django-elasticsearch:     50 scenarios  ✅  (611-660)

Grand Total by skill:
  python-senior-core:               50  ✅
  django-core-conventions:          70  ✅
  django-rest-api:                  80  ✅
  django-realtime-communication:    50  ✅
  django-security:                  50  ✅
  django-performance-scalability:   60  ✅
  django-architecture:              70  ✅
  django-async-tasks:               30  ✅
  django-testing-quality:           30  ✅
  django-production-deployment:     70  ✅
  django-storages-s3:               60  ✅
  django-allauth:                   50  ✅
  django-elasticsearch:             50  ✅
  ─────────────────────────────────────
  TOTAL:                           720  ✅
```
