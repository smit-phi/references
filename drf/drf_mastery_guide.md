# Django REST Framework — A Comprehensive Mastery Guide

> **Who this is for:** Django developers who want to deeply understand *why* DRF exists, not just *how* to use it. Every component is explained first by the problem it solves, then by how it fits into the REST architecture, and finally by its mechanics.

---

## Table of Contents

1. [Why REST? The Architecture Before the Framework](#1-why-rest-the-architecture-before-the-framework)
2. [Why DRF? The Problem with Raw Django for APIs](#2-why-drf-the-problem-with-raw-django-for-apis)
3. [The DRF Request & Response Layer](#3-the-drf-request--response-layer)
4. [Serializers — The Heart of DRF](#4-serializers--the-heart-of-drf)
5. [Views — From Function-Based to Class-Based](#5-views--from-function-based-to-class-based)
6. [Generic Views & Mixins](#6-generic-views--mixins)
7. [ViewSets & Routers](#7-viewsets--routers)
8. [Authentication](#8-authentication)
9. [Permissions](#9-permissions)
10. [Throttling](#10-throttling)
11. [Filtering, Searching & Ordering](#11-filtering-searching--ordering)
12. [Pagination](#12-pagination)
13. [Parsers & Renderers](#13-parsers--renderers)
14. [Content Negotiation](#14-content-negotiation)
15. [Versioning](#15-versioning)
16. [Routers — URL Routing at Scale](#16-routers--url-routing-at-scale)
17. [The Browsable API](#17-the-browsable-api)
18. [Settings & Configuration](#18-settings--configuration)
19. [Putting It All Together — The DRF Request Lifecycle](#19-putting-it-all-together--the-drf-request-lifecycle)
20. [Mental Model Cheat Sheet](#20-mental-model-cheat-sheet)

---

## 1. Why REST? The Architecture Before the Framework

Before writing a single line of DRF code, you need to understand what you're building *toward*. REST is not a protocol or a library — it's an **architectural style** for distributed hypermedia systems, described by Roy Fielding in his 2000 dissertation.

### The 6 Constraints of REST

REST is defined by six constraints. When you respect all of them, your system is called *RESTful*.

**1. Client–Server Separation**
The UI (client) and the data store (server) are decoupled. The client doesn't care how data is stored. The server doesn't care how data is displayed. This lets them evolve independently. Your Django backend and your React frontend can be built, deployed, and scaled completely independently.

**2. Statelessness**
Every request from a client must contain *all the information* the server needs to process it. The server stores no session state between requests. This is why REST APIs use tokens (JWT, API keys) rather than server-side sessions — the auth context travels *with* the request.

**3. Cacheability**
Responses must declare whether they are cacheable. This allows clients and intermediaries (CDNs, proxies) to cache responses, drastically reducing server load. DRF integrates with Django's cache framework for this.

**4. Uniform Interface**
This is the central feature of REST and has four sub-constraints:
- **Resource identification in requests** — resources are identified by URIs (e.g., `/api/articles/42/`).
- **Resource manipulation through representations** — clients manipulate resources via the representation they receive (JSON/XML), not direct database access.
- **Self-descriptive messages** — each message includes enough information to describe how to process it (Content-Type headers, HTTP methods).
- **HATEOAS** (Hypermedia as the Engine of Application State) — responses include links to related actions. DRF's HyperlinkedModelSerializer implements this.

**5. Layered System**
The client doesn't know if it's talking directly to the server or to an intermediary (load balancer, CDN, gateway). Layers can be added without the client or server knowing.

**6. Code on Demand (Optional)**
Servers can extend client functionality by sending executable code (e.g., JavaScript). This is rarely used in API contexts.

### HTTP Methods as REST Verbs

REST maps CRUD operations to HTTP methods in a standardized way:

| HTTP Method | CRUD Equivalent | Safe? | Idempotent? | Typical Use |
|-------------|-----------------|-------|-------------|-------------|
| GET | Read | ✅ Yes | ✅ Yes | Retrieve a resource or list |
| POST | Create | ❌ No | ❌ No | Create a new resource |
| PUT | Replace | ❌ No | ✅ Yes | Replace a resource entirely |
| PATCH | Update | ❌ No | ✅ Yes | Partially update a resource |
| DELETE | Delete | ❌ No | ✅ Yes | Delete a resource |
| HEAD | Read (headers) | ✅ Yes | ✅ Yes | Check existence without body |
| OPTIONS | Metadata | ✅ Yes | ✅ Yes | Discover allowed methods |

**Safe** means the request does not modify server state. **Idempotent** means calling it multiple times produces the same result as calling it once.

### Resources and URLs

In REST, everything is a **resource** — a conceptual entity. A resource has:
- An **identifier** (its URI): `/api/articles/42/`
- A **representation** (what's returned): JSON, XML, HTML
- **Actions** (performed via HTTP methods)

Good URL design follows these conventions:
- Use nouns, not verbs: `/api/articles/` not `/api/getArticles/`
- Use plurals for collections: `/api/articles/` for a list, `/api/articles/42/` for a specific one
- Nest related resources: `/api/articles/42/comments/`
- Use query params for filtering, not path segments: `/api/articles/?author=john`

---

## 2. Why DRF? The Problem with Raw Django for APIs

You already know Django. So why not just use `JsonResponse` and write API views manually? Let's look at what that actually means.

### What Raw Django Gives You

```python
# The "naive" way — writing an API without DRF
import json
from django.http import JsonResponse, HttpResponseNotAllowed
from django.views.decorators.csrf import csrf_exempt
from .models import Article

@csrf_exempt
def article_list(request):
    if request.method == 'GET':
        articles = Article.objects.all()
        data = list(articles.values('id', 'title', 'content', 'author', 'created_at'))
        return JsonResponse(data, safe=False)

    elif request.method == 'POST':
        try:
            body = json.loads(request.body)
        except json.JSONDecodeError:
            return JsonResponse({'error': 'Invalid JSON'}, status=400)

        # Manual validation — every field, every time
        if not body.get('title'):
            return JsonResponse({'error': 'title is required'}, status=400)
        if not body.get('content'):
            return JsonResponse({'error': 'content is required'}, status=400)

        article = Article.objects.create(
            title=body['title'],
            content=body['content'],
            author=request.user  # But wait — is user authenticated?
        )
        return JsonResponse({'id': article.id, 'title': article.title}, status=201)

    return HttpResponseNotAllowed(['GET', 'POST'])
```

This is just for *two* methods on *one* endpoint. Problems compound rapidly:

- **Serialization is manual** — `values()` doesn't handle nested relations, custom fields, or transformations.
- **Deserialization is manual** — parsing, type-coercion, validation all written by hand.
- **Error handling is inconsistent** — every developer writes their own error format.
- **No standard authentication layer** — you check `request.user` everywhere, differently.
- **No automatic OPTIONS/HEAD support** — clients can't discover your API.
- **No content negotiation** — you can't serve XML to one client and JSON to another without if/else spaghetti.
- **No browsable API** — debugging requires curl or Postman for every change.
- **Pagination written from scratch** — every list endpoint.
- **Filtering written from scratch** — every list endpoint.
- **Throttling written from scratch** — if you even remember to add it.

### What DRF Solves

DRF provides a **consistent, composable, battle-tested layer** on top of Django that handles all of the above. Its architecture is built around the principle that API concerns are orthogonal to business logic — and they should be handled separately, at the right layer.

Here is the DRF component stack, from the most foundational to the most abstract:

```
URL Router
    └── ViewSet / View
            ├── Authentication (who are you?)
            ├── Permissions (are you allowed?)
            ├── Throttling (how often can you do this?)
            ├── Content Negotiation (what format do you want?)
            ├── Parser (read the incoming data)
            ├── Serializer (validate and transform data)
            └── Renderer (format the outgoing response)
```

Each layer has one job. Each is swappable. Each is testable independently.

---

## 3. The DRF Request & Response Layer

### The Problem

Django's `HttpRequest` object was built for HTML forms and browser-based web applications. It knows how to parse `application/x-www-form-urlencoded` and `multipart/form-data`. But a modern API receives `application/json`, `application/xml`, `multipart/form-data`, and more. Django doesn't handle this uniformly.

Similarly, Django's `HttpResponse` and `JsonResponse` are blunt instruments. A proper API response needs to:
- Carry a negotiated content type
- Serialize arbitrary Python data structures
- Support status codes with semantic meaning
- Be format-agnostic (JSON today, XML tomorrow)

### The Solution: `Request` and `Response`

DRF wraps Django's `HttpRequest` in its own `Request` class and provides a `Response` class that works with renderers.

#### `rest_framework.request.Request`

```python
# DRF's Request wraps Django's HttpRequest
# Access it as `request` in any DRF view — it IS a DRF Request, not a Django one

request.data          # Parsed request body — works for JSON, form data, multipart, etc.
request.query_params  # Same as request.GET — clearer, more semantic name
request.user          # Authenticated user (set by authentication classes)
request.auth          # Authentication token/credential
request.accepted_renderer   # The renderer chosen by content negotiation
request.accepted_media_type # The MIME type chosen
```

The key insight: **`request.data`** is universal. Whether the client sends `{"title": "Hello"}` as JSON or `title=Hello` as a form POST, `request.data` gives you the same Python dict. This is parsed by DRF's **Parser** classes (more on those later).

Compare:
```python
# Django — you have to handle each content type manually
if request.content_type == 'application/json':
    data = json.loads(request.body)
elif request.content_type == 'application/x-www-form-urlencoded':
    data = request.POST

# DRF — always the same
data = request.data
```

#### `rest_framework.response.Response`

```python
from rest_framework.response import Response
from rest_framework import status

# Don't serialize manually — pass Python data, let the renderer handle it
return Response(
    data={'id': 42, 'title': 'Hello'},
    status=status.HTTP_201_CREATED,
    headers={'Location': '/api/articles/42/'}
)
```

`Response` doesn't commit to JSON. It stores your data as Python and lets the **Renderer** (chosen through content negotiation) serialize it at the last moment. This means the same view can return JSON to a JavaScript client and HTML to a browser — with zero extra code.

Note: `status` is a module of semantic constants (`HTTP_200_OK`, `HTTP_404_NOT_FOUND`, etc.) so you're not writing magic numbers everywhere.

---

## 4. Serializers — The Heart of DRF

### The Problem

Your data lives in Django models — Python objects with ORM relationships, custom methods, and complex types. Your API speaks JSON — flat key-value structures with strings, numbers, arrays, and objects.

Moving between these two worlds involves:
1. **Serialization**: Python objects → JSON-compatible dicts (for output)
2. **Deserialization**: JSON dicts → validated Python data → model instances (for input)
3. **Validation**: Enforcing data integrity at the API boundary

In raw Django, you write this logic by hand, everywhere, inconsistently. DRF's **Serializer** is the single, reusable component that handles all three.

### `Serializer` — The Base

```python
from rest_framework import serializers

class ArticleSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    published = serializers.BooleanField(default=False)
    created_at = serializers.DateTimeField(read_only=True)

    def create(self, validated_data):
        return Article.objects.create(**validated_data)

    def update(self, instance, validated_data):
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.save()
        return instance
```

**Serializing (output):**
```python
article = Article.objects.get(pk=42)
serializer = ArticleSerializer(article)
serializer.data  # {'id': 42, 'title': '...', ...}  — a Python dict, ready for Response

# Multiple objects
articles = Article.objects.all()
serializer = ArticleSerializer(articles, many=True)
serializer.data  # A list of dicts
```

**Deserializing and Validating (input):**
```python
data = {'title': 'New Article', 'content': 'Hello world'}
serializer = ArticleSerializer(data=data)

if serializer.is_valid():
    article = serializer.save()   # calls create() or update()
else:
    serializer.errors  # {'title': ['This field is required.']}
```

**The `save()` decision:** If you pass an `instance`, `save()` calls `update()`. If you don't, it calls `create()`. This is how a single serializer handles both POST and PUT/PATCH.

### Validation in Depth

DRF has three levels of validation:

**Field-level validation** — built into field types:
```python
title = serializers.CharField(max_length=200, min_length=5)
email = serializers.EmailField()
url = serializers.URLField()
```

**Field-level custom validation** — `validate_<fieldname>`:
```python
def validate_title(self, value):
    if 'django' not in value.lower():
        raise serializers.ValidationError("Must be about Django!")
    return value
```

**Object-level validation** — `validate()` (cross-field logic):
```python
def validate(self, data):
    if data['start_date'] > data['end_date']:
        raise serializers.ValidationError("Start must be before end.")
    return data
```

### `ModelSerializer` — The Productivity Accelerator

Writing a `Serializer` that mirrors your model field-by-field is tedious. `ModelSerializer` introspects your model and generates fields automatically.

```python
class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'author', 'published', 'created_at']
        read_only_fields = ['id', 'created_at']
```

`ModelSerializer` also auto-generates `create()` and `update()` for you. It maps Django model fields to DRF serializer fields automatically:

| Django Model Field | DRF Serializer Field |
|-------------------|----------------------|
| `CharField` | `CharField` |
| `IntegerField` | `IntegerField` |
| `DateTimeField` | `DateTimeField` |
| `BooleanField` | `BooleanField` |
| `ForeignKey` | `PrimaryKeyRelatedField` |
| `ManyToManyField` | `PrimaryKeyRelatedField(many=True)` |

### Nested Serializers and Relationships

This is where serializers get powerful — and where many developers first hit complexity.

```python
class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email']

class ArticleSerializer(serializers.ModelSerializer):
    # Instead of returning author's pk, return the full nested object
    author = AuthorSerializer(read_only=True)

    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'author']
```

Result:
```json
{
  "id": 42,
  "title": "Hello",
  "content": "...",
  "author": {
    "id": 1,
    "username": "john",
    "email": "john@example.com"
  }
}
```

**The write-back problem:** Nested serializers are `read_only=True` by default because writing nested data requires custom `create()`/`update()` logic. This is intentional — DRF doesn't assume how you want to handle nested writes.

### `HyperlinkedModelSerializer`

Instead of returning primary keys for relationships, returns full URLs — implementing HATEOAS:

```python
class ArticleSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Article
        fields = ['url', 'title', 'content', 'author']
        # url is automatically generated from the router
```

Result:
```json
{
  "url": "http://api.example.com/articles/42/",
  "title": "Hello",
  "author": "http://api.example.com/users/1/"
}
```

### `SerializerMethodField` — Computed Fields

For fields that don't exist on the model but need to be in the output:

```python
class ArticleSerializer(serializers.ModelSerializer):
    word_count = serializers.SerializerMethodField()
    is_owner = serializers.SerializerMethodField()

    def get_word_count(self, obj):
        return len(obj.content.split())

    def get_is_owner(self, obj):
        request = self.context.get('request')
        return obj.author == request.user

    class Meta:
        model = Article
        fields = ['id', 'title', 'word_count', 'is_owner']
```

**The `context` parameter** is how you pass extra information (like the current `request`) into a serializer. DRF generic views pass the request automatically via `get_serializer_context()`.

---

## 5. Views — From Function-Based to Class-Based

DRF's view layer is a hierarchy that progressively handles more concerns for you. Understanding each level shows you exactly what DRF is doing under the hood.

### `@api_view` — Function-Based Views

The simplest entry point. A decorator that turns a regular function into a DRF view.

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status

@api_view(['GET', 'POST'])
def article_list(request):
    if request.method == 'GET':
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

What `@api_view` does for you:
- Wraps Django's `HttpRequest` in DRF's `Request`
- Runs authentication, permissions, throttling
- Handles content negotiation and rendering
- Returns a `405 Method Not Allowed` for unlisted methods
- Powers the Browsable API

### `APIView` — Class-Based Views

The direct class-based equivalent. Each HTTP method becomes a method on the class:

```python
from rest_framework.views import APIView

class ArticleList(APIView):
    def get(self, request):
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class ArticleDetail(APIView):
    def get_object(self, pk):
        try:
            return Article.objects.get(pk=pk)
        except Article.DoesNotExist:
            raise Http404

    def get(self, request, pk):
        article = self.get_object(pk)
        serializer = ArticleSerializer(article)
        return Response(serializer.data)

    def put(self, request, pk):
        article = self.get_object(pk)
        serializer = ArticleSerializer(article, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk):
        article = self.get_object(pk)
        article.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**`APIView` is the foundation of everything.** All higher-level DRF views extend it. It handles:
- Authentication, permissions, throttling (via `initial()`)
- Content negotiation
- Parsing and rendering setup
- Exception handling via `handle_exception()`

Notice the pattern in `ArticleDetail`: you write `get_object()`, `get()`, `put()`, `delete()`. You'll write this same pattern dozens of times. That's exactly the repetition that generic views are designed to eliminate.

---

## 6. Generic Views & Mixins

### The Problem

Every single resource in your API needs the same basic operations: list, create, retrieve, update, destroy. The code is structurally identical — only the model, serializer, and queryset change. Writing this from scratch 20 times is error-prone and unmaintainable.

### Mixins — Single Behaviors

Mixins are small classes that each provide one standard action. They don't know about URLs or models — they're given that context by the view.

```python
from rest_framework.mixins import (
    ListModelMixin,
    CreateModelMixin,
    RetrieveModelMixin,
    UpdateModelMixin,
    DestroyModelMixin
)
```

| Mixin | Method | HTTP Action |
|-------|--------|-------------|
| `ListModelMixin` | `list()` | GET → list |
| `CreateModelMixin` | `create()` | POST → create |
| `RetrieveModelMixin` | `retrieve()` | GET → detail |
| `UpdateModelMixin` | `update()` | PUT/PATCH → update |
| `DestroyModelMixin` | `destroy()` | DELETE → delete |

Each mixin is designed to be composed. Want a read-only endpoint? Mix in List + Retrieve but not Create/Update/Destroy.

### `GenericAPIView` — The Base with Context

`GenericAPIView` extends `APIView` with awareness of queryset, serializer, and lookup:

```python
from rest_framework.generics import GenericAPIView

class ArticleList(ListModelMixin, CreateModelMixin, GenericAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

Key attributes on `GenericAPIView`:
- `queryset` — what objects this view works with
- `serializer_class` — how to serialize/deserialize them
- `lookup_field` — what URL kwarg identifies a single object (default: `pk`)
- `lookup_url_kwarg` — override the URL kwarg name
- `pagination_class` — which paginator to use
- `filter_backends` — list of filter classes

Key methods you can override:
- `get_queryset()` — dynamic queryset (e.g., filter by current user)
- `get_serializer_class()` — choose serializer based on request
- `get_object()` — retrieve a single object; calls `check_object_permissions()`
- `get_serializer()` — instantiate the serializer with context

### Concrete Generic Views — Batteries Included

DRF ships with pre-composed views for every common combination:

```python
from rest_framework.generics import (
    ListAPIView,           # GET list
    CreateAPIView,         # POST
    RetrieveAPIView,       # GET detail
    UpdateAPIView,         # PUT + PATCH
    DestroyAPIView,        # DELETE
    ListCreateAPIView,     # GET list + POST
    RetrieveUpdateAPIView, # GET detail + PUT + PATCH
    RetrieveDestroyAPIView,# GET detail + DELETE
    RetrieveUpdateDestroyAPIView,  # GET + PUT + PATCH + DELETE
)
```

In practice, most resources need just two endpoints:
- `ListCreateAPIView` — `/api/articles/`
- `RetrieveUpdateDestroyAPIView` — `/api/articles/<pk>/`

```python
class ArticleList(generics.ListCreateAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticated]

class ArticleDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
```

That's the full CRUD API for articles — 10 lines. Compare to the 50+ lines of raw Django.

### Dynamic Queryset and Serializer

Override `get_queryset()` to filter based on the request:

```python
class UserArticleList(generics.ListAPIView):
    serializer_class = ArticleSerializer

    def get_queryset(self):
        # Only return articles belonging to the current user
        return Article.objects.filter(author=self.request.user)
```

Override `get_serializer_class()` to use different serializers for different situations:

```python
class ArticleDetail(generics.RetrieveUpdateAPIView):
    queryset = Article.objects.all()

    def get_serializer_class(self):
        if self.request.method in ('PUT', 'PATCH'):
            return ArticleWriteSerializer  # Only writable fields
        return ArticleReadSerializer  # Full nested representation
```

---

## 7. ViewSets & Routers

### The Problem

Your `/api/articles/` and `/api/articles/<pk>/` endpoints share the same model, queryset, and serializer. Yet they're two separate view classes, wired up manually in urls.py. For a 20-resource API, that's 40 classes and 40+ URL patterns. This doesn't scale.

### ViewSets — Views as a Unit

A `ViewSet` groups all the related actions for a resource into a single class. Instead of methods named after HTTP verbs (`get`, `post`), you define methods named after *actions* (`list`, `create`, `retrieve`, `update`, `partial_update`, `destroy`).

```python
from rest_framework import viewsets

class ArticleViewSet(viewsets.ViewSet):
    def list(self, request):
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)

    def create(self, request):
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def retrieve(self, request, pk=None):
        article = get_object_or_404(Article, pk=pk)
        serializer = ArticleSerializer(article)
        return Response(serializer.data)
```

### `ModelViewSet` — The Ultimate Shortcut

Combine `ViewSet` with all 5 mixins and you get `ModelViewSet`:

```python
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
```

This single class provides: `list`, `create`, `retrieve`, `update`, `partial_update`, `destroy` — the full CRUD API.

For read-only resources, use `ReadOnlyModelViewSet` (provides only `list` and `retrieve`).

### Custom Actions with `@action`

Beyond CRUD, you often need custom endpoints. The `@action` decorator adds them:

```python
from rest_framework.decorators import action

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer

    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """POST /api/articles/42/publish/"""
        article = self.get_object()
        article.published = True
        article.save()
        return Response({'status': 'published'})

    @action(detail=False, methods=['get'])
    def featured(self, request):
        """GET /api/articles/featured/"""
        featured = Article.objects.filter(featured=True)
        serializer = self.get_serializer(featured, many=True)
        return Response(serializer.data)
```

- `detail=True` — action on a specific object (`/articles/{pk}/publish/`)
- `detail=False` — action on the collection (`/articles/featured/`)

---

## 8. Authentication

### The Problem

"Who is making this request?" is the most fundamental security question. Your API needs a consistent, pluggable way to identify the caller before any other check happens.

Authentication answers: **who are you?** (Identity)

It's separate from permissions (what you can do) and throttling (how often you can do it).

### How DRF Authentication Works

DRF authentication is a **list of classes** tried in order. The first one to successfully authenticate the request wins. If none authenticate, `request.user` is `AnonymousUser` and `request.auth` is `None`.

Crucially: **authentication failure does not automatically deny the request.** That's the job of permissions. You can have a view that accepts both authenticated and anonymous users — authentication just populates `request.user` with the right identity.

### Built-in Authentication Classes

**`BasicAuthentication`**
Reads `Authorization: Basic <base64(username:password)>` from the request header. Only use this over HTTPS. Good for testing, rarely appropriate for production APIs.

```python
from rest_framework.authentication import BasicAuthentication
```

**`SessionAuthentication`**
Uses Django's session framework. Works for same-origin browser clients. Requires CSRF tokens for non-safe methods (POST, PUT, etc.). Good for APIs consumed by your own frontend served from the same domain.

```python
from rest_framework.authentication import SessionAuthentication
```

**`TokenAuthentication`**
Each user gets a database-stored token. The client sends `Authorization: Token <token_value>`. This is DRF's simplest stateless authentication.

```python
# settings.py
INSTALLED_APPS = ['rest_framework.authtoken']

# views.py
from rest_framework.authentication import TokenAuthentication
from rest_framework.authtoken.models import Token

# Create a token for a user
token = Token.objects.create(user=user)
```

```bash
# Client usage
curl -H "Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b" http://api/articles/
```

**`RemoteUserAuthentication`**
Delegates authentication to a web server (nginx, Apache). Useful in enterprise environments with existing auth infrastructure.

### Third-Party: `djangorestframework-simplejwt`

For most modern APIs, you want **JWT (JSON Web Tokens)**. JWTs are self-contained — the server doesn't need to look up a database token. The token carries the user identity, expiry, and any custom claims, all cryptographically signed.

```python
# Install: pip install djangorestframework-simplejwt
from rest_framework_simplejwt.authentication import JWTAuthentication
```

JWT flow:
1. Client sends credentials → server returns `access` token (short-lived) + `refresh` token (long-lived)
2. Client sends `Authorization: Bearer <access_token>` with each request
3. When access token expires, client uses refresh token to get a new one
4. No database lookup needed to validate — just signature verification

### Configuration

Set globally in `settings.py`:

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ]
}
```

Or override per view:

```python
class ArticleList(generics.ListAPIView):
    authentication_classes = [TokenAuthentication]
    # ...
```

---

## 9. Permissions

### The Problem

Authentication tells you *who* is making the request. Permissions answer: *should they be allowed to?*

The key insight is that authentication and permissions are separate concerns:
- A request can be authenticated but still denied (a user without admin rights trying to delete someone else's post)
- A request can be unauthenticated and still allowed (a public GET endpoint)

### How DRF Permissions Work

DRF checks permissions at two points:

1. **Before dispatch** — `check_permissions()` on `request` + `view`. Runs for every request.
2. **On object access** — `check_object_permissions()` when `get_object()` is called. Runs only for detail views.

If any permission class returns `False`, DRF raises:
- `HTTP 403 Forbidden` if the user is authenticated
- `HTTP 401 Unauthorized` if they are not

### Built-in Permission Classes

**`AllowAny`**
No restrictions. Any request is allowed. The default if you set `DEFAULT_PERMISSION_CLASSES = []`.

**`IsAuthenticated`**
Request must come from any authenticated user. If `request.user.is_authenticated` is False, deny.

**`IsAdminUser`**
Request must come from a user with `is_staff = True`.

**`IsAuthenticatedOrReadOnly`**
The most common pattern for public APIs: anyone can GET, but only authenticated users can POST/PUT/DELETE.

```python
# Anonymous user can list and retrieve articles
# Only authenticated users can create/update/delete
class ArticleViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticatedOrReadOnly]
```

**`DjangoModelPermissions`**
Maps HTTP methods to Django's model-level permissions system (`add_`, `change_`, `delete_`, `view_`). Good for admin-oriented APIs.

**`DjangoObjectPermissions`**
Extends model permissions to object-level, integrating with `django-guardian`.

### Custom Permissions

This is where you implement your own business logic:

```python
from rest_framework.permissions import BasePermission, SAFE_METHODS

class IsOwnerOrReadOnly(BasePermission):
    """
    Read access to everyone.
    Write access only to the object's owner.
    """
    def has_object_permission(self, request, view, obj):
        # SAFE_METHODS = ('GET', 'HEAD', 'OPTIONS')
        if request.method in SAFE_METHODS:
            return True
        return obj.author == request.user
```

There are two methods to implement:

- `has_permission(request, view)` — runs before any object is fetched. Use for resource-level checks.
- `has_object_permission(request, view, obj)` — runs when `get_object()` is called. Use for ownership checks.

Important: `has_object_permission` is only called if `has_permission` returned `True`. Always implement both if needed.

### Composing Permissions

In DRF, permission classes in a list are AND-ed together — all must pass:

```python
permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
```

For OR logic, you need to override `has_permission` or use third-party packages like `drf-access-policy`.

---

## 10. Throttling

### The Problem

Even authorized users shouldn't be able to make unlimited requests. Without throttling, a single client could hammer your API, degrading service for everyone, or abuse endpoints to scrape data or brute-force passwords.

Throttling answers: **how often can you do this?**

### How DRF Throttling Works

Each throttle class implements `allow_request(request, view) -> bool`. DRF checks all configured throttles; if any returns `False`, it returns `HTTP 429 Too Many Requests` with a `Retry-After` header.

DRF's built-in throttles use a cache (default: Django's cache framework) to track request rates per identity.

### Built-in Throttle Classes

**`AnonRateThrottle`** — rate-limits by IP address for unauthenticated requests
**`UserRateThrottle`** — rate-limits per authenticated user
**`ScopedRateThrottle`** — different rates for different parts of your API

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',   # 100 requests per day for anonymous users
        'user': '1000/day',  # 1000 per day for authenticated users
    }
}
```

Rate format: `<number>/<period>` where period is `second`, `minute`, `hour`, or `day`.

**Scoped throttling** for specific endpoints:

```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'login': '5/minute',
        'uploads': '20/hour',
    }
}

class LoginView(APIView):
    throttle_scope = 'login'  # Uses the 'login' rate

class FileUploadView(APIView):
    throttle_scope = 'uploads'  # Uses the 'uploads' rate
```

---

## 11. Filtering, Searching & Ordering

### The Problem

A list endpoint returning 100,000 records to the client who wanted 10 is both wasteful and unusable. The client needs to be able to say: "give me articles by author X, published after date Y, ordered by views descending."

### Filter Backends

DRF uses **filter backend classes** — each one is responsible for a different filtering mechanism.

**`DjangoFilterBackend`** (from `django-filter` package)

Provides exact-match and relational filtering via URL query parameters:

```bash
pip install django-filter
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ['django_filters.rest_framework.DjangoFilterBackend']
}

# views.py
class ArticleList(generics.ListAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    filterset_fields = ['author', 'published', 'category']
```

Now clients can: `GET /api/articles/?author=1&published=true`

For complex filtering, define a `FilterSet`:

```python
import django_filters

class ArticleFilter(django_filters.FilterSet):
    min_date = django_filters.DateFilter(field_name='created_at', lookup_expr='gte')
    max_date = django_filters.DateFilter(field_name='created_at', lookup_expr='lte')
    title_contains = django_filters.CharFilter(field_name='title', lookup_expr='icontains')

    class Meta:
        model = Article
        fields = ['author', 'published', 'min_date', 'max_date', 'title_contains']
```

**`SearchFilter`**

Full-text search across specified fields:

```python
from rest_framework.filters import SearchFilter

class ArticleList(generics.ListAPIView):
    filter_backends = [SearchFilter]
    search_fields = ['title', 'content', 'author__username']
```

Clients use: `GET /api/articles/?search=django`

Search lookups:
- `^` starts-with: `search_fields = ['^title']`
- `=` exact: `search_fields = ['=username']`
- `@` full-text (MySQL): `search_fields = ['@content']`
- `$` regex: `search_fields = ['$title']`

**`OrderingFilter`**

Allows clients to choose sort order:

```python
from rest_framework.filters import OrderingFilter

class ArticleList(generics.ListAPIView):
    filter_backends = [OrderingFilter]
    ordering_fields = ['created_at', 'title', 'views']
    ordering = ['-created_at']  # Default ordering
```

Clients use: `GET /api/articles/?ordering=-views,title`

### Combining Filter Backends

```python
class ArticleList(generics.ListAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_fields = ['author', 'published']
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'views']
    ordering = ['-created_at']
```

---

## 12. Pagination

### The Problem

List endpoints return collections. Collections can be large. Returning millions of records in a single response would kill your server's memory, exhaust the client's bandwidth, and time out. Pagination divides results into manageable pages.

### Why Pagination is Framework-Level

Without a pagination framework, every list endpoint needs custom "page/limit" logic. The pagination format (how offsets are communicated, what metadata is included) would differ across endpoints. DRF standardizes this.

### Built-in Pagination Classes

**`PageNumberPagination`** — most intuitive; uses page numbers

```python
from rest_framework.pagination import PageNumberPagination

class StandardPagination(PageNumberPagination):
    page_size = 10           # Default page size
    page_size_query_param = 'page_size'  # Client can override: ?page_size=5
    max_page_size = 100      # Client can't request more than this
```

Response:
```json
{
    "count": 1024,
    "next": "http://api.example.com/articles/?page=3",
    "previous": "http://api.example.com/articles/?page=1",
    "results": [...]
}
```

`next` and `previous` are hypermedia links — HATEOAS in action.

**`LimitOffsetPagination`** — database-style; more flexible

`GET /api/articles/?limit=10&offset=30` → records 31–40

```python
from rest_framework.pagination import LimitOffsetPagination
```

**`CursorPagination`** — for real-time data; most robust

Uses an opaque cursor instead of page numbers or offsets. Cannot be jumped to an arbitrary page, but is consistent even when data is inserted/deleted between requests — ideal for feeds and timelines.

```python
from rest_framework.pagination import CursorPagination

class ArticleCursorPagination(CursorPagination):
    page_size = 10
    ordering = '-created_at'
```

### Setting Pagination Globally

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```

---

## 13. Parsers & Renderers

### Parsers — Handling Incoming Data

When a client sends data, it specifies the format via the `Content-Type` header. DRF's parser classes read that header and convert the raw request body into `request.data`.

Built-in parsers:
- `JSONParser` — handles `application/json` (default)
- `FormParser` — handles `application/x-www-form-urlencoded`
- `MultiPartParser` — handles `multipart/form-data` (file uploads)
- `FileUploadParser` — handles raw file upload streams

```python
from rest_framework.parsers import JSONParser, MultiPartParser

class FileUploadView(APIView):
    parser_classes = [MultiPartParser]

    def post(self, request):
        file = request.FILES['file']
        # handle file...
```

### Renderers — Formatting Outgoing Data

Renderers convert Python data structures into the format the client wants, as determined by the `Accept` header.

Built-in renderers:
- `JSONRenderer` — outputs `application/json`
- `BrowsableAPIRenderer` — outputs `text/html` (the interactive UI in the browser)
- `StaticHTMLRenderer` — outputs pre-rendered HTML
- `TemplateHTMLRenderer` — renders a Django template

Third-party renderers exist for CSV, XML, YAML, and more.

```python
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    ]
}
```

---

## 14. Content Negotiation

### The Problem

Different clients want data in different formats. A JavaScript SPA wants JSON. A legacy enterprise system wants XML. An admin browsing the API in a browser wants HTML. Should you write separate endpoints for each?

No. **Content negotiation** allows a single endpoint to serve multiple formats by inspecting the `Accept` header of each request.

### How It Works

1. Client sends `Accept: application/json` in the request header
2. DRF's `DefaultContentNegotiation` class compares the client's `Accept` header against the view's `renderer_classes`
3. The best matching renderer is selected
4. `Response(data)` renders `data` using that renderer
5. The response's `Content-Type` header is set to the renderer's `media_type`

Clients can also specify format in the URL:
- `GET /api/articles/?format=json`
- `GET /api/articles.json`

```python
# Enable URL-based format suffixes
from rest_framework.urlpatterns import format_suffix_patterns

urlpatterns = [
    path('articles/', views.ArticleList.as_view()),
]
urlpatterns = format_suffix_patterns(urlpatterns)
```

---

## 15. Versioning

### The Problem

APIs evolve. You'll add fields, remove them, rename endpoints, change behavior. But existing clients depend on your current API. Breaking changes break clients. You need a way to release new API versions while supporting old ones.

### Versioning Schemes

DRF supports four versioning schemes. Each determines how the client communicates which version it wants.

**`URLPathVersioning`** — version in the URL path (most explicit)
```
GET /api/v1/articles/
GET /api/v2/articles/
```

**`NamespaceVersioning`** — version via Django URL namespaces
```
GET /v1/articles/  (namespace: 'v1')
GET /v2/articles/  (namespace: 'v2')
```

**`QueryParameterVersioning`** — version as query parameter
```
GET /api/articles/?version=v1
```

**`AcceptHeaderVersioning`** — version in the Accept header (most RESTful, least visible)
```
Accept: application/json; version=1.0
```

### Using Versions in Views

```python
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
    'VERSION_PARAM': 'version',
}

class ArticleList(generics.ListAPIView):
    def get_serializer_class(self):
        if self.request.version == 'v2':
            return ArticleSerializerV2
        return ArticleSerializer
```

---

## 16. Routers — URL Routing at Scale

### The Problem

With generic views, you still write URL patterns manually. With ViewSets, DRF can generate URLs for you — because a ViewSet has a known, predictable set of actions.

### How Routers Work

A `Router` introspects a `ViewSet` and auto-generates the URL patterns for standard actions and any custom `@action` decorators.

```python
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'articles', ArticleViewSet, basename='article')
router.register(r'users', UserViewSet, basename='user')

urlpatterns = router.urls
```

For `ArticleViewSet(ModelViewSet)`, this generates:

| URL Pattern | HTTP Method | Action | Name |
|-------------|-------------|--------|------|
| `articles/` | GET | `list` | `article-list` |
| `articles/` | POST | `create` | `article-list` |
| `articles/{pk}/` | GET | `retrieve` | `article-detail` |
| `articles/{pk}/` | PUT | `update` | `article-detail` |
| `articles/{pk}/` | PATCH | `partial_update` | `article-detail` |
| `articles/{pk}/` | DELETE | `destroy` | `article-detail` |
| `articles/{pk}/publish/` | POST | `publish` (custom) | `article-publish` |
| `articles/featured/` | GET | `featured` (custom) | `article-featured` |

`DefaultRouter` also generates a root API endpoint at `/api/` that lists all registered routes — useful for discoverability.

`SimpleRouter` is the same but without the root API endpoint.

---

## 17. The Browsable API

### The Problem

Testing an API normally requires tools like Postman, curl, or Insomnia. During development, context-switching to an external tool for every change slows you down considerably.

### What It Is

`BrowsableAPIRenderer` generates a full HTML interface for your API — rendered in the browser. Every endpoint is accessible, explorable, and interactive. You can:
- See the response in formatted JSON
- See which HTTP methods are allowed
- Submit POST/PUT/PATCH forms
- Authenticate via session

This is what you see when you visit `http://localhost:8000/api/articles/` in a browser while developing a DRF API. It's powered by the content negotiation system — a browser sends `Accept: text/html`, so DRF picks the HTML renderer.

In production, you typically remove `BrowsableAPIRenderer` from `DEFAULT_RENDERER_CLASSES` so browsers get JSON instead.

---

## 18. Settings & Configuration

DRF's global behavior is configured in `settings.py` under the `REST_FRAMEWORK` dict. This is the single place to establish defaults that all views inherit, reducing per-view boilerplate.

```python
REST_FRAMEWORK = {
    # Authentication
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],

    # Permissions — default is AllowAny if not set
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],

    # Throttling
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
    },

    # Pagination
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,

    # Filtering
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],

    # Rendering
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',  # Remove in production
    ],

    # Parsing
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
        'rest_framework.parsers.FormParser',
        'rest_framework.parsers.MultiPartParser',
    ],

    # Versioning
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],

    # Exception handling
    'EXCEPTION_HANDLER': 'rest_framework.views.exception_handler',

    # Datetime handling
    'DATETIME_FORMAT': 'iso-8601',
}
```

---

## 19. Putting It All Together — The DRF Request Lifecycle

This is the complete picture of what happens when a request hits your DRF API:

```
Incoming HTTP Request
        │
        ▼
  Django URL Dispatcher
  (matches to a ViewSet/View via urlpatterns or Router)
        │
        ▼
  ViewSet.as_view() / APIView.dispatch()
        │
        ▼
  1. AUTHENTICATION
     └── Try each class in authentication_classes
     └── Set request.user and request.auth
        │
        ▼
  2. PERMISSIONS (view-level)
     └── check_permissions() on each class in permission_classes
     └── HTTP 403/401 if any fails
        │
        ▼
  3. THROTTLING
     └── check_throttles() on each class in throttle_classes
     └── HTTP 429 if any fails
        │
        ▼
  4. CONTENT NEGOTIATION
     └── Match client's Accept header to renderer_classes
     └── Match client's Content-Type to parser_classes
        │
        ▼
  5. DISPATCH to method handler
     (get, post, put, patch, delete — or list, create, etc. for ViewSets)
        │
        ▼
  6. PARSING (inside request.data access)
     └── Parse request body using negotiated parser
     └── Populate request.data
        │
        ▼
  7. FILTERING (for list views)
     └── Apply each filter_backend to queryset
        │
        ▼
  8. PAGINATION (for list views)
     └── Slice queryset per pagination settings
        │
        ▼
  9. SERIALIZATION
     └── Instantiate serializer with queryset/instance
     └── For writes: validate via is_valid(), save()
     └── Serialize output to Python dict
        │
        ▼
  10. OBJECT-LEVEL PERMISSIONS (detail views)
      └── check_object_permissions() when get_object() is called
      └── HTTP 403 if any fails
        │
        ▼
  11. RESPONSE
      └── Response(data, status=...)
      └── Renderer converts Python dict to bytes (JSON, HTML, etc.)
      └── HTTP Response sent to client
```

---

## 20. Mental Model Cheat Sheet

### The Hierarchy of Abstraction

```
HttpRequest → APIView → GenericAPIView → Concrete Generic View → ViewSet → ModelViewSet
  (Django)    (DRF base)  (+ queryset/  (+ standard actions)  (+ routing) (+ all CRUD)
                           serializer)
```

Each step up the hierarchy adds more conventions and removes more boilerplate. Start with `ModelViewSet` for CRUD resources. Drop down to `GenericAPIView` or `APIView` when you need control.

### When to Use What

| Situation | Use |
|-----------|-----|
| Simple CRUD resource | `ModelViewSet` + `DefaultRouter` |
| Read-only public resource | `ReadOnlyModelViewSet` |
| Non-standard endpoint (e.g., login, search) | `APIView` or `@api_view` |
| Endpoint with some but not all CRUD | `GenericAPIView` + specific Mixins |
| Custom action on a resource | `@action` on a ViewSet |
| Complex nested relationships (write) | Custom `create()`/`update()` on Serializer |
| Same data, different shapes | Override `get_serializer_class()` |
| User-scoped queryset | Override `get_queryset()` |

### Serializer Quick Reference

```python
# Output only (GET)
ArticleSerializer(instance)
ArticleSerializer(queryset, many=True)

# Input validation + create (POST)
ArticleSerializer(data=request.data)
serializer.is_valid(raise_exception=True)  # auto-raises 400 on failure
serializer.save()  # calls create()

# Input validation + update (PUT/PATCH)
ArticleSerializer(instance, data=request.data, partial=True)  # partial for PATCH
serializer.is_valid(raise_exception=True)
serializer.save()  # calls update()

# With request context (for SerializerMethodField that needs request)
ArticleSerializer(instance, context={'request': request})
```

### Permission Decision Tree

```
Is request method in SAFE_METHODS (GET, HEAD, OPTIONS)?
├── Yes → AllowAny or IsAuthenticatedOrReadOnly → ALLOW
└── No (POST/PUT/PATCH/DELETE)
    ├── Is user authenticated?
    │   ├── No → HTTP 401
    │   └── Yes
    │       ├── Is user the owner? (IsOwnerOrReadOnly)
    │       │   ├── Yes → ALLOW
    │       │   └── No → HTTP 403
    │       └── Is user admin? (IsAdminUser)
    │           ├── Yes → ALLOW
    │           └── No → HTTP 403
```

---

## Further Learning Path

Follow these in order alongside the official DRF tutorial:

1. **DRF Official Tutorial** — Quickstart → Serialization → Requests/Responses → Class-based views → Auth & Permissions → Relationships & Hyperlinked APIs → ViewSets & Routers
2. **`django-filter` docs** — for complex filtering
3. **`djangorestframework-simplejwt`** — for JWT authentication in production
4. **DRF Spectacular or drf-yasg** — for auto-generating OpenAPI/Swagger docs
5. **`django-cors-headers`** — for cross-origin requests from a separate frontend
6. **Nested routers** (drf-nested-routers) — for `/api/articles/42/comments/`

---

*Built to be read alongside the official DRF documentation at https://www.django-rest-framework.org/tutorial/quickstart/*
