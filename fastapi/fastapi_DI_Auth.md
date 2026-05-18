# FastAPI: Dependencies, Database Integration & Authentication

> A comprehensive, production-oriented guide covering FastAPI's dependency injection system,
> database integration with SQLAlchemy, and secure authentication with OAuth2 + JWT.

---

## Table of Contents

1. [Dependencies & Dependency Injection](#1-dependencies--dependency-injection)
   - 1.1 [What is Dependency Injection?](#11-what-is-dependency-injection)
   - 1.2 [First Principles: The Problem DI Solves](#12-first-principles-the-problem-di-solves)
   - 1.3 [FastAPI's `Depends()` System](#13-fastapis-depends-system)
   - 1.4 [Types of Dependencies](#14-types-of-dependencies)
   - 1.5 [Dependency Chains & Nesting](#15-dependency-chains--nesting)
   - 1.6 [Scoped & Shared Dependencies](#16-scoped--shared-dependencies)
   - 1.7 [Real-World Dependency Patterns](#17-real-world-dependency-patterns)

2. [Database Integration & ORM](#2-database-integration--orm)
   - 2.1 [Why Use an ORM?](#21-why-use-an-orm)
   - 2.2 [SQLAlchemy Core Concepts](#22-sqlalchemy-core-concepts)
   - 2.3 [Setting Up SQLAlchemy with FastAPI](#23-setting-up-sqlalchemy-with-fastapi)
   - 2.4 [Defining Models](#24-defining-models)
   - 2.5 [The Database Dependency Pattern](#25-the-database-dependency-pattern)
   - 2.6 [CRUD Operations](#26-crud-operations)
   - 2.7 [Async SQLAlchemy](#27-async-sqlalchemy)
   - 2.8 [Alembic Migrations](#28-alembic-migrations)
   - 2.9 [ORM Alternatives](#29-orm-alternatives)

3. [Authentication & Authorization](#3-authentication--authorization)
   - 3.1 [OAuth2 with Password Flow (JWT)](#31-oauth2-with-password-flow-jwt)
   - 3.2 [JWT Deep Dive](#32-jwt-deep-dive)
   - 3.3 [Implementing the Full Auth System](#33-implementing-the-full-auth-system)
   - 3.4 [Role-Based Access Control (RBAC)](#34-role-based-access-control-rbac)
   - 3.5 [Security Best Practices](#35-security-best-practices)

---

## 1. Dependencies & Dependency Injection

### 1.1 What is Dependency Injection?

**Dependency Injection (DI)** is a design pattern where an object (the *consumer*) receives the things it needs (its *dependencies*) from an external source, rather than creating them itself.

Think of it like a restaurant kitchen. The chef (your route handler) doesn't go to the farm to get eggs — someone *injects* them into the kitchen. The chef just cooks.

```
Without DI:                          With DI:
┌──────────────────┐                 ┌──────────────────┐
│  Route Handler   │                 │  Route Handler   │
│  ┌────────────┐  │                 │  (just uses it)  │
│  │ Creates DB │  │                 └────────┬─────────┘
│  │ connection │  │                          │ injected
│  │ Validates  │  │                 ┌────────▼─────────┐
│  │ auth token │  │                 │  Dependency      │
│  │ Parses     │  │                 │  (DB, Auth, etc) │
│  │ pagination │  │                 └──────────────────┘
│  └────────────┘  │
└──────────────────┘
```

### 1.2 First Principles: The Problem DI Solves

Consider a naive API without DI:

```python
# ❌ WITHOUT dependency injection — hard to test, lots of duplication
from fastapi import FastAPI
import sqlite3

app = FastAPI()

@app.get("/users/{user_id}")
def get_user(user_id: int, token: str):
    # Auth logic duplicated in every endpoint
    if not token or token != "secret":
        raise HTTPException(status_code=401, detail="Unauthorized")

    # DB connection created fresh every time
    conn = sqlite3.connect("mydb.db")
    cursor = conn.cursor()
    user = cursor.execute("SELECT * FROM users WHERE id=?", (user_id,)).fetchone()
    conn.close()
    return user

@app.get("/posts/{post_id}")
def get_post(post_id: int, token: str):
    # Same auth logic duplicated AGAIN
    if not token or token != "secret":
        raise HTTPException(status_code=401, detail="Unauthorized")

    conn = sqlite3.connect("mydb.db")
    # ...same pattern again
```

**Problems with the above:**
- Auth logic is copy-pasted everywhere — a bug means fixing it in 50 places.
- Database connections are unmanaged — no cleanup, no pooling.
- Impossible to unit test without a real database and a real token.
- Impossible to swap implementations (e.g., switch from SQLite to PostgreSQL).

DI solves all of this elegantly.

### 1.3 FastAPI's `Depends()` System

FastAPI's `Depends()` is the core primitive for declaring dependencies. It tells FastAPI: *"Before calling my function, call this other function and give me its return value."*

```python
from fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

# ─── A simple dependency ─────────────────────────────────────────────────────
def verify_token(token: str) -> str:
    """This is a dependency — a plain Python function."""
    if token != "secret":
        raise HTTPException(status_code=401, detail="Invalid token")
    return token

# ─── Route uses it via Depends() ─────────────────────────────────────────────
@app.get("/secure-data")
def get_secure_data(verified_token: str = Depends(verify_token)):
    # FastAPI resolved `verify_token` before calling this function.
    # If it raised an HTTPException, this line is never reached.
    return {"data": "sensitive info", "verified_by": verified_token}
```

**How FastAPI resolves dependencies:**

```
HTTP Request: GET /secure-data?token=secret
        │
        ▼
FastAPI inspects get_secure_data's signature
        │
        ▼
Finds Depends(verify_token)
        │
        ▼
Calls verify_token(token="secret")  ← query param auto-extracted
        │
        ├─ Raises HTTPException? → return 401 immediately
        │
        └─ Returns "secret" → inject as `verified_token`
                │
                ▼
        Calls get_secure_data(verified_token="secret")
                │
                ▼
        Returns {"data": "sensitive info", ...}
```

### 1.4 Types of Dependencies

#### Function Dependencies

The most common type — just a Python function:

```python
from fastapi import Depends, Query

def common_pagination(
    page: int = Query(default=1, ge=1),
    page_size: int = Query(default=20, ge=1, le=100),
) -> dict:
    offset = (page - 1) * page_size
    return {"offset": offset, "limit": page_size}

@app.get("/items")
def list_items(pagination: dict = Depends(common_pagination)):
    # pagination = {"offset": 0, "limit": 20} for defaults
    return fetch_items(**pagination)

@app.get("/products")
def list_products(pagination: dict = Depends(common_pagination)):
    # Same pagination logic reused — zero duplication
    return fetch_products(**pagination)
```

#### Class Dependencies

Classes give you a callable with state — useful for configurable dependencies:

```python
class PaginationParams:
    def __init__(
        self,
        page: int = Query(default=1, ge=1),
        page_size: int = Query(default=20, ge=1, le=100),
    ):
        self.offset = (page - 1) * page_size
        self.limit = page_size

@app.get("/items")
def list_items(pagination: PaginationParams = Depends(PaginationParams)):
    items = fetch_items(offset=pagination.offset, limit=pagination.limit)
    return items

# ── Configurable class dependency ────────────────────────────────────────────
class APIKeyChecker:
    def __init__(self, service_name: str):
        self.service_name = service_name

    def __call__(self, api_key: str = Header(...)):
        valid_keys = get_valid_keys_for_service(self.service_name)
        if api_key not in valid_keys:
            raise HTTPException(status_code=403, detail="Invalid API key")
        return api_key

# Two instances with different configuration
check_analytics_key = APIKeyChecker(service_name="analytics")
check_payments_key  = APIKeyChecker(service_name="payments")

@app.post("/analytics/event")
def track_event(key: str = Depends(check_analytics_key)):
    ...

@app.post("/payments/charge")
def charge_card(key: str = Depends(check_payments_key)):
    ...
```

#### Generator Dependencies (with `yield`)

The most powerful form — lets you run cleanup code *after* the request is done.
This is how database sessions and other context-managed resources are handled:

```python
from contextlib import contextmanager

def get_db_connection():
    """Generator dependency — setup before yield, teardown after."""
    conn = create_connection()   # runs BEFORE the route handler
    try:
        yield conn               # injected into the route handler
    finally:
        conn.close()             # runs AFTER the route handler (always)

@app.get("/users/{user_id}")
def get_user(user_id: int, conn = Depends(get_db_connection)):
    return conn.execute("SELECT * FROM users WHERE id=?", user_id)
    # conn.close() is called automatically after this returns
```

#### Async Dependencies

FastAPI handles async dependencies naturally alongside sync ones:

```python
import httpx

async def get_http_client():
    async with httpx.AsyncClient() as client:
        yield client  # client is alive during the request, closed after

@app.get("/weather/{city}")
async def get_weather(city: str, client: httpx.AsyncClient = Depends(get_http_client)):
    response = await client.get(f"https://api.weather.com/{city}")
    return response.json()
```

### 1.5 Dependency Chains & Nesting

Dependencies can depend on other dependencies. FastAPI builds and resolves the entire tree automatically:

```python
# Layer 1: raw token extraction
def extract_token(authorization: str = Header(...)) -> str:
    scheme, _, token = authorization.partition(" ")
    if scheme.lower() != "bearer":
        raise HTTPException(status_code=401, detail="Invalid auth scheme")
    return token

# Layer 2: token validation (depends on layer 1)
def validate_token(token: str = Depends(extract_token)) -> dict:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")

# Layer 3: get actual user (depends on layer 2)
def get_current_user(payload: dict = Depends(validate_token)) -> User:
    user = db.get_user(payload["sub"])
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

# Layer 4: require active user (depends on layer 3)
def require_active_user(user: User = Depends(get_current_user)) -> User:
    if not user.is_active:
        raise HTTPException(status_code=403, detail="Account deactivated")
    return user

# Route only sees the final result
@app.get("/profile")
def get_profile(user: User = Depends(require_active_user)):
    return user  # all validation has already happened
```

```
Dependency resolution order:
  get_profile
      └── require_active_user
              └── get_current_user
                      └── validate_token
                              └── extract_token
                                      └── HTTP Header
```

### 1.6 Scoped & Shared Dependencies

FastAPI is smart about dependency caching — within a single request, a dependency is called **only once**, even if multiple things depend on it:

```python
call_count = 0

def expensive_dependency():
    global call_count
    call_count += 1
    print(f"Called {call_count} times")
    return {"data": "expensive result"}

def dep_a(shared = Depends(expensive_dependency)):
    return f"A got: {shared['data']}"

def dep_b(shared = Depends(expensive_dependency)):
    return f"B got: {shared['data']}"

@app.get("/test")
def test_route(
    a: str = Depends(dep_a),
    b: str = Depends(dep_b),
):
    # expensive_dependency is called ONCE, not twice.
    # FastAPI caches it for the duration of this request.
    return {"a": a, "b": b}
```

To **disable** caching and force re-evaluation:

```python
@app.get("/test")
def test_route(
    a: str = Depends(dep_a, use_cache=False),
    b: str = Depends(dep_b, use_cache=False),
):
    # Now expensive_dependency is called twice
    ...
```

### 1.7 Real-World Dependency Patterns

#### Request Context Object

A common pattern is building a rich context object that holds everything a handler needs:

```python
from dataclasses import dataclass

@dataclass
class RequestContext:
    user: User
    db: Session
    request_id: str
    pagination: PaginationParams

def get_request_context(
    user: User = Depends(require_active_user),
    db: Session = Depends(get_db),
    request: Request = None,
    pagination: PaginationParams = Depends(PaginationParams),
) -> RequestContext:
    return RequestContext(
        user=user,
        db=db,
        request_id=request.headers.get("X-Request-ID", str(uuid.uuid4())),
        pagination=pagination,
    )

@app.get("/feed")
def get_feed(ctx: RequestContext = Depends(get_request_context)):
    return fetch_feed(ctx.db, ctx.user, ctx.pagination)
```

#### Feature Flags

```python
from functools import lru_cache

@lru_cache()
def get_settings():
    return Settings()  # loaded once from env

def require_feature(feature_name: str):
    """Returns a dependency that checks if a feature flag is enabled."""
    def _check(settings: Settings = Depends(get_settings)):
        if not getattr(settings, f"enable_{feature_name}", False):
            raise HTTPException(status_code=404, detail="Feature not available")
    return _check

@app.post("/beta/ai-suggestions")
def ai_suggestions(
    _: None = Depends(require_feature("ai_suggestions")),
    user: User = Depends(get_current_user),
):
    ...
```

#### Rate Limiting

```python
from collections import defaultdict
import time

request_counts = defaultdict(list)

def rate_limit(max_requests: int = 100, window_seconds: int = 60):
    def _limiter(request: Request):
        client_ip = request.client.host
        now = time.time()

        # Remove old requests outside the window
        request_counts[client_ip] = [
            t for t in request_counts[client_ip]
            if now - t < window_seconds
        ]

        if len(request_counts[client_ip]) >= max_requests:
            raise HTTPException(
                status_code=429,
                detail=f"Rate limit exceeded: {max_requests} req/{window_seconds}s"
            )

        request_counts[client_ip].append(now)

    return _limiter

@app.get("/search")
def search(
    q: str,
    _: None = Depends(rate_limit(max_requests=10, window_seconds=60)),
):
    return perform_search(q)
```

---

## 2. Database Integration & ORM

### 2.1 Why Use an ORM?

An **ORM (Object-Relational Mapper)** lets you interact with your database using Python objects instead of raw SQL. It bridges the gap between the relational world (tables, rows, foreign keys) and the object-oriented world (classes, instances, relationships).

```
Without ORM (raw SQL):             With ORM (SQLAlchemy):
─────────────────────              ──────────────────────
cursor.execute("""                 user = User(
  INSERT INTO users                    name="Alice",
  (name, email, created_at)            email="alice@example.com"
  VALUES (?, ?, NOW())             )
""", ("Alice", "alice@..."))       db.add(user)
user_id = cursor.lastrowid         db.commit()
                                   # user.id is set automatically
```

**ORM benefits:**
- Database-agnostic code — switch from SQLite to PostgreSQL by changing one connection string.
- Automatic schema introspection, relationship loading, and type safety.
- Built-in query builder prevents SQL injection by design.
- Migration tooling (Alembic) tracks schema changes.

### 2.2 SQLAlchemy Core Concepts

SQLAlchemy has two layers:

```
┌─────────────────────────────────────────────────────┐
│                    ORM Layer                        │
│  (models, sessions, relationships, high-level API)  │
├─────────────────────────────────────────────────────┤
│                   Core Layer                        │
│  (Engine, Connection, SQL expression language)      │
├─────────────────────────────────────────────────────┤
│                   DBAPI Layer                       │
│  (psycopg2, asyncpg, sqlite3, pymysql, ...)         │
└─────────────────────────────────────────────────────┘
```

Key objects to know:

| Object | Purpose |
|--------|---------|
| `Engine` | The connection pool + dialect. Created once per app. |
| `Session` | Unit of work. Tracks objects, flushes changes to DB. |
| `Base` | Declarative base class — all models inherit from it. |
| `Model` | A Python class mapped to a database table. |
| `relationship()` | Declares how models relate to each other. |
| `Column` | A typed column definition on a model. |

### 2.3 Setting Up SQLAlchemy with FastAPI

**Install dependencies:**

```bash
pip install fastapi sqlalchemy psycopg2-binary alembic python-dotenv
# For async support:
pip install asyncpg sqlalchemy[asyncio]
```

**Project structure:**

```
myapp/
├── main.py
├── database.py       ← engine, session factory, Base
├── models/
│   ├── __init__.py
│   ├── user.py
│   └── post.py
├── schemas/          ← Pydantic models for request/response
│   ├── user.py
│   └── post.py
├── crud/             ← database operations
│   ├── user.py
│   └── post.py
├── routers/
│   ├── users.py
│   └── posts.py
└── alembic/          ← migration files
```

**`database.py` — the foundation:**

```python
# database.py
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
import os

# Connection URL format:
# postgresql://user:password@host:port/dbname
# sqlite:///./myapp.db  (relative path)
# sqlite:///:memory:   (in-memory, great for tests)

DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./myapp.db")

# The Engine is the gateway to the database.
# pool_pre_ping=True checks connections before using them (handles dropped connections).
# connect_args is SQLite-specific — disables same-thread check for use with FastAPI.
engine = create_engine(
    DATABASE_URL,
    pool_pre_ping=True,
    connect_args={"check_same_thread": False} if "sqlite" in DATABASE_URL else {},
)

# SessionLocal is a factory for creating database sessions.
# autocommit=False: changes must be explicitly committed.
# autoflush=False: prevents premature SQL emission.
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# All models inherit from Base.
# Base holds the metadata (table names, columns, etc.)
Base = declarative_base()

def create_tables():
    """Create all tables defined by models inheriting from Base."""
    Base.metadata.create_all(bind=engine)
```

### 2.4 Defining Models

```python
# models/user.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime, ForeignKey, Enum
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from database import Base
import enum

class UserRole(enum.Enum):
    admin = "admin"
    editor = "editor"
    viewer = "viewer"

class User(Base):
    __tablename__ = "users"

    # Primary key — auto-incremented integer
    id = Column(Integer, primary_key=True, index=True)

    # Unique constraints and indexes for query performance
    email = Column(String(255), unique=True, index=True, nullable=False)
    username = Column(String(100), unique=True, index=True, nullable=False)

    # Hashed password — NEVER store plain text
    hashed_password = Column(String(255), nullable=False)

    # Role using Python enum
    role = Column(Enum(UserRole), default=UserRole.viewer, nullable=False)

    is_active = Column(Boolean, default=True, nullable=False)
    is_verified = Column(Boolean, default=False, nullable=False)

    # Automatic timestamps using server-side defaults
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    # Relationships — SQLAlchemy loads related objects automatically
    posts = relationship("Post", back_populates="author", cascade="all, delete-orphan")
    profile = relationship("UserProfile", back_populates="user", uselist=False)

    def __repr__(self):
        return f"<User id={self.id} email={self.email!r}>"


# models/post.py
from sqlalchemy import Column, Integer, String, Text, Boolean, DateTime, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from database import Base

class Post(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(500), nullable=False)
    content = Column(Text, nullable=False)
    is_published = Column(Boolean, default=False)

    # Foreign key links to the users table
    author_id = Column(Integer, ForeignKey("users.id"), nullable=False)

    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    # Back-reference to the User model
    author = relationship("User", back_populates="posts")
    tags = relationship("Tag", secondary="post_tags", back_populates="posts")
```

**Pydantic schemas (separate from SQLAlchemy models):**

```python
# schemas/user.py
from pydantic import BaseModel, EmailStr
from datetime import datetime
from enum import Enum

class UserRole(str, Enum):
    admin = "admin"
    editor = "editor"
    viewer = "viewer"

# What clients send to CREATE a user
class UserCreate(BaseModel):
    email: EmailStr
    username: str
    password: str  # plain text — will be hashed before storage

# What clients send to UPDATE a user
class UserUpdate(BaseModel):
    username: str | None = None
    is_active: bool | None = None

# What the API returns — NEVER includes hashed_password
class UserResponse(BaseModel):
    id: int
    email: str
    username: str
    role: UserRole
    is_active: bool
    created_at: datetime

    class Config:
        from_attributes = True  # enables ORM mode (Pydantic v2)
```

### 2.5 The Database Dependency Pattern

This is the central pattern for database access in FastAPI — a generator dependency that manages the session lifecycle:

```python
# database.py (addition)
from typing import Generator
from sqlalchemy.orm import Session

def get_db() -> Generator[Session, None, None]:
    """
    Dependency that provides a SQLAlchemy session.

    Lifecycle:
    1. Create a new Session for this request.
    2. Yield it to the route handler.
    3. If an exception occurred: rollback any pending changes.
    4. Always close the session (returns connection to pool).
    """
    db = SessionLocal()
    try:
        yield db          # ← route handler runs here
        db.commit()       # commit if no exception was raised
    except Exception:
        db.rollback()     # rollback on any error
        raise
    finally:
        db.close()        # always close — return connection to pool
```

```python
# routers/users.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from database import get_db
from models.user import User
from schemas.user import UserCreate, UserResponse

router = APIRouter(prefix="/users", tags=["users"])

@router.post("/", response_model=UserResponse, status_code=201)
def create_user(
    user_data: UserCreate,
    db: Session = Depends(get_db),   # ← session injected here
):
    # Check for duplicate email
    existing = db.query(User).filter(User.email == user_data.email).first()
    if existing:
        raise HTTPException(status_code=409, detail="Email already registered")

    # Create and persist the user
    user = User(
        email=user_data.email,
        username=user_data.username,
        hashed_password=hash_password(user_data.password),
    )
    db.add(user)
    db.flush()   # flush to get the generated ID (without committing)
    return user  # session.commit() and session.close() happen in get_db()

@router.get("/{user_id}", response_model=UserResponse)
def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

### 2.6 CRUD Operations

A clean pattern is to separate CRUD logic from routing logic:

```python
# crud/user.py
from sqlalchemy.orm import Session
from sqlalchemy import or_
from models.user import User
from schemas.user import UserCreate, UserUpdate
from auth.utils import hash_password

def get_user_by_id(db: Session, user_id: int) -> User | None:
    return db.query(User).filter(User.id == user_id).first()

def get_user_by_email(db: Session, email: str) -> User | None:
    return db.query(User).filter(User.email == email).first()

def get_users(
    db: Session,
    *,
    offset: int = 0,
    limit: int = 20,
    search: str | None = None,
    active_only: bool = True,
) -> list[User]:
    query = db.query(User)

    if active_only:
        query = query.filter(User.is_active == True)

    if search:
        query = query.filter(
            or_(
                User.username.ilike(f"%{search}%"),
                User.email.ilike(f"%{search}%"),
            )
        )

    return query.order_by(User.created_at.desc()).offset(offset).limit(limit).all()

def create_user(db: Session, user_data: UserCreate) -> User:
    user = User(
        email=user_data.email,
        username=user_data.username,
        hashed_password=hash_password(user_data.password),
    )
    db.add(user)
    db.flush()
    db.refresh(user)  # reloads from DB — gets server-generated values like created_at
    return user

def update_user(db: Session, user: User, updates: UserUpdate) -> User:
    update_data = updates.model_dump(exclude_unset=True)  # only fields that were sent
    for field, value in update_data.items():
        setattr(user, field, value)
    db.flush()
    db.refresh(user)
    return user

def delete_user(db: Session, user: User) -> None:
    db.delete(user)
    db.flush()

def count_users(db: Session) -> int:
    return db.query(User).count()
```

### 2.7 Async SQLAlchemy

For high-throughput applications, async SQLAlchemy avoids blocking the event loop:

```python
# database_async.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase

# Note: postgresql+asyncpg:// instead of postgresql://
ASYNC_DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/mydb"

async_engine = create_async_engine(ASYNC_DATABASE_URL, pool_pre_ping=True, echo=False)

AsyncSessionLocal = async_sessionmaker(
    async_engine,
    class_=AsyncSession,
    autocommit=False,
    autoflush=False,
    expire_on_commit=False,  # important for async — prevents lazy loading after commit
)

class Base(DeclarativeBase):
    pass

# Async dependency
async def get_async_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise


# Usage in async routes
from sqlalchemy import select

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_async_db)):
    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

### 2.8 Alembic Migrations

Alembic tracks and applies schema changes over time — essential for production:

```bash
# Initialize Alembic in your project
alembic init alembic

# Create a migration (Alembic auto-detects model changes)
alembic revision --autogenerate -m "add users table"

# Apply pending migrations
alembic upgrade head

# Roll back one migration
alembic downgrade -1

# View migration history
alembic history --verbose
```

Configure `alembic/env.py` to use your models:

```python
# alembic/env.py (relevant section)
from database import Base
from models import user, post  # import all models so Alembic knows about them

target_metadata = Base.metadata
```

Example generated migration:

```python
# alembic/versions/abc123_add_users_table.py
def upgrade():
    op.create_table(
        "users",
        sa.Column("id", sa.Integer(), nullable=False),
        sa.Column("email", sa.String(255), nullable=False),
        sa.Column("hashed_password", sa.String(255), nullable=False),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.text("now()")),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint("email"),
    )
    op.create_index("ix_users_email", "users", ["email"])

def downgrade():
    op.drop_index("ix_users_email", table_name="users")
    op.drop_table("users")
```

### 2.9 ORM Alternatives

| Library | Type | Best For |
|---------|------|---------|
| **SQLAlchemy** | Full ORM + Core | Production apps needing power & flexibility |
| **SQLModel** | ORM (built on SQLAlchemy + Pydantic) | FastAPI-native, eliminates schema duplication |
| **Tortoise ORM** | Async ORM | Django-like, fully async from the ground up |
| **Databases** | Async query builder | Lightweight, raw SQL with async support |
| **Peewee** | Lightweight ORM | Small projects, simpler than SQLAlchemy |

**SQLModel** is worth highlighting for FastAPI apps — it unifies Pydantic models and SQLAlchemy models into one:

```python
# With SQLModel — one class serves as both DB model AND API schema
from sqlmodel import SQLModel, Field

class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    email: str = Field(unique=True, index=True)
    username: str
    is_active: bool = True

# This class can be used in both route signatures AND DB queries
# No more separate "UserResponse" / "UserCreate" schemas needed
```

---

## 3. Authentication & Authorization

### 3.1 OAuth2 with Password Flow (JWT)

**OAuth2 Password Flow** is the standard pattern for a first-party application (your own frontend + your own API) where the user gives their username and password directly to your app:

```
Client (browser/app)          Your FastAPI Server          Database
─────────────────            ──────────────────           ────────
    POST /token
    {username, password}  ──►  1. Lookup user by username
                               2. Verify password hash
                               3. Generate JWT access token ──► [users]
                          ◄──  {access_token, token_type}

    GET /profile
    Authorization: Bearer <token>
                          ──►  4. Decode & validate JWT
                               5. Extract user ID from claims
                               6. Fetch user from DB         ──► [users]
                          ◄──  {id, email, username, ...}
```

This flow is appropriate when:
- You own both the client and the server (not a third-party OAuth integration).
- Users are comfortable giving their credentials directly to your app.
- You want stateless authentication (no server-side session storage).

### 3.2 JWT Deep Dive

A **JSON Web Token (JWT)** is a compact, self-contained token that encodes claims about a user. It consists of three Base64URL-encoded parts separated by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   ← Header
.eyJzdWIiOiIxMjMiLCJlbWFpbCI6ImFsaWNlQGV4YW1wbGUuY29tIiwiZXhwIjoxNzAwMDAwMDAwfQ==
                                         ← Payload (claims)
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
                                         ← Signature
```

Decoded payload:
```json
{
  "sub": "123",
  "email": "alice@example.com",
  "role": "editor",
  "exp": 1700000000,
  "iat": 1699996400
}
```

**Standard claims:**

| Claim | Meaning | Notes |
|-------|---------|-------|
| `sub` | Subject (usually user ID) | Identifies WHO the token is about |
| `exp` | Expiration time (Unix timestamp) | Token is invalid after this time |
| `iat` | Issued at | When the token was created |
| `nbf` | Not before | Token invalid before this time |
| `jti` | JWT ID | Unique ID — used for revocation |

**Critical properties:**
- JWTs are **signed** but not encrypted. Anyone can decode the payload — never put secrets in claims.
- The **signature** guarantees the payload hasn't been tampered with.
- JWTs are **stateless** — the server needs no database lookup to validate them (just verify the signature).

### 3.3 Implementing the Full Auth System

**Install dependencies:**

```bash
pip install python-jose[cryptography] passlib[bcrypt] python-multipart
```

**Step 1: Configuration**

```python
# auth/config.py
from pydantic_settings import BaseSettings
from datetime import timedelta

class AuthSettings(BaseSettings):
    # Generate with: openssl rand -hex 32
    SECRET_KEY: str = "change-this-in-production-use-openssl-rand"
    ALGORITHM: str = "HS256"

    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7

    class Config:
        env_file = ".env"

auth_settings = AuthSettings()
```

**Step 2: Password hashing utilities**

```python
# auth/utils.py
from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta, timezone
from auth.config import auth_settings

# bcrypt is the gold standard for password hashing.
# It's intentionally slow (work factor) to defeat brute-force attacks.
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(plain_password: str) -> str:
    """Hash a plain-text password. Output includes salt and work factor."""
    return pwd_context.hash(plain_password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify a plain-text password against a stored hash."""
    return pwd_context.verify(plain_password, hashed_password)

def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    """Create a signed JWT access token."""
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + (
        expires_delta or timedelta(minutes=auth_settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    to_encode.update({"exp": expire, "iat": datetime.now(timezone.utc)})
    return jwt.encode(to_encode, auth_settings.SECRET_KEY, algorithm=auth_settings.ALGORITHM)

def create_refresh_token(user_id: int) -> str:
    """Create a long-lived refresh token."""
    data = {
        "sub": str(user_id),
        "type": "refresh",  # distinguish from access tokens
        "exp": datetime.now(timezone.utc) + timedelta(days=auth_settings.REFRESH_TOKEN_EXPIRE_DAYS),
    }
    return jwt.encode(data, auth_settings.SECRET_KEY, algorithm=auth_settings.ALGORITHM)

def decode_token(token: str) -> dict:
    """Decode and validate a JWT. Raises JWTError if invalid or expired."""
    return jwt.decode(token, auth_settings.SECRET_KEY, algorithms=[auth_settings.ALGORITHM])
```

**Step 3: Auth schemas**

```python
# schemas/auth.py
from pydantic import BaseModel

class Token(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int  # seconds

class TokenData(BaseModel):
    user_id: int
    email: str
    role: str

class LoginRequest(BaseModel):
    username: str
    password: str
```

**Step 4: Auth dependencies**

```python
# auth/dependencies.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session
from jose import JWTError
from database import get_db
from models.user import User
from auth.utils import decode_token

# Tells FastAPI where to find the token and documents it in the OpenAPI schema.
# The tokenUrl must match your login endpoint path.
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")

def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db),
) -> User:
    """
    Core auth dependency. Extracts user from JWT token.
    Raises 401 if token is missing, expired, or invalid.
    """
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},  # RFC 6750 compliance
    )

    try:
        payload = decode_token(token)
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = db.query(User).filter(User.id == int(user_id)).first()
    if user is None:
        raise credentials_exception

    return user

def get_current_active_user(
    current_user: User = Depends(get_current_user),
) -> User:
    """Extends get_current_user — also checks if the account is active."""
    if not current_user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Account is deactivated"
        )
    return current_user
```

**Step 5: Auth router**

```python
# routers/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session
from database import get_db
from models.user import User
from schemas.auth import Token
from auth.utils import verify_password, create_access_token, create_refresh_token
from auth.config import auth_settings

router = APIRouter(prefix="/auth", tags=["authentication"])

@router.post("/token", response_model=Token)
def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db),
):
    """
    OAuth2 Password Flow login endpoint.
    Accepts username + password, returns JWT tokens.
    OAuth2PasswordRequestForm expects application/x-www-form-urlencoded.
    """
    # Look up user (OAuth2 spec uses 'username' field)
    user = db.query(User).filter(User.email == form_data.username).first()

    # Use constant-time comparison to prevent timing attacks
    if not user or not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Account is deactivated"
        )

    # Embed the minimum necessary claims — not sensitive data
    access_token = create_access_token(
        data={"sub": str(user.id), "email": user.email, "role": user.role.value}
    )
    refresh_token = create_refresh_token(user_id=user.id)

    return Token(
        access_token=access_token,
        refresh_token=refresh_token,
        expires_in=auth_settings.ACCESS_TOKEN_EXPIRE_MINUTES * 60,
    )

@router.post("/refresh", response_model=Token)
def refresh_tokens(refresh_token: str, db: Session = Depends(get_db)):
    """Exchange a valid refresh token for a new access token."""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Invalid or expired refresh token"
    )

    try:
        payload = decode_token(refresh_token)
        if payload.get("type") != "refresh":
            raise credentials_exception
        user_id = int(payload["sub"])
    except Exception:
        raise credentials_exception

    user = db.query(User).filter(User.id == user_id, User.is_active == True).first()
    if not user:
        raise credentials_exception

    new_access_token = create_access_token(
        data={"sub": str(user.id), "email": user.email, "role": user.role.value}
    )
    new_refresh_token = create_refresh_token(user_id=user.id)

    return Token(
        access_token=new_access_token,
        refresh_token=new_refresh_token,
        expires_in=auth_settings.ACCESS_TOKEN_EXPIRE_MINUTES * 60,
    )

@router.get("/me")
def get_me(current_user: User = Depends(get_current_active_user)):
    """Return the currently authenticated user's profile."""
    return current_user
```

**Step 6: Wire it all together**

```python
# main.py
from fastapi import FastAPI
from database import create_tables
from routers import auth, users, posts

app = FastAPI(title="My API", version="1.0.0")

@app.on_event("startup")
def startup():
    create_tables()

app.include_router(auth.router)
app.include_router(users.router)
app.include_router(posts.router)
```

### 3.4 Role-Based Access Control (RBAC)

RBAC restricts what actions a user can perform based on their assigned role.

**The dependency approach — clean and composable:**

```python
# auth/rbac.py
from fastapi import Depends, HTTPException, status
from models.user import User, UserRole
from auth.dependencies import get_current_active_user
from functools import wraps
from typing import Callable

def require_role(*roles: UserRole) -> Callable:
    """
    Factory that returns a dependency requiring the user to have one of the given roles.

    Usage:
        @app.delete("/users/{id}")
        def delete_user(user: User = Depends(require_role(UserRole.admin))):
            ...
    """
    def _check_role(current_user: User = Depends(get_current_active_user)) -> User:
        if current_user.role not in roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Requires one of: {[r.value for r in roles]}"
            )
        return current_user
    return _check_role

# ── Pre-built convenience dependencies ───────────────────────────────────────
require_admin  = require_role(UserRole.admin)
require_editor = require_role(UserRole.admin, UserRole.editor)
require_viewer = require_role(UserRole.admin, UserRole.editor, UserRole.viewer)
```

**Using RBAC in routes:**

```python
# routers/admin.py
from fastapi import APIRouter, Depends
from auth.rbac import require_admin, require_editor
from models.user import User

router = APIRouter(prefix="/admin", tags=["admin"])

@router.get("/dashboard")
def admin_dashboard(admin: User = Depends(require_admin)):
    return {"message": f"Welcome, {admin.username}!", "stats": get_platform_stats()}

@router.delete("/users/{user_id}")
def delete_user(user_id: int, admin: User = Depends(require_admin)):
    # Only admins can delete users
    delete_user_by_id(user_id)
    return {"message": "User deleted"}

@router.post("/posts/{post_id}/publish")
def publish_post(post_id: int, editor: User = Depends(require_editor)):
    # Admins AND editors can publish posts
    publish_post_by_id(post_id)
    return {"message": "Post published"}
```

**Resource ownership checks (user can only edit their own content):**

```python
def get_own_post_or_admin(
    post_id: int,
    current_user: User = Depends(get_current_active_user),
    db: Session = Depends(get_db),
) -> Post:
    """Allow access if user owns the post OR is an admin."""
    post = db.query(Post).filter(Post.id == post_id).first()
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")

    is_owner = post.author_id == current_user.id
    is_admin = current_user.role == UserRole.admin

    if not (is_owner or is_admin):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="You don't have permission to modify this post"
        )

    return post

@router.put("/posts/{post_id}")
def update_post(
    update_data: PostUpdate,
    post: Post = Depends(get_own_post_or_admin),
    db: Session = Depends(get_db),
):
    # post is already verified as accessible to current_user
    return update_post_in_db(db, post, update_data)
```

**Permission-based RBAC (more granular than role-based):**

```python
# For fine-grained control, define explicit permissions per role
from enum import Enum

class Permission(str, Enum):
    read_users   = "read:users"
    write_users  = "write:users"
    delete_users = "delete:users"
    publish_posts = "publish:posts"
    manage_system = "manage:system"

ROLE_PERMISSIONS: dict[UserRole, set[Permission]] = {
    UserRole.viewer: {
        Permission.read_users,
    },
    UserRole.editor: {
        Permission.read_users,
        Permission.publish_posts,
    },
    UserRole.admin: set(Permission),  # all permissions
}

def require_permission(permission: Permission):
    def _check(current_user: User = Depends(get_current_active_user)) -> User:
        user_permissions = ROLE_PERMISSIONS.get(current_user.role, set())
        if permission not in user_permissions:
            raise HTTPException(
                status_code=403,
                detail=f"Missing required permission: {permission.value}"
            )
        return current_user
    return _check

@router.delete("/users/{user_id}")
def delete_user(
    user_id: int,
    _: User = Depends(require_permission(Permission.delete_users)),
):
    ...
```

### 3.5 Security Best Practices

#### Password Hashing

```python
# ✅ CORRECT: Use bcrypt via passlib
from passlib.context import CryptContext

pwd_context = CryptContext(
    schemes=["bcrypt"],
    deprecated="auto",
    bcrypt__rounds=12  # work factor — increase for more security (slower)
)

# ❌ NEVER do any of these:
import hashlib
hashlib.md5(password.encode()).hexdigest()    # MD5 — catastrophically broken
hashlib.sha256(password.encode()).hexdigest() # SHA256 — not a password hash
password                                      # plain text — obviously never
```

Why bcrypt? It's deliberately slow and includes a random salt, which means:
- Pre-computed rainbow tables are useless (each hash is unique due to the salt).
- Brute-force attacks take much longer (tunable work factor).
- Future-proof — you can increase the work factor as hardware improves.

#### Token Security

```python
# ✅ Short-lived access tokens + long-lived refresh tokens
ACCESS_TOKEN_EXPIRE_MINUTES = 15   # 15 minutes — aggressive but secure
REFRESH_TOKEN_EXPIRE_DAYS = 7      # 7 days — user needs to re-login weekly

# ✅ Store refresh tokens in the database for revocation capability
class RefreshToken(Base):
    __tablename__ = "refresh_tokens"
    id = Column(Integer, primary_key=True)
    token_hash = Column(String(255), unique=True, index=True)  # store hash, not the token
    user_id = Column(Integer, ForeignKey("users.id"))
    expires_at = Column(DateTime(timezone=True))
    is_revoked = Column(Boolean, default=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())

# ✅ On logout: revoke the refresh token
@router.post("/auth/logout")
def logout(
    refresh_token: str,
    current_user: User = Depends(get_current_active_user),
    db: Session = Depends(get_db),
):
    token_hash = hashlib.sha256(refresh_token.encode()).hexdigest()
    db_token = db.query(RefreshToken).filter(
        RefreshToken.token_hash == token_hash,
        RefreshToken.user_id == current_user.id,
    ).first()
    if db_token:
        db_token.is_revoked = True
    return {"message": "Logged out successfully"}
```

#### Environment-Based Secrets

```python
# ✅ Load secrets from environment variables, never hardcode
# .env file (NOT committed to version control — add to .gitignore)
SECRET_KEY=your-256-bit-random-key-generated-with-openssl-rand-hex-32
DATABASE_URL=postgresql://user:password@localhost/mydb

# In Python:
import os
SECRET_KEY = os.environ["SECRET_KEY"]  # fails loudly if not set

# Or with pydantic-settings (recommended):
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    SECRET_KEY: str          # required — no default
    DATABASE_URL: str        # required
    DEBUG: bool = False      # optional with default

    class Config:
        env_file = ".env"

settings = Settings()
```

#### HTTPS & Secure Headers

```python
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI()

# Redirect HTTP to HTTPS (use in production)
app.add_middleware(HTTPSRedirectMiddleware)

# Prevent host header injection
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["yourdomain.com", "www.yourdomain.com"]
)

# Add security headers
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Content-Security-Policy"] = "default-src 'self'"
        return response

app.add_middleware(SecurityHeadersMiddleware)
```

#### Rate Limiting Authentication Endpoints

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@router.post("/auth/token")
@limiter.limit("5/minute")  # max 5 login attempts per minute per IP
def login(request: Request, form_data: OAuth2PasswordRequestForm = Depends(), ...):
    ...
```

#### Preventing Timing Attacks

```python
# ❌ VULNERABLE: short-circuits if user doesn't exist
# An attacker can measure response time to enumerate valid usernames
def login_vulnerable(email: str, password: str, db: Session):
    user = db.query(User).filter(User.email == email).first()
    if not user:
        raise HTTPException(status_code=401)   # fast — user not found
    if not verify_password(password, user.hashed_password):
        raise HTTPException(status_code=401)   # slow — bcrypt ran

# ✅ SAFE: always run bcrypt, regardless of whether user exists
DUMMY_HASH = hash_password("dummy-password-to-prevent-timing-attack")

def login_safe(email: str, password: str, db: Session):
    user = db.query(User).filter(User.email == email).first()
    hash_to_verify = user.hashed_password if user else DUMMY_HASH
    password_valid = verify_password(password, hash_to_verify)
    if not user or not password_valid:
        raise HTTPException(status_code=401)   # always takes the same time
```

#### Input Validation

```python
from pydantic import BaseModel, validator, EmailStr, constr

class UserCreate(BaseModel):
    email: EmailStr  # Pydantic validates email format automatically
    username: constr(min_length=3, max_length=50, pattern=r"^[a-zA-Z0-9_]+$")
    password: constr(min_length=8, max_length=128)

    @validator("password")
    def password_strength(cls, v):
        if not any(c.isupper() for c in v):
            raise ValueError("Password must contain at least one uppercase letter")
        if not any(c.isdigit() for c in v):
            raise ValueError("Password must contain at least one digit")
        return v
```

#### Security Checklist

```
Authentication
  ✅ Passwords hashed with bcrypt (work factor ≥ 12)
  ✅ JWT tokens signed with HS256 or RS256
  ✅ Access tokens expire in 15-30 minutes
  ✅ Refresh tokens stored (hashed) in DB for revocation
  ✅ Logout revokes refresh token

Authorization
  ✅ Every protected endpoint uses a dependency
  ✅ Resource ownership verified before modification
  ✅ Admin endpoints explicitly require admin role

Transport & Infrastructure
  ✅ HTTPS enforced in production
  ✅ Security headers set (HSTS, CSP, X-Frame-Options)
  ✅ CORS configured with explicit allowed origins
  ✅ Rate limiting on login, register, and password reset endpoints

Secrets & Config
  ✅ SECRET_KEY is random, 256-bit, stored in environment variable
  ✅ DATABASE_URL not hardcoded
  ✅ .env file is in .gitignore
  ✅ Different secrets for dev/staging/production

Code & Data
  ✅ No sensitive data in JWT payload (no passwords, credit cards)
  ✅ SQL queries use ORM or parameterized statements (no string interpolation)
  ✅ Input validated with Pydantic before DB interaction
  ✅ Error messages don't leak internal details (stack traces, SQL errors)
  ✅ Timing-safe password comparison (always run bcrypt)
```

---

## Putting It All Together

Here's how the three pillars — DI, database, and auth — compose in a real endpoint:

```python
@router.put("/posts/{post_id}", response_model=PostResponse)
async def update_post(
    post_id: int,
    update_data: PostUpdate,                           # ← validated by Pydantic
    current_user: User = Depends(get_current_active_user),  # ← auth via DI
    db: Session = Depends(get_db),                     # ← db session via DI
):
    """
    Update a post. User must be authenticated and either
    the post author or an admin.
    """
    # 1. Fetch resource
    post = db.query(Post).filter(Post.id == post_id).first()
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")

    # 2. Authorization check
    if post.author_id != current_user.id and current_user.role != UserRole.admin:
        raise HTTPException(status_code=403, detail="Permission denied")

    # 3. Apply updates
    for field, value in update_data.model_dump(exclude_unset=True).items():
        setattr(post, field, value)

    # 4. Commit happens automatically in get_db()
    db.flush()
    db.refresh(post)

    return post
```

Each concern is handled by a dedicated, testable component:
- **Pydantic** validates and parses input.
- **`get_db()`** manages the database session lifecycle.
- **`get_current_active_user()`** handles authentication end-to-end.
- The route handler itself focuses purely on **business logic**.

This separation makes each piece independently testable, swappable, and understandable.

---

*FastAPI documentation: https://fastapi.tiangolo.com*
*SQLAlchemy documentation: https://docs.sqlalchemy.org*
*JWT specification: https://datatracker.ietf.org/doc/html/rfc7519*
