# Python Backend Internship — Evaluation Study Guide
### *From the desk of a Python Department Manager*

> This guide covers everything a Python backend intern is expected to know before stepping into a client-serving environment. Topics are ordered by **priority** — what gets asked first, and what gets you hired or let go. Go deep, not just broad.

---

## PART 1 — PYTHON CORE

---

### 1.1 Python Fundamentals (Non-Negotiable Basics)

These are table-stakes. If you stumble here, the evaluation ends early.

- **Data types**: `int`, `float`, `str`, `bool`, `bytes`, `NoneType`
- **Collections**: `list`, `tuple`, `set`, `dict`, `frozenset`
  - Mutability rules and when each is appropriate
  - Dict ordering (guaranteed from Python 3.7+)
  - Set operations: union, intersection, difference
- **Comprehensions**: list, dict, set, generator expressions
- **Unpacking**: `*args`, `**kwargs`, starred assignment
- **String handling**: f-strings, `.format()`, slicing, encoding/decoding (`utf-8`)
- **Type hints**: `from typing import List, Dict, Optional, Union, Tuple, Any`
  - Why they matter in a production codebase
  - `mypy` basics for static type checking
- **Truthiness and falsy values**: when `if obj` is dangerous vs. safe
- **`is` vs `==`**: identity vs. equality — a classic trap

---

### 1.2 Object-Oriented Programming (OOP)

One of the most heavily tested areas for backend developers.

#### Core Concepts
- **Classes and instances**: `__init__`, instance vs. class vs. static methods
- **`self` vs `cls`**: when and why
- **`@staticmethod` vs `@classmethod`**: practical use cases in services and factories
- **Properties**: `@property`, `@setter`, `@deleter` — encapsulation without sacrificing interface

#### Inheritance
- Single and multiple inheritance
- **Method Resolution Order (MRO)**: `__mro__`, `C3 linearization`
- `super()` — correct usage in cooperative multiple inheritance
- Abstract classes: `from abc import ABC, abstractmethod`
- Mixins — what they are, why Django uses them everywhere

#### Dunder (Magic) Methods
These come up constantly in Django/DRF customization:
- `__str__`, `__repr__`
- `__len__`, `__contains__`, `__iter__`, `__next__`
- `__getitem__`, `__setitem__`, `__delitem__`
- `__enter__`, `__exit__` — context managers
- `__eq__`, `__hash__` — critical for use in sets/dicts
- `__call__` — callable objects

#### Composition vs. Inheritance
- "Favour composition over inheritance" — what this means in practice
- Example: building a `NotificationService` that composes email + SMS handlers

---

### 1.3 Decorators

Decorators are everywhere in Django and DRF. You must understand them from first principles.

- Functions as first-class objects
- Closures and `nonlocal`
- Writing a decorator from scratch
- `functools.wraps` — why it matters (`__name__`, `__doc__` preservation)
- Decorators with arguments (decorator factories)
- Class-based decorators
- Stacking decorators and execution order

**Must-know standard library decorators:**
- `@staticmethod`, `@classmethod`, `@property`
- `@functools.lru_cache`, `@functools.cached_property`
- `@functools.wraps`

---

### 1.4 Iterators, Generators & Lazy Evaluation

Critical for handling large datasets in client projects — you don't load 500k records into memory.

- **Iterables vs iterators**: `__iter__` vs `__next__`, `StopIteration`
- **Generators**: `yield`, `yield from`, generator pipelines
- **`itertools`**: `chain`, `islice`, `groupby`, `product`, `combinations`, `permutations`
- Lazy evaluation: why it matters for database queries and file I/O
- Comparison: `list` vs `generator` for memory and performance
- `next()` with default values

---

### 1.5 Memory Management

Expect this to be asked in any senior-leaning evaluation. Understanding memory means writing efficient, leak-free code for client applications.

#### How Python Manages Memory
- **CPython memory allocator**: PyMalloc, object pools
- **Reference counting**: `sys.getrefcount()`, increment/decrement rules
- **Garbage collector (GC)**: `gc` module, cyclic reference detection
  - Generational GC: generations 0, 1, 2
  - `gc.collect()`, `gc.disable()`, when to use them
- **`__del__`**: why it's unreliable and what to use instead (context managers)

#### Practical Memory Topics
- **`sys.getsizeof()`**: understanding object overhead
- **`__slots__`**: reducing per-instance memory footprint
  - When to use: large numbers of instances (e.g., model-like objects)
  - Tradeoffs: no `__dict__`, breaks some introspection
- **Interning**: string interning (`sys.intern`), small integer caching (`-5` to `256`)
- **Weak references**: `weakref.ref`, `WeakValueDictionary` — avoid circular ref memory leaks
- **Memory profiling tools**: `tracemalloc`, `memory_profiler`, `objgraph`
- **Copy semantics**: shallow vs. deep copy (`copy.copy` vs. `copy.deepcopy`)
- Context managers (`with` statement) as the correct pattern for resource cleanup

---

### 1.6 Concurrency & Parallelism

This is a major component — client systems often need async APIs, background workers, and parallel processing.

#### The GIL (Global Interpreter Lock)
- What the GIL is and why CPython has it
- What it protects (object reference counts) and what it doesn't (your data)
- GIL and I/O-bound vs. CPU-bound work — the key distinction
- Thread safety despite the GIL: why `list.append()` is safe but `+=` isn't always

#### Threading (`threading` module)
- `Thread`, `start()`, `join()`, `daemon` threads
- **Race conditions**: what they are, how to reproduce them
- **`threading.Lock`**: `acquire()`, `release()`, `with lock:`
- **`threading.RLock`**: reentrant lock — when you need it
- **`threading.Event`**: signaling between threads
- **`threading.Semaphore`**: limiting concurrent access
- **`threading.local()`**: thread-local storage
- **`queue.Queue`**: producer-consumer pattern, thread-safe by design

#### Multiprocessing (`multiprocessing` module)
- When to use multiprocessing vs. threading (CPU-bound tasks)
- `Process`, `Pool`, `Pool.map()`, `Pool.starmap()`
- **IPC**: `Queue`, `Pipe`, `Value`, `Array` (shared memory)
- `Manager` for shared state across processes
- `concurrent.futures.ProcessPoolExecutor`

#### Asyncio (Async/Await)
- **Event loop**: how it works, single-threaded concurrency
- `async def`, `await`, `asyncio.run()`
- **Coroutines vs. tasks vs. futures**
- `asyncio.gather()` — running coroutines concurrently
- `asyncio.create_task()`
- `asyncio.sleep()` vs. `time.sleep()` — why you never block the event loop
- **`async with`** and **`async for`**: async context managers and iterators
- `asyncio.Queue` — async producer-consumer
- When asyncio is appropriate (I/O-bound), when it isn't (CPU-bound)
- ASGI vs. WSGI — relevance to Django async views

#### Concurrency Patterns
- Producer-consumer
- Worker pool
- Fan-out / fan-in
- Circuit breaker (conceptual, for client integrations)

---

### 1.7 Error Handling & Exceptions

- Exception hierarchy: `BaseException` → `Exception` → specific exceptions
- Custom exceptions: when to create them, naming conventions, adding context
- `try / except / else / finally` — what each block is for
- Re-raising: `raise` vs. `raise e` vs. `raise NewException from e`
- Context suppression: `contextlib.suppress`
- `ExceptionGroup` (Python 3.11+) — awareness-level
- Logging exceptions correctly: `logger.exception()` vs. `logger.error()`

---

### 1.8 Modules, Packages & Imports

- `import` mechanics: `sys.modules`, module caching
- Relative vs. absolute imports
- `__init__.py`: its role and what to expose
- Circular imports: how to detect and fix them
- `__all__`: controlling public API of a module
- `importlib` for dynamic imports (used in plugin-based systems)
- Virtual environments: `venv`, `pip`, `requirements.txt`, `pip freeze`

---

### 1.9 Standard Library Essentials

Things you use in real projects daily:

| Module | Key Usage |
|---|---|
| `os`, `pathlib` | File system, paths |
| `sys` | Runtime info, `argv`, `exit` |
| `datetime` | Date/time arithmetic, timezones |
| `json` | Serialization/deserialization |
| `re` | Regular expressions |
| `collections` | `defaultdict`, `Counter`, `OrderedDict`, `deque`, `namedtuple` |
| `functools` | `lru_cache`, `partial`, `reduce`, `wraps` |
| `itertools` | Lazy iterators, combinatorics |
| `contextlib` | `contextmanager`, `suppress`, `nullcontext` |
| `logging` | Structured logging, handlers, formatters |
| `unittest` / `pytest` | Testing |
| `dataclasses` | `@dataclass`, `field()`, `__post_init__` |
| `enum` | `Enum`, `IntEnum`, `Flag` |
| `uuid` | UUID generation |
| `hashlib` | Hashing |
| `subprocess` | Shell commands |
| `socket` | Low-level networking |

---

### 1.10 Testing

- **Unit testing**: testing individual functions/methods in isolation
- **Integration testing**: testing components working together
- `pytest`: fixtures, parametrize, markers, `conftest.py`
- Mocking: `unittest.mock.Mock`, `MagicMock`, `patch`, `patch.object`
- Test coverage: `pytest-cov`, what 80% coverage means (and doesn't)
- TDD basics: red-green-refactor cycle
- `factory_boy` for test data generation (common in Django projects)

---

## PART 2 — DJANGO

---

### 2.1 Django Architecture

A manager needs to know you understand the full request-response cycle before touching any code.

```
Browser Request
    → URL Dispatcher (urls.py)
        → Middleware (request phase)
            → View (logic)
                → ORM (database)
            → Template / Serializer (response data)
        → Middleware (response phase)
    → HTTP Response
```

- **MVT pattern**: Model, View, Template — how it differs from MVC
- `manage.py` and `django-admin` commands
- Django settings: `BASE_DIR`, `INSTALLED_APPS`, `MIDDLEWARE`, `DATABASES`, `STATIC_ROOT`, `MEDIA_ROOT`
- `settings.py` best practices: environment variables with `django-environ` or `python-decouple`
- `django.setup()` — when and why you need it outside of the web process

---

### 2.2 Models & ORM

The backbone of every Django application. Expect deep questions here.

#### Model Definition
- `models.Model` base class
- Field types: `CharField`, `TextField`, `IntegerField`, `FloatField`, `DecimalField`, `BooleanField`, `DateField`, `DateTimeField`, `JSONField`, `UUIDField`, `FileField`, `ImageField`
- Field options: `null`, `blank`, `default`, `unique`, `db_index`, `choices`, `verbose_name`, `editable`
- **`null=True` vs `blank=True`**: the difference and when to use each
- `Meta` class: `ordering`, `verbose_name`, `verbose_name_plural`, `unique_together`, `indexes`, `constraints`, `db_table`, `abstract`
- `__str__` method — always implement it

#### Relationships
- `ForeignKey`: `on_delete` options (`CASCADE`, `SET_NULL`, `PROTECT`, `RESTRICT`, `SET_DEFAULT`, `DO_NOTHING`)
- `OneToOneField`: when vs. `ForeignKey`
- `ManyToManyField`: direct vs. through model
- `related_name` and `related_query_name`
- `select_related` (SQL JOIN for FK/O2O) vs. `prefetch_related` (separate query for M2M/reverse FK)
- Understanding the N+1 query problem and how to solve it

#### QuerySet API
- `all()`, `filter()`, `exclude()`, `get()`, `first()`, `last()`, `exists()`, `count()`
- `order_by()`, `distinct()`, `values()`, `values_list()`, `only()`, `defer()`
- `annotate()`, `aggregate()` — with `Count`, `Sum`, `Avg`, `Max`, `Min`, `F`, `Q`
- **`F` expressions**: database-level operations, avoid race conditions
- **`Q` objects**: complex OR/AND queries
- **Lazy evaluation**: QuerySets are lazy — SQL fires on iteration/slicing/evaluation
- QuerySet chaining — each method returns a new QuerySet (immutable chain)
- `update()` vs. `save()` — when to use each, signals and `auto_now` behavior
- `bulk_create()`, `bulk_update()`, `get_or_create()`, `update_or_create()`
- Raw SQL: `raw()`, `connection.cursor()` — when to reach for it

#### Migrations
- `makemigrations`, `migrate`, `showmigrations`, `sqlmigrate`
- Migration dependencies
- Data migrations: `RunPython`
- Squashing migrations
- `--fake`, `--fake-initial` flags
- Migration conflicts in team environments

#### Model Managers
- Default manager: `Model.objects`
- Custom managers: overriding `get_queryset()` for soft-delete patterns
- Multiple managers on one model

#### Model Signals
- `pre_save`, `post_save`, `pre_delete`, `post_delete`, `m2m_changed`
- `@receiver` decorator and `Signal.connect()`
- When signals are appropriate vs. overriding `save()`
- Downsides of signals: hidden logic, testing complexity

---

### 2.3 Views

#### Function-Based Views (FBVs)
- `HttpRequest` and `HttpResponse` objects
- `request.method`, `request.GET`, `request.POST`, `request.FILES`, `request.user`
- Decorators: `@login_required`, `@permission_required`, `@require_http_methods`

#### Class-Based Views (CBVs)
- `View`, `TemplateView`, `RedirectView`
- Generic display views: `ListView`, `DetailView`
- Generic edit views: `CreateView`, `UpdateView`, `DeleteView`, `FormView`
- `dispatch()` — the entry point, how method routing works
- `get_queryset()`, `get_object()`, `get_context_data()`
- Mixins: `LoginRequiredMixin`, `PermissionRequiredMixin`, `UserPassesTestMixin`
- When FBVs are better than CBVs and vice versa

---

### 2.4 URL Configuration

- `path()` and `re_path()` with converters
- URL namespaces: `app_name` and `name` parameter
- `include()` for modular URL configs
- `reverse()` and `reverse_lazy()` — programmatic URL generation
- URL parameters: path converters (`<int:pk>`, `<slug:slug>`, `<uuid:id>`)

---

### 2.5 Middleware

- Middleware execution order (request: top-down, response: bottom-up)
- Writing custom middleware (class-based, function-based)
- Built-in middleware: `SecurityMiddleware`, `SessionMiddleware`, `AuthenticationMiddleware`, `CsrfViewMiddleware`
- Use cases: request logging, tenant identification, IP whitelisting, JWT injection

---

### 2.6 Authentication & Authorization

- `django.contrib.auth`: `User`, `AbstractUser`, `AbstractBaseUser`
- Custom user model — why you always define one at project start
- `authenticate()`, `login()`, `logout()`
- Session-based auth vs. token-based auth
- Permissions: model-level (`add`, `change`, `delete`, `view`), object-level
- Groups and permission assignment
- `has_perm()`, `has_module_perms()`
- `@login_required`, `LoginRequiredMixin`
- Password hashing: PBKDF2, bcrypt — Django's `make_password`, `check_password`

---

### 2.7 Django Caching

- Cache backends: `LocMemCache`, `FileBasedCache`, `RedisCache`, `DatabaseCache`
- Cache API: `cache.get()`, `cache.set()`, `cache.delete()`, `cache.get_or_set()`
- Per-view caching: `@cache_page`
- Template fragment caching: `{% cache %}`
- Low-level cache patterns: cache-aside, write-through
- Cache invalidation strategies (the hardest problem in CS)
- `django-redis` setup and configuration

---

### 2.8 Django Admin

- `ModelAdmin`: `list_display`, `list_filter`, `search_fields`, `ordering`, `readonly_fields`, `fieldsets`
- Custom admin actions
- `InlineModelAdmin`: `TabularInline`, `StackedInline`
- Overriding `save_model()`, `get_queryset()`
- Admin security: restricting to staff users, custom permissions

---

### 2.9 Forms (Awareness Level for API-focused roles)

- `Form` vs `ModelForm`
- `cleaned_data`, `is_valid()`, `save()`
- Custom validators
- Why DRF Serializers are the API equivalent of Forms

---

### 2.10 Django Security

- CSRF: how it works, `{% csrf_token %}`, `CsrfViewMiddleware`
- XSS: Django's auto-escaping in templates
- SQL injection: how Django's ORM prevents it (parameterized queries)
- Clickjacking: `X-Frame-Options`
- `SECRET_KEY`: rotation, environment separation
- `DEBUG = False` in production: what it changes
- `ALLOWED_HOSTS`
- HTTPS: `SECURE_SSL_REDIRECT`, `HSTS`

---

## PART 3 — DJANGO REST FRAMEWORK (DRF)

---

### 3.1 DRF Architecture & Setup

- Installing and configuring DRF in `settings.py`
- `DEFAULT_AUTHENTICATION_CLASSES`, `DEFAULT_PERMISSION_CLASSES`, `DEFAULT_RENDERER_CLASSES`
- `DEFAULT_PAGINATION_CLASS`, `PAGE_SIZE`
- Global vs. per-view settings

---

### 3.2 Serializers

The most important DRF component to master. Serializers are the contract between your API and the outside world.

#### Serializer Basics
- `Serializer` vs. `ModelSerializer` — when to use each
- Field types: `CharField`, `IntegerField`, `FloatField`, `BooleanField`, `DateTimeField`, `UUIDField`, `SerializerMethodField`, `ListField`, `DictField`
- `read_only`, `write_only`, `required`, `allow_null`, `default`, `source`
- `source`: mapping serializer field names to model field names or methods

#### Validation
- Field-level validation: `validate_<field_name>()`
- Object-level validation: `validate()`
- Custom validators: `validators=[]` on fields
- Raising `serializers.ValidationError` with structured error messages
- `UniqueTogetherValidator`, `UniqueValidator`

#### Nested Serializers
- Serializing related objects
- Writable nested serializers: overriding `create()` and `update()`
- `many=True` for lists

#### `SerializerMethodField`
- Read-only computed fields
- `get_<field_name>()` method signature
- Access to `self.context` (e.g., `request` object)

#### `create()` and `update()`
- How `serializer.save()` delegates to these
- Passing extra context: `serializer.save(user=request.user)`
- Transactional creates with `transaction.atomic()`

#### Performance
- `select_related` and `prefetch_related` in the view's `get_queryset()` — not in the serializer
- Avoid per-object database hits inside `SerializerMethodField`

#### Serializer Context
- Passing `context={'request': request}` for hyperlinked serializers and dynamic fields
- Accessing context in validators and `SerializerMethodField`

---

### 3.3 Views in DRF

#### APIView
- The base class for all DRF views
- `get()`, `post()`, `put()`, `patch()`, `delete()` methods
- `request.data`, `request.user`, `request.auth`, `request.query_params`
- `Response` object and status codes (`status.HTTP_200_OK`, etc.)

#### Generic Views
- `GenericAPIView`: adds `get_queryset()`, `get_serializer()`, `get_object()`
- Concrete generic views: `ListAPIView`, `CreateAPIView`, `RetrieveAPIView`, `UpdateAPIView`, `DestroyAPIView`
- Combo views: `ListCreateAPIView`, `RetrieveUpdateDestroyAPIView`
- Overriding `get_queryset()` for dynamic filtering
- Overriding `perform_create()`, `perform_update()`, `perform_destroy()`

#### ViewSets
- `ViewSet`: base, manual URL mapping
- `ModelViewSet`: full CRUD, auto-routing
- `ReadOnlyModelViewSet`: list + retrieve only
- `@action` decorator: custom endpoints on viewsets (`detail=True/False`, `methods`, `url_path`)
- `get_serializer_class()`: different serializers per action
- `get_permissions()`: different permissions per action
- `get_queryset()`: dynamic queryset based on action or user

#### Routers
- `DefaultRouter` vs `SimpleRouter`
- `router.register()`: `prefix`, `viewset`, `basename`
- Nested routers: `drf-nested-routers`
- URL naming conventions generated by routers

---

### 3.4 Authentication in DRF

- **`BasicAuthentication`**: HTTP Basic — dev/testing only
- **`SessionAuthentication`**: CSRF-required, for browser clients
- **`TokenAuthentication`**: DRF built-in, one token per user
- **JWT (`djangorestframework-simplejwt`)**:
  - Access token + Refresh token flow
  - `TokenObtainPairView`, `TokenRefreshView`, `TokenVerifyView`
  - Token expiry, blacklisting (`BLACKLIST_AFTER_ROTATION`)
  - Custom claims in JWT payload
- `request.user`, `request.auth` — what they contain per auth class
- Writing custom authentication classes

---

### 3.5 Permissions in DRF

- Built-in: `AllowAny`, `IsAuthenticated`, `IsAdminUser`, `IsAuthenticatedOrReadOnly`
- `DjangoModelPermissions`, `DjangoObjectPermissions`
- Writing custom permission classes: `has_permission()`, `has_object_permission()`
- Combining permissions with `AND` (`&`) and `OR` (`|`) operators (DRF 3.9+)
- Object-level permissions: when to use `get_object()` vs. queryset filtering

---

### 3.6 Throttling

- `AnonRateThrottle`, `UserRateThrottle`, `ScopedRateThrottle`
- Rate configuration in settings: `DEFAULT_THROTTLE_RATES`
- Custom throttle classes
- Using Redis as the throttle backend for distributed systems
- `429 Too Many Requests` response

---

### 3.7 Filtering, Searching & Ordering

- `django-filter`: `DjangoFilterBackend`, `filterset_fields`, custom `FilterSet`
- `SearchFilter`: `search_fields`, `^` (starts with), `=` (exact), `@` (full-text), `$` (regex)
- `OrderingFilter`: `ordering_fields`, `ordering` default
- Combining multiple backends
- Custom filter backends

---

### 3.8 Pagination

- `PageNumberPagination`: `page`, `page_size`, `max_page_size`
- `LimitOffsetPagination`: `limit`, `offset`
- `CursorPagination`: for real-time data feeds, no skipping pages
- Custom pagination classes: overriding `get_paginated_response()`
- When to use which: cursor for infinite scroll, page number for traditional UI

---

### 3.9 Exception Handling

- DRF's default exception handler: `EXCEPTION_HANDLER` setting
- Custom exception handler: standardizing error response format
- `APIException` and its subclasses: `ValidationError`, `AuthenticationFailed`, `NotAuthenticated`, `PermissionDenied`, `NotFound`, `MethodNotAllowed`, `Throttled`
- Consistent error response schema for client projects

---

### 3.10 Content Negotiation & Renderers/Parsers

- `JSONRenderer`, `BrowsableAPIRenderer`
- `JSONParser`, `MultiPartParser`, `FileUploadParser`
- Content type negotiation via `Accept` and `Content-Type` headers
- Disabling `BrowsableAPIRenderer` in production

---

### 3.11 Versioning

- `URLPathVersioning`: `/api/v1/`, `/api/v2/`
- `NamespaceVersioning`, `QueryParameterVersioning`, `HeaderVersioning`
- Accessing `request.version` in views
- Routing different serializers/logic based on version

---

### 3.12 Celery & Celery Beat (Background Tasks)

A must in client-facing production systems. Every Django project of scale uses this.

#### Why Celery?
- Offload long-running tasks (email, PDF generation, 3rd-party API calls, report generation)
- Don't block the HTTP response — return `202 Accepted` immediately
- Scale workers independently of web servers

#### Celery Core Concepts
- **Broker**: the message queue — Redis or RabbitMQ
- **Worker**: the process that consumes and executes tasks
- **Backend (Result Store)**: stores task results — Redis, Django DB
- **Task**: a Python function decorated with `@shared_task` or `@app.task`

#### Celery Setup with Django
```
pip install celery redis django-celery-results django-celery-beat
```
- `celery.py` — app configuration file in Django project root
- `CELERY_BROKER_URL`, `CELERY_RESULT_BACKEND` in settings
- `autodiscover_tasks()` — finds `tasks.py` in all `INSTALLED_APPS`
- `@shared_task` vs `@app.task` — prefer `@shared_task` for reusable apps

#### Task Fundamentals
- `task.delay()` — shortcut for `apply_async` with positional args
- `task.apply_async()` — full control: `args`, `kwargs`, `countdown`, `eta`, `expires`, `queue`
- `task.si()` — immutable signature (ignores parent result in chain)
- Task retries: `self.retry(exc=exc, countdown=60, max_retries=3)`
- `autoretry_for`, `max_retries`, `retry_backoff`, `retry_jitter`
- Task states: `PENDING`, `STARTED`, `SUCCESS`, `FAILURE`, `RETRY`, `REVOKED`
- `AsyncResult`: checking task status, retrieving results
- Idempotency — tasks must be safe to retry

#### Canvas — Workflow Primitives
- `chain`: sequential task execution — output of one is input of next
- `group`: parallel execution of tasks
- `chord`: group + callback (runs after all group tasks complete)
- `chunks`: split large iterables into batches

#### Celery Beat (Periodic Tasks)
- `django-celery-beat` — stores schedules in the Django database
- `CELERY_BEAT_SCHEDULE` in settings for static schedules
- Beat scheduler: `celerybeat` process
- Schedule types: `crontab()`, `timedelta`, `solar`
- Dynamic periodic tasks: adding/removing via Django Admin or code
- **Important**: only one Beat process should run at a time

#### Queues & Routing
- Named queues: `default`, `high_priority`, `emails`, `reports`
- Task routing: `CELERY_TASK_ROUTES`
- Worker consumption: `celery -A proj worker -Q high_priority,default`
- Priority queues with Redis

#### Monitoring & Observability
- `Flower`: real-time web monitor for Celery workers and tasks
- Celery events: `celery -A proj events`
- Dead Letter Queue patterns for failed tasks
- Alerting on `FAILURE` state via signals: `task_failure` signal

#### Production Best Practices
- Use `acks_late=True` for at-least-once delivery guarantees
- Avoid passing Django model instances in task args — pass PKs instead
- Set task time limits: `soft_time_limit`, `time_limit`
- Keep tasks small and composable
- Use `transaction.on_commit()` to trigger tasks after DB commit (avoid race condition)

---

### 3.13 ORM in DRF Context

- Always define `get_queryset()` in views — never access `Model.objects.all()` directly
- Optimize for API performance: `select_related`, `prefetch_related`
- Use `.only()` and `.defer()` to limit columns fetched
- `Prefetch` objects with custom querysets for nested serializer optimization
- Avoid database queries in serializers — annotate in the view's queryset
- `transaction.atomic()` in `serializer.create()` for multi-model operations
- Database-level constraints in `Meta.constraints` as a last line of defense

---

### 3.14 API Design Principles

These are what separate juniors from professionals in client-serving roles.

- RESTful resource naming: nouns, not verbs (`/users/`, not `/getUsers/`)
- HTTP method semantics: `GET` (safe, idempotent), `POST` (non-idempotent), `PUT` (full replace, idempotent), `PATCH` (partial update, idempotent), `DELETE` (idempotent)
- Status codes you must know:
  - `200 OK`, `201 Created`, `204 No Content`
  - `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`
  - `422 Unprocessable Entity`, `429 Too Many Requests`
  - `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`
- Consistent error response format across all endpoints
- Versioning strategy from day one
- Pagination on all list endpoints — never return unbounded lists to clients
- API documentation: `drf-spectacular` (OpenAPI 3.0), `Swagger UI`, `ReDoc`
- Idempotency keys for payment/critical endpoints

---

## PART 4 — DEPLOYMENT, TOOLING & PRODUCTION

---

### 4.1 WSGI & ASGI

- **WSGI**: synchronous, `gunicorn`, `uWSGI` — traditional Django deployment
- **ASGI**: asynchronous, `uvicorn`, `daphne` — for Django Channels, async views
- `gunicorn` worker types: `sync`, `gevent`, `eventlet`, `uvicorn.workers.UvicornWorker`
- Worker count heuristics: `(2 * CPU cores) + 1`

---

### 4.2 Environment & Configuration Management

- Never hardcode secrets — use environment variables
- `django-environ` or `python-decouple`
- `.env` file patterns and `.gitignore`
- Separate settings files: `base.py`, `development.py`, `production.py`, `testing.py`

---

### 4.3 Database

- Connection pooling: `django-db-geventpool`, `pgBouncer`
- Read replicas: `DATABASE_ROUTERS`
- Django's `CONN_MAX_AGE` setting
- Indexing strategy: when indexes help, when they hurt (write overhead)
- `EXPLAIN ANALYZE` — reading query plans

---

### 4.4 Redis in the Django Stack

- Session storage
- Cache backend
- Celery broker and result backend
- Rate limiting backend (throttling)
- Django Channels layer

---

### 4.5 Logging & Monitoring

- Django's `LOGGING` configuration: handlers, formatters, loggers, levels
- Structured logging (JSON) for log aggregation systems
- `Sentry` for error tracking — `sentry-sdk` Django integration
- Request/response logging middleware
- Health check endpoints

---

### 4.6 Docker & DevOps Awareness

- `Dockerfile` for a Django application
- `docker-compose.yml` with web, db, redis, celery worker, celery beat services
- `ENTRYPOINT` vs `CMD`
- Health checks for containers
- Environment variables via `docker-compose.env`

---

## QUICK REFERENCE: Topics by Evaluation Weight

| Weight | Topic |
|---|---|
| 🔴 Must-Know | Python OOP, Decorators, Django ORM, DRF Serializers, DRF Views/ViewSets, JWT Auth, Permissions |
| 🔴 Must-Know | Celery tasks, Celery Beat, N+1 queries, `select_related`/`prefetch_related` |
| 🟠 High Priority | Memory management, GIL, Threading vs Async, Generators, Custom Managers |
| 🟠 High Priority | DRF Filtering/Pagination, Custom Exception Handler, API Design principles |
| 🟡 Good to Know | Django Signals, Caching, Middleware, Migrations, Versioning |
| 🟡 Good to Know | Celery Canvas (chain/group/chord), Flower, Redis routing |
| 🟢 Awareness | ASGI/Channels, Docker, DB connection pooling, Sentry integration |

---

> *"You don't need to memorize every parameter. You need to understand why things work the way they do — that's what I'm actually testing."*
>
> — Manager, Python Department

---

*Last updated: May 2026 | Python 3.11+ | Django 4.2 LTS / 5.x | DRF 3.15 | Celery 5.x*
