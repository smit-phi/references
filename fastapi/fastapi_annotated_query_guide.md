# `Annotated` and `Query` in FastAPI — Complete Guide
> *Building intuition from first principles, then mastering every argument*

---

## Table of Contents

1. [Why Does `Annotated` Exist At All?](#1-why-does-annotated-exist-at-all)
2. [The Problem It Solves — A Visual Build-Up](#2-the-problem-it-solves--a-visual-build-up)
3. [How `Annotated` Actually Works](#3-how-annotated-actually-works)
4. [Annotated in FastAPI — The Mental Model](#4-annotated-in-fastapi--the-mental-model)
5. [`Query` — Every Argument Explained](#5-query--every-argument-explained)
6. [`Path` — Every Argument Explained](#6-path--every-argument-explained)
7. [`Header` and `Cookie`](#7-header-and-cookie)
8. [`Body` — Every Argument Explained](#8-body--every-argument-explained)
9. [`Field` — Pydantic-Level Validation](#9-field--pydantic-level-validation)
10. [Reusable Annotated Types — The Real Power](#10-reusable-annotated-types--the-real-power)
11. [Annotated with `Depends` — DI Integration](#11-annotated-with-depends--di-integration)
12. [Common Patterns & Real-World Examples](#12-common-patterns--real-world-examples)
13. [Quick Reference Cheatsheet](#13-quick-reference-cheatsheet)

---

## 1. Why Does `Annotated` Exist At All?

Before FastAPI, before Pydantic — Python type hints were always meant to be **just hints**. They describe what a variable should be, but Python never enforces them at runtime. They're for your editor, your type checker, and documentation.

```python
def greet(name: str) -> str:
    return f"Hello {name}"

greet(123)  # Python doesn't care — no error
```

Over time, libraries like Pydantic started **reading** these type hints at runtime to do actual validation work. But a problem emerged:

> **A type hint can only carry a type. What if you need to carry more information alongside the type?**

What if you want to say: *"This is a string, AND it must be between 3 and 50 characters, AND it should appear in the OpenAPI docs as 'username'"*?

There's nowhere to put that extra information inside `str`. You can't write `str(min_length=3)` — that's not valid Python.

This is the exact problem `Annotated` was created to solve.

---

## 2. The Problem It Solves — A Visual Build-Up

### Stage 1 — The old way (no metadata)

```python
@app.get("/items")
def list_items(limit: int = 10):
    return []
```

Works fine. But you have no way to say: *limit must be between 1 and 100*. You'd need to manually validate inside the function:

```python
@app.get("/items")
def list_items(limit: int = 10):
    if limit < 1 or limit > 100:
        raise HTTPException(400, "limit must be between 1 and 100")
    return []
```

This is repetitive, not self-documenting, and doesn't appear in your OpenAPI schema.

---

### Stage 2 — The `Query()` way (before `Annotated`)

FastAPI introduced `Query()` to attach constraints to parameters:

```python
from fastapi import Query

@app.get("/items")
def list_items(limit: int = Query(default=10, ge=1, le=100)):
    return []
```

Better — now FastAPI validates automatically and the constraint shows up in `/docs`. But now `limit` has a messy signature that mixes the *default value* with *validation rules*.

---

### Stage 3 — The `Annotated` way (clean separation)

```python
from typing import Annotated
from fastapi import Query

@app.get("/items")
def list_items(limit: Annotated[int, Query(ge=1, le=100)] = 10):
    return []
```

Notice what changed:
- The **type** (`int`) lives inside `Annotated`
- The **constraints** (`Query(ge=1, le=100)`) live inside `Annotated`
- The **default value** (`= 10`) is back where it belongs — as a regular Python default

The function signature is now clean and each concern is in the right place.

---

## 3. How `Annotated` Actually Works

`Annotated` is from Python's standard `typing` module (Python 3.9+). It takes this form:

```python
Annotated[ActualType, metadata1, metadata2, ...]
```

The first argument is always the real type. Everything after is arbitrary metadata that tools can read.

```python
from typing import Annotated, get_type_hints, get_args

# Plain Python — Annotated is just a container
x: Annotated[int, "must be positive", "used in reports"]

# The type is still int
args = get_args(Annotated[int, "some metadata"])
print(args)  # (int, 'some metadata')
```

Python itself ignores the metadata. But libraries like FastAPI and Pydantic inspect it using `get_type_hints()` and `get_args()` at runtime to extract the extra information and act on it.

FastAPI's logic is roughly:

```
See Annotated[int, Query(ge=1)]
         ↓
Extract type → int
Extract metadata → Query(ge=1)
         ↓
"This is an int query param, validate ge=1"
         ↓
Build OpenAPI schema, run validation, inject value into function
```

---

## 4. Annotated in FastAPI — The Mental Model

Think of `Annotated` as a **label attached to a type**. The type tells FastAPI *what* the value is. The label tells FastAPI *where it comes from* and *what rules apply*.

```
Annotated[ int,  Query(ge=1, le=100, description="Page size") ]
           ↑              ↑
        Python          FastAPI
         type           metadata
    (what it is)   (where it is + rules)
```

Without `Annotated`, you're cramming type + source + rules + default all into one expression:

```python
# Everything jammed together — hard to read, hard to reuse
limit: int = Query(default=10, ge=1, le=100, description="Page size limit")
```

With `Annotated`, each concern has its place:

```python
# Type and rules in Annotated, default stays as Python default
limit: Annotated[int, Query(ge=1, le=100, description="Page size limit")] = 10
```

---

## 5. `Query` — Every Argument Explained

`Query` is used for **query string parameters** (the `?key=value` part of a URL).

```python
from fastapi import Query
from typing import Annotated

@app.get("/items")
def list_items(
    limit: Annotated[int, Query(...)] = 10
):
    ...
```

### Full signature

```python
Query(
    default=...,          # default value (use ... for required)
    default_factory=None, # callable that produces the default
    alias=None,           # different name in URL than in function
    alias_priority=None,  # priority when alias conflicts
    title=None,           # OpenAPI title
    description=None,     # OpenAPI description (shows in /docs)
    gt=None,              # greater than (exclusive)
    ge=None,              # greater than or equal (inclusive)
    lt=None,              # less than (exclusive)
    le=None,              # less than or equal (inclusive)
    min_length=None,      # string minimum length
    max_length=None,      # string maximum length
    pattern=None,         # regex pattern the value must match
    discriminator=None,   # for union type discrimination
    strict=None,          # strict type checking (no coercion)
    deprecated=None,      # marks param as deprecated in OpenAPI
    include_in_schema=True, # whether to show in OpenAPI docs
    examples=None,        # example values for OpenAPI docs
)
```

---

### `default` and required params

```python
# Required — no default, client MUST provide it
search: Annotated[str, Query()] 
# OR equivalently:
search: Annotated[str, Query(...)]

# Optional with default
limit: Annotated[int, Query()] = 10

# Optional, defaults to None
search: Annotated[str | None, Query()] = None
```

---

### `alias` — different URL name vs Python name

Python variables can't have hyphens, but URLs often do. `alias` bridges this gap.

```python
@app.get("/items")
def list_items(
    sort_by: Annotated[str, Query(alias="sort-by")] = "name"
):
    # Client sends: /items?sort-by=price
    # Python receives: sort_by = "price"
    return {"sort_by": sort_by}
```

Also useful when the external API contract uses reserved Python words:

```python
# "from" is a Python keyword — use alias
date_from: Annotated[str | None, Query(alias="from")] = None
```

---

### `gt`, `ge`, `lt`, `le` — numeric constraints

```python
@app.get("/items")
def list_items(
    page: Annotated[int, Query(ge=1)] = 1,              # page >= 1
    limit: Annotated[int, Query(ge=1, le=100)] = 10,    # 1 <= limit <= 100
    min_price: Annotated[float, Query(gt=0)] = None,    # price > 0 (strictly)
    max_price: Annotated[float, Query(gt=0)] = None,
):
    ...
```

| Argument | Meaning | Example |
|---|---|---|
| `gt=5` | value > 5 (strictly greater) | 6, 7, 8... |
| `ge=5` | value >= 5 (inclusive) | 5, 6, 7... |
| `lt=100` | value < 100 (strictly less) | 99, 98... |
| `le=100` | value <= 100 (inclusive) | 100, 99... |

---

### `min_length`, `max_length`, `pattern` — string constraints

```python
@app.get("/search")
def search(
    q: Annotated[str, Query(min_length=3, max_length=100)] = None,
    username: Annotated[str, Query(pattern=r"^[a-z0-9_]+$")] = None,
    # pattern: regex the entire string must match
):
    ...
```

---

### `description` and `title` — OpenAPI documentation

These appear directly in your `/docs` Swagger UI — essential for API consumers.

```python
@app.get("/products")
def list_products(
    category: Annotated[
        str | None,
        Query(
            title="Product Category",
            description="Filter by product category. Use 'all' to return everything.",
        )
    ] = None,
):
    ...
```

---

### `deprecated` — mark old params in docs

```python
@app.get("/items")
def list_items(
    sort: Annotated[str, Query(deprecated=True)] = "name",
    order_by: Annotated[str, Query(description="Use this instead of 'sort'")] = "name",
):
    ...
```

The `/docs` UI will show `sort` with a strikethrough and deprecation notice.

---

### `include_in_schema` — hide from OpenAPI docs

Useful for internal params, debugging flags, or feature flags you don't want in public docs.

```python
@app.get("/items")
def list_items(
    debug: Annotated[bool, Query(include_in_schema=False)] = False,
):
    ...
```

---

### `examples` — show sample values in docs

```python
@app.get("/search")
def search(
    q: Annotated[
        str,
        Query(
            examples={
                "simple": {"value": "laptop", "summary": "Simple search"},
                "complex": {"value": "laptop 16gb ram", "summary": "Multi-word search"},
            }
        )
    ]
):
    ...
```

---

### `strict` — prevent type coercion

By default FastAPI coerces `"10"` (string from URL) to `10` (int). With `strict=True`, it won't:

```python
# Without strict: /items?limit=10.5 → limit = 10 (coerced)
# With strict:    /items?limit=10.5 → 422 Validation Error
limit: Annotated[int, Query(strict=True)] = 10
```

---

## 6. `Path` — Every Argument Explained

`Path` works identically to `Query` but for **path parameters** (`/items/{item_id}`).

```python
from fastapi import Path

@app.get("/items/{item_id}")
def get_item(
    item_id: Annotated[int, Path(
        title="Item ID",
        description="The ID of the item to retrieve",
        ge=1,               # IDs must be positive
        examples={
            "valid": {"value": 42}
        }
    )]
):
    return {"id": item_id}
```

Path parameters are **always required** — there's no `default` argument because the value is part of the URL structure. Everything else (`gt`, `ge`, `lt`, `le`, `title`, `description`, `deprecated`, `examples`) works exactly the same as `Query`.

---

## 7. `Header` and `Cookie`

### `Header`

```python
from fastapi import Header

@app.get("/protected")
def protected(
    authorization: Annotated[str | None, Header()] = None,
    x_request_id: Annotated[str | None, Header(alias="X-Request-ID")] = None,
):
    ...
```

**Important:** FastAPI automatically converts underscores to hyphens for headers. `x_request_id` maps to `X-Request-Id` by default. Use `alias` for exact control.

```python
# Disable auto-conversion
user_agent: Annotated[str | None, Header(convert_underscores=False)] = None
```

### `Cookie`

```python
from fastapi import Cookie

@app.get("/dashboard")
def dashboard(
    session_id: Annotated[str | None, Cookie()] = None,
):
    ...
```

Same arguments as `Query` — `alias`, `min_length`, `max_length`, `description`, etc.

---

## 8. `Body` — Every Argument Explained

`Body` is for request body parameters — typically used when you need to mix multiple body models or embed a single value in the body.

```python
from fastapi import Body

# Single value from body (not a full Pydantic model)
@app.post("/items/{item_id}/rate")
def rate_item(
    item_id: int,
    rating: Annotated[int, Body(ge=1, le=5, description="Rating from 1 to 5")],
):
    return {"item_id": item_id, "rating": rating}
```

### `embed` — wrap single model in a key

```python
class Item(BaseModel):
    name: str
    price: float

# Without embed — body is directly: {"name": "x", "price": 9.9}
@app.post("/items")
def create_item(item: Item):
    ...

# With embed — body must be: {"item": {"name": "x", "price": 9.9}}
@app.post("/items")
def create_item(item: Annotated[Item, Body(embed=True)]):
    ...
```

`embed=True` is useful when you have multiple body parameters — FastAPI wraps each in its own key:

```python
@app.put("/items/{item_id}")
def update_item(
    item_id: int,
    item: Annotated[Item, Body(embed=True)],
    user: Annotated[User, Body(embed=True)],
    # Body expected: {"item": {...}, "user": {...}}
):
    ...
```

---

## 9. `Field` — Pydantic-Level Validation

`Field` is Pydantic's equivalent of `Query`/`Path`/`Body` but it lives **inside** a Pydantic model, not in the function signature.

```python
from pydantic import BaseModel, Field

class ItemCreate(BaseModel):
    name: str = Field(
        min_length=2,
        max_length=100,
        description="The item name",
        examples=["Laptop", "Desk Chair"],
    )
    price: float = Field(
        gt=0,
        description="Price in USD",
    )
    quantity: int = Field(
        default=1,
        ge=0,
        le=10000,
    )
    sku: str = Field(
        pattern=r"^[A-Z]{3}-\d{4}$",
        description="SKU format: ABC-1234",
    )
```

`Field` takes the same numeric and string constraints as `Query`. The difference is scope:

| | Scope | Used In |
|---|---|---|
| `Query` | FastAPI routing layer | Function parameters (query string) |
| `Path` | FastAPI routing layer | Function parameters (URL path) |
| `Body` | FastAPI routing layer | Function parameters (request body) |
| `Field` | Pydantic model layer | Inside `BaseModel` class definitions |

---

## 10. Reusable Annotated Types — The Real Power

This is where `Annotated` truly shines and why it was introduced over the old `= Query(...)` style. You can **define a type once** and reuse it everywhere.

### Without Annotated — repetition everywhere

```python
@app.get("/items")
def list_items(limit: int = Query(ge=1, le=100, description="Results per page") = 20):
    ...

@app.get("/users")
def list_users(limit: int = Query(ge=1, le=100, description="Results per page") = 20):
    ...

@app.get("/orders")
def list_orders(limit: int = Query(ge=1, le=100, description="Results per page") = 20):
    ...
```

The `ge=1, le=100, description=...` block is copy-pasted everywhere. If you need to change the max to 200, you update every route.

### With Annotated — define once, use everywhere

```python
from typing import Annotated
from fastapi import Query

# Define your type once
PageLimit = Annotated[int, Query(ge=1, le=100, description="Results per page")]
PageOffset = Annotated[int, Query(ge=0, description="Number of results to skip")]
SearchQuery = Annotated[str | None, Query(min_length=2, max_length=200, description="Search string")]

# Use it like a regular type across all routes
@app.get("/items")
def list_items(limit: PageLimit = 20, offset: PageOffset = 0, q: SearchQuery = None):
    ...

@app.get("/users")
def list_users(limit: PageLimit = 20, offset: PageOffset = 0, q: SearchQuery = None):
    ...

@app.get("/orders")
def list_orders(limit: PageLimit = 20, offset: PageOffset = 0, q: SearchQuery = None):
    ...
```

Change the rule in one place, all routes get updated. This is the primary reason FastAPI recommends `Annotated` over the old inline style.

---

### Building a library of domain types

```python
# types.py — your project's type vocabulary
from typing import Annotated
from fastapi import Query, Path

# Pagination
PageLimit   = Annotated[int, Query(ge=1, le=500, description="Items per page")]
PageOffset  = Annotated[int, Query(ge=0, description="Offset for pagination")]

# IDs
ItemId   = Annotated[int, Path(ge=1, description="Item primary key")]
UserId   = Annotated[int, Path(ge=1, description="User primary key")]
OrderId  = Annotated[int, Path(ge=1, description="Order primary key")]

# Common filters
SearchStr    = Annotated[str | None, Query(min_length=2, max_length=200)]
SortOrder    = Annotated[str, Query(pattern=r"^(asc|desc)$")]
DateFilter   = Annotated[str | None, Query(pattern=r"^\d{4}-\d{2}-\d{2}$", description="Date in YYYY-MM-DD")]
```

```python
# routers/items.py
from types import ItemId, PageLimit, PageOffset, SearchStr

@router.get("/items")
def list_items(limit: PageLimit = 20, offset: PageOffset = 0, q: SearchStr = None):
    ...

@router.get("/items/{item_id}")
def get_item(item_id: ItemId):
    ...
```

Your routes are now clean, consistent, and self-documenting.

---

## 11. Annotated with `Depends` — DI Integration

`Annotated` works seamlessly with FastAPI's dependency injection system, making DI cleaner:

### Old style

```python
from fastapi import Depends

@app.get("/items")
def list_items(db: Session = Depends(get_db), user = Depends(get_current_user)):
    ...
```

### Annotated style

```python
DB = Annotated[Session, Depends(get_db)]
CurrentUser = Annotated[User, Depends(get_current_user)]
AdminUser = Annotated[User, Depends(require_admin)]

@app.get("/items")
def list_items(db: DB, user: CurrentUser):
    ...

@app.delete("/items/{item_id}")
def delete_item(item_id: int, db: DB, admin: AdminUser):
    ...
```

Now `DB`, `CurrentUser`, and `AdminUser` are reusable type aliases that carry their dependency resolution with them. Any route that needs a database session just declares `db: DB` — one word, no repetition.

### Combining Query + Depends in the same Annotated

```python
def verify_api_key(api_key: str = Header(...)):
    if api_key != settings.api_key:
        raise HTTPException(403, "Invalid API key")
    return api_key

# Single type that means: "a string from headers, validated as API key"
ApiKey = Annotated[str, Depends(verify_api_key)]

@app.get("/internal/stats")
def get_stats(api_key: ApiKey):
    return {"data": "..."}
```

---

## 12. Common Patterns & Real-World Examples

### Pagination dependency

```python
from dataclasses import dataclass
from typing import Annotated
from fastapi import Query, Depends

@dataclass
class PaginationParams:
    limit: int = Query(default=20, ge=1, le=500)
    offset: int = Query(default=0, ge=0)

Pagination = Annotated[PaginationParams, Depends()]

@app.get("/items")
def list_items(pagination: Pagination):
    return {
        "limit": pagination.limit,
        "offset": pagination.offset,
    }
```

### Validated ID from path

```python
from fastapi import Path, HTTPException
from sqlalchemy.orm import Session

def get_item_or_404(
    item_id: Annotated[int, Path(ge=1)],
    db: DB,
):
    item = db.query(Item).get(item_id)
    if not item:
        raise HTTPException(404, f"Item {item_id} not found")
    return item

ItemOrError = Annotated[Item, Depends(get_item_or_404)]

@app.get("/items/{item_id}")
def get_item(item: ItemOrError):
    return item

@app.put("/items/{item_id}")
def update_item(item: ItemOrError, data: ItemUpdate):
    # item is already validated and fetched — no boilerplate
    ...
```

### Sorting and filtering pattern

```python
from enum import Enum

class SortField(str, Enum):
    name = "name"
    price = "price"
    created_at = "created_at"

class SortOrder(str, Enum):
    asc = "asc"
    desc = "desc"

@app.get("/products")
def list_products(
    sort_by: Annotated[SortField, Query(description="Field to sort by")] = SortField.name,
    order: Annotated[SortOrder, Query(description="Sort direction")] = SortOrder.asc,
    min_price: Annotated[float | None, Query(gt=0, description="Minimum price filter")] = None,
    max_price: Annotated[float | None, Query(gt=0, description="Maximum price filter")] = None,
    in_stock: Annotated[bool | None, Query(description="Filter by stock availability")] = None,
):
    ...
```

Using `Enum` for `sort_by` means FastAPI automatically validates the value is one of the allowed options AND shows them as a dropdown in `/docs`.

### Full route with everything

```python
from typing import Annotated
from fastapi import APIRouter, Query, Path, Depends, Header
from pydantic import BaseModel, Field

router = APIRouter(prefix="/orders", tags=["orders"])

# Reusable types
DB = Annotated[Session, Depends(get_db)]
CurrentUser = Annotated[User, Depends(get_current_user)]
OrderId = Annotated[int, Path(ge=1, description="Order primary key")]
PageLimit = Annotated[int, Query(ge=1, le=100)] 

class OrderUpdate(BaseModel):
    status: str = Field(pattern=r"^(pending|confirmed|shipped|delivered)$")
    notes: str | None = Field(default=None, max_length=500)

@router.get("/")
def list_orders(
    db: DB,
    user: CurrentUser,
    limit: PageLimit = 20,
    offset: Annotated[int, Query(ge=0)] = 0,
    status: Annotated[str | None, Query(description="Filter by order status")] = None,
):
    query = db.query(Order).filter(Order.user_id == user.id)
    if status:
        query = query.filter(Order.status == status)
    return query.offset(offset).limit(limit).all()

@router.patch("/{order_id}")
def update_order(
    order_id: OrderId,
    data: OrderUpdate,
    db: DB,
    user: CurrentUser,
):
    order = db.query(Order).filter(Order.id == order_id, Order.user_id == user.id).first()
    if not order:
        raise HTTPException(404, "Order not found")
    for field, value in data.model_dump(exclude_none=True).items():
        setattr(order, field, value)
    db.commit()
    return order
```

---

## 13. Quick Reference Cheatsheet

### Where each tool lives

```
URL: /items?limit=10&search=laptop
              ↑              ↑
           Query()        Query()

URL: /items/{item_id}/reviews/{review_id}
              ↑                  ↑
           Path()             Path()

HTTP Headers → Header()
HTTP Cookies → Cookie()
Request Body → Body() or Pydantic model
Inside model → Field()
```

### Constraint arguments at a glance

| Argument | Type | Works in | Meaning |
|---|---|---|---|
| `ge` | number | Query, Path, Body, Field | value >= N |
| `gt` | number | Query, Path, Body, Field | value > N |
| `le` | number | Query, Path, Body, Field | value <= N |
| `lt` | number | Query, Path, Body, Field | value < N |
| `min_length` | str | Query, Path, Body, Field | len >= N |
| `max_length` | str | Query, Path, Body, Field | len <= N |
| `pattern` | str | Query, Path, Body, Field | must match regex |
| `alias` | any | Query, Path, Header, Cookie | different external name |
| `description` | any | all | OpenAPI docs |
| `title` | any | all | OpenAPI docs |
| `deprecated` | any | Query, Path | shows strikethrough in docs |
| `include_in_schema` | any | Query, Path | hide from OpenAPI |
| `examples` | any | all | sample values in docs |
| `strict` | any | Query, Path | no type coercion |
| `embed` | model | Body | wrap in named key |

### Style comparison — old vs Annotated

```python
# Old style — type and metadata mixed in default value
def fn(limit: int = Query(default=10, ge=1, le=100)):

# Annotated style — clean separation
def fn(limit: Annotated[int, Query(ge=1, le=100)] = 10):

# Reusable Annotated type — the best style
PageLimit = Annotated[int, Query(ge=1, le=100)]
def fn(limit: PageLimit = 10):
```

---

*The core idea to remember: `Annotated[Type, metadata]` lets you attach rules and source information to a type without touching the default value. FastAPI reads this metadata to validate, convert, and document your parameters automatically.*
