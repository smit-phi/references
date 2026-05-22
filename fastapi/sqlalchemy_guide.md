# SQLAlchemy: The Complete Backend Developer's Guide

> Written for FastAPI developers coming from Django. Where Django's ORM is opinionated and batteries-included, SQLAlchemy is a **toolkit** — it gives you tremendous power and flexibility, but expects you to understand what's happening. That tradeoff is the lens through which to read this entire guide.

---

## Table of Contents

1. [The Big Picture: SQLAlchemy vs Django ORM](#1-the-big-picture)
2. [Architecture: Core vs ORM](#2-architecture-core-vs-orm)
3. [Installation & Project Setup](#3-installation--project-setup)
4. [SQLAlchemy Core](#4-sqlalchemy-core)
5. [SQLAlchemy ORM — Declarative Models](#5-sqlalchemy-orm--declarative-models)
6. [Engine, Connection, and Session](#6-engine-connection-and-session)
7. [CRUD with the ORM](#7-crud-with-the-orm)
8. [Relationships](#8-relationships)
9. [Querying — Legacy vs Modern Style](#9-querying--legacy-vs-modern-style)
10. [Migrations with Alembic](#10-migrations-with-alembic)
11. [Async SQLAlchemy](#11-async-sqlalchemy)
12. [FastAPI Integration Pattern](#12-fastapi-integration-pattern)
13. [The Unit of Work Pattern & Session Lifecycle](#13-the-unit-of-work-pattern--session-lifecycle)
14. [Tricky But Valuable Topics](#14-tricky-but-valuable-topics)
15. [Performance & Query Optimization](#15-performance--query-optimization)
16. [Quick Reference Cheatsheet](#16-quick-reference-cheatsheet)

---

## 1. The Big Picture

### Django ORM vs SQLAlchemy — Philosophy

| | Django ORM | SQLAlchemy |
|---|---|---|
| Design philosophy | Batteries included, opinionated | Toolkit, explicit |
| Entry point | `Model` class, just works | You configure Engine → Session → Model |
| Migrations | Built-in `makemigrations` | Separate tool: Alembic |
| Query style | `User.objects.filter(...)` | `select(User).where(...)` |
| Async support | Django 4.1+ (limited) | First-class via `asyncio` |
| Raw SQL | Possible but awkward | First-class citizen (Core layer) |
| Connection pooling | Managed for you | Fully configurable |
| Multiple databases | Via `using=` | Multiple engines, naturally |

### The Mental Model Shift

In Django, the ORM hides the database behind a clean ActiveRecord-style API. The model *is* the query interface.

In SQLAlchemy, the model is just a **Python class mapped to a table**. The querying is done separately, through a `Session` object. This separation — **Data Mapper pattern** vs Django's **ActiveRecord pattern** — is the most important conceptual shift.

```python
# Django — model IS the query interface
users = User.objects.filter(active=True).order_by('-created_at')

# SQLAlchemy — model is just a class; Session does the querying
stmt = select(User).where(User.active == True).order_by(User.created_at.desc())
users = session.scalars(stmt).all()
```

---

## 2. Architecture: Core vs ORM

SQLAlchemy has two distinct layers. Most developers use both.

```
┌─────────────────────────────────────────┐
│             Your Application            │
├─────────────────────────────────────────┤
│         SQLAlchemy ORM (Layer 2)        │
│  Mapped classes, Session, relationships │
├─────────────────────────────────────────┤
│        SQLAlchemy Core (Layer 1)        │
│  Engine, Connection, Table, select()    │
├─────────────────────────────────────────┤
│              DBAPI (PEP 249)            │
│   psycopg2, asyncpg, aiosqlite, etc.    │
├─────────────────────────────────────────┤
│              Database                   │
│   PostgreSQL, MySQL, SQLite, etc.       │
└─────────────────────────────────────────┘
```

**SQLAlchemy Core** deals with:
- Engines and connections
- Table definitions as schema objects
- SQL expression language (`select`, `insert`, `update`, `delete`)
- Transactions

**SQLAlchemy ORM** adds on top:
- Python class-to-table mapping
- `Session` (the Unit of Work)
- Relationships and lazy/eager loading
- Identity map (same row = same Python object)

> **Django parallel:** Django only exposes the ORM layer. There is no "Django Core" that you interact with directly. SQLAlchemy's Core is like having access to Django's internals, but cleanly and intentionally.

---

## 3. Installation & Project Setup

```bash
pip install sqlalchemy        # Core + ORM
pip install psycopg2-binary   # PostgreSQL sync driver
pip install asyncpg           # PostgreSQL async driver
pip install aiosqlite         # SQLite async driver
pip install alembic           # Migrations
pip install greenlet          # Required for async ORM (auto-installed with sqlalchemy[asyncio])
```

Or the async bundle:
```bash
pip install "sqlalchemy[asyncio]" asyncpg
```

A typical project structure:

```
project/
├── app/
│   ├── db/
│   │   ├── __init__.py
│   │   ├── base.py        # declarative Base
│   │   ├── engine.py      # engine + sessionmaker
│   │   └── models/
│   │       ├── user.py
│   │       └── post.py
│   ├── routers/
│   └── main.py
├── alembic/
│   ├── env.py
│   └── versions/
└── alembic.ini
```

---

## 4. SQLAlchemy Core

Core is where you interact with the database at the SQL expression level. You rarely use it directly in application code, but understanding it makes the ORM make sense.

### Defining Tables (Core style)

```python
from sqlalchemy import MetaData, Table, Column, Integer, String, ForeignKey, DateTime
from datetime import datetime, timezone

metadata = MetaData()

users_table = Table(
    "users",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("username", String(50), nullable=False, unique=True),
    Column("email", String(255), nullable=False),
    Column("created_at", DateTime, default=datetime.now(timezone.utc)),
)

posts_table = Table(
    "posts",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("title", String(200), nullable=False),
    Column("user_id", Integer, ForeignKey("users.id"), nullable=False),
)
```

> **Django parallel:** This is like writing `CREATE TABLE` SQL by hand, but in Python. Django generates this automatically from `Model` classes. In SQLAlchemy you'll usually use ORM-style models instead of raw `Table` objects, but the underlying `Table` object always exists — the ORM just generates it for you.

### Core SQL Expressions

```python
from sqlalchemy import select, insert, update, delete, and_, or_, text

# SELECT
stmt = select(users_table).where(users_table.c.username == "alice")

# INSERT
stmt = insert(users_table).values(username="alice", email="alice@example.com")

# UPDATE
stmt = (
    update(users_table)
    .where(users_table.c.id == 1)
    .values(email="new@example.com")
)

# DELETE
stmt = delete(users_table).where(users_table.c.id == 1)

# Raw SQL (escape hatch — always use text() to avoid injection)
stmt = text("SELECT * FROM users WHERE username = :name").bindparams(name="alice")
```

### Executing Core Statements

```python
from sqlalchemy import create_engine

engine = create_engine("postgresql+psycopg2://user:pass@localhost/mydb")

with engine.connect() as conn:
    result = conn.execute(select(users_table))
    for row in result:
        print(row.username, row.email)

    # Writes need explicit commit
    conn.execute(insert(users_table).values(username="bob", email="bob@example.com"))
    conn.commit()
```

---

## 5. SQLAlchemy ORM — Declarative Models

This is where most application code lives. Modern SQLAlchemy (2.0+) uses `DeclarativeBase`.

### Defining the Base

```python
# db/base.py
from sqlalchemy.orm import DeclarativeBase, MappedColumn, mapped_column
from sqlalchemy import Integer

class Base(DeclarativeBase):
    pass
```

### Defining Models

```python
# db/models/user.py
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy import String, Boolean, DateTime
from datetime import datetime, timezone
from app.db.base import Base


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, nullable=False)
    email: Mapped[str] = mapped_column(String(255), nullable=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc)
    )

    # Relationship (covered later)
    posts: Mapped[list["Post"]] = relationship(back_populates="author")

    def __repr__(self) -> str:
        return f"<User id={self.id} username={self.username!r}>"
```

### The Modern `Mapped` + `mapped_column` Syntax (SQLAlchemy 2.0+)

This is a crucial thing to get right. SQLAlchemy 2.0 introduced type-annotated column declarations that make models much cleaner and give you IDE support.

```python
# OLD style (SQLAlchemy 1.x — still works but avoid for new code)
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(50))

# NEW style (SQLAlchemy 2.0+ — prefer this)
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    bio: Mapped[str | None] = mapped_column(String(500))  # nullable
```

The `Mapped[T]` annotation is not just a type hint — SQLAlchemy reads it at class creation time. `Mapped[str]` means NOT NULL. `Mapped[str | None]` means nullable. This is much more Pythonic than the old explicit `nullable=False`.

> **Django parallel:** Django's `CharField(max_length=50)` handles both the Python type and the DB column in one declaration. SQLAlchemy 2.0 achieves similar expressiveness with `Mapped[str] = mapped_column(String(50))`. The difference is that `Mapped` carries real Python typing that mypy/pyright understands.

### Column Types Reference

```python
from sqlalchemy import (
    Integer, BigInteger, SmallInteger,
    String, Text, Unicode,
    Float, Numeric, Double,
    Boolean,
    Date, DateTime, Time, Interval,
    JSON, JSONB,          # JSONB is PostgreSQL-specific
    Enum,
    UUID,                 # from sqlalchemy import UUID
    LargeBinary,
    ARRAY,                # PostgreSQL arrays
)

class Product(Base):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text)
    price: Mapped[float] = mapped_column(Numeric(10, 2))  # 10 digits, 2 decimal
    tags: Mapped[list[str] | None] = mapped_column(ARRAY(String))  # PostgreSQL
    metadata_: Mapped[dict | None] = mapped_column(JSONB)
    status: Mapped[str] = mapped_column(
        Enum("active", "inactive", "pending", name="product_status"),
        default="active"
    )
```

### Server Defaults vs Python Defaults

This is one of the trickier parts when coming from Django.

```python
from sqlalchemy import func, text
from sqlalchemy.orm import mapped_column
from sqlalchemy import DateTime, Integer

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)

    # Python-side default — runs in Python before INSERT
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc)   # called each time
    )

    # Server-side default — the DB sets this, Python doesn't touch it
    # The value won't be populated in your object until you refresh/expire it
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now()          # DB function
    )

    # onupdate — Python-side, fires on UPDATE
    modified_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True),
        onupdate=lambda: datetime.now(timezone.utc)
    )
```

> **Tricky:** When using `server_default`, SQLAlchemy does NOT populate the value on your Python object after insert. You need to call `session.refresh(obj)` or access it after a commit (which expires the object, forcing a reload on next access).

---

## 6. Engine, Connection, and Session

This is the most different part from Django. Django manages connections for you. SQLAlchemy makes you understand the plumbing.

### The Engine

The `Engine` is the factory for database connections. Create it once at startup.

```python
# db/engine.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# Connection URL format:
# dialect+driver://username:password@host:port/database
DATABASE_URL = "postgresql+psycopg2://postgres:password@localhost:5432/myapp"

engine = create_engine(
    DATABASE_URL,
    pool_size=10,           # Number of connections to keep open
    max_overflow=20,        # Extra connections beyond pool_size under load
    pool_timeout=30,        # Seconds to wait for a connection from pool
    pool_recycle=1800,      # Recycle connections after 30 min (avoids stale connections)
    echo=False,             # Set True to log all SQL (great for debugging)
)
```

> **Django parallel:** Django's `DATABASES` setting is analogous. Django manages the pool internally (via the database backend). SQLAlchemy's `create_engine` is where you configure the pool explicitly.

### The Session

The `Session` is SQLAlchemy's **Unit of Work**. It tracks all the objects you've loaded or created, and flushes changes to the database as a coherent unit.

```python
from sqlalchemy.orm import sessionmaker, Session

SessionLocal = sessionmaker(
    bind=engine,
    autocommit=False,   # Never use autocommit=True
    autoflush=True,     # Flush before queries (usually what you want)
    expire_on_commit=True,  # Objects expire after commit (forces reload on access)
)

# Using a session
def get_users():
    with SessionLocal() as session:
        users = session.scalars(select(User)).all()
        return users
```

> **Django parallel:** The `Session` is somewhat like Django's `QuerySet` execution context, but more explicit. In Django, every `.save()` hits the DB immediately. In SQLAlchemy, changes are **tracked in memory** and only written when you `flush()` or `commit()`.

### Session Lifecycle (Critical to Understand)

```
New Object      →   session.add(obj)    →   Pending
Pending         →   session.flush()     →   Persistent (in DB, in session)
Persistent      →   session.commit()    →   Persistent (expires attributes)
Persistent      →   session.expunge()  →   Detached
Persistent/Det. →   session.close()     →   Detached or garbage collected
```

```python
with SessionLocal() as session:
    user = User(username="alice", email="alice@example.com")
    # user is "transient" — not connected to any session

    session.add(user)
    # user is "pending" — tracked by session, not yet in DB

    session.flush()
    # user is "persistent" — written to DB, but transaction not committed
    # user.id is now populated

    session.commit()
    # transaction committed, user.id is saved, user attributes are "expired"
    # next access to user.username will trigger a SELECT (lazy reload)

    print(user.username)  # triggers SELECT to reload because attributes expired
```

---

## 7. CRUD with the ORM

### Create

```python
def create_user(session: Session, username: str, email: str) -> User:
    user = User(username=username, email=email)
    session.add(user)
    session.commit()
    session.refresh(user)   # reload from DB to get server defaults
    return user

# Bulk insert — much faster than adding one by one
def bulk_create_users(session: Session, users_data: list[dict]) -> None:
    session.execute(insert(User), users_data)
    session.commit()
```

### Read

```python
from sqlalchemy import select

# Get by primary key (uses identity map — won't re-query if already loaded)
user = session.get(User, 1)

# SELECT with conditions
stmt = select(User).where(User.is_active == True)
users = session.scalars(stmt).all()

# Single result (raises if not found or multiple)
user = session.scalars(select(User).where(User.id == 1)).one()
user = session.scalars(select(User).where(User.id == 1)).one_or_none()
user = session.scalars(select(User).where(User.id == 1)).first()  # None if not found
```

### Update

```python
# Method 1: Mutate the object (recommended for single objects)
user = session.get(User, 1)
user.email = "new@example.com"
session.commit()  # SQLAlchemy detects the change and generates UPDATE

# Method 2: Bulk update with Core expression (recommended for many rows)
session.execute(
    update(User)
    .where(User.is_active == False)
    .values(is_active=True)
)
session.commit()
```

### Delete

```python
# Method 1: Delete a loaded object
user = session.get(User, 1)
session.delete(user)
session.commit()

# Method 2: Bulk delete
session.execute(delete(User).where(User.created_at < some_date))
session.commit()
```

> **Django parallel:**
> ```python
> # Django
> user.save()          # UPDATE
> user.delete()        # DELETE
> User.objects.filter(...).update(...)  # bulk UPDATE
> User.objects.filter(...).delete()    # bulk DELETE
> ```
> The pattern is the same, but SQLAlchemy separates "track a change" (mutation) from "write the change" (commit).

---

## 8. Relationships

### ForeignKey and relationship()

```python
from sqlalchemy.orm import relationship, Mapped, mapped_column
from sqlalchemy import ForeignKey

class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    body: Mapped[str] = mapped_column(Text)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))

    author: Mapped["User"] = relationship(back_populates="posts")


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(50))

    posts: Mapped[list["Post"]] = relationship(back_populates="author")
```

> **Django parallel:** `ForeignKey` is the same concept. `relationship()` is like Django's reverse accessor (`user.post_set`) but you define it explicitly on both sides with `back_populates`. Django generates the reverse relation automatically.

### Relationship Loading Strategies

This is one of the most important things to understand to avoid N+1 queries. By default, SQLAlchemy uses **lazy loading** — accessing a relationship triggers a new SELECT.

```python
from sqlalchemy.orm import selectinload, joinedload, subqueryload, lazyload, noload

# LAZY LOADING (default) — fires a new SELECT when you access the relationship
users = session.scalars(select(User)).all()
for user in users:
    print(user.posts)  # SELECT fired N times! Classic N+1 problem.

# JOINED LOAD — single LEFT OUTER JOIN query
# Good for many-to-one / one-to-one (e.g., post.author)
stmt = select(Post).options(joinedload(Post.author))
posts = session.scalars(stmt).all()

# SELECTIN LOAD — two queries: one for parent, one IN clause for children
# Good for one-to-many (e.g., user.posts) — avoids cartesian product
stmt = select(User).options(selectinload(User.posts))
users = session.scalars(stmt).all()

# SUBQUERY LOAD — uses a correlated subquery
# Old approach, selectinload is usually better
stmt = select(User).options(subqueryload(User.posts))

# NO LOAD — explicitly don't load, raises error if accessed
stmt = select(User).options(noload(User.posts))

# Nested eager loading
stmt = (
    select(User)
    .options(
        selectinload(User.posts)
        .selectinload(Post.comments)
    )
)
```

> **Django parallel:** Django's `select_related()` = `joinedload`. Django's `prefetch_related()` = `selectinload`. The same N+1 problem exists in Django — SQLAlchemy just makes the loading strategy more explicit.

### Many-to-Many

```python
from sqlalchemy import Table, Column, Integer, ForeignKey

# Association table (no model needed for pure many-to-many)
post_tags = Table(
    "post_tags",
    Base.metadata,
    Column("post_id", Integer, ForeignKey("posts.id"), primary_key=True),
    Column("tag_id", Integer, ForeignKey("tags.id"), primary_key=True),
)

class Tag(Base):
    __tablename__ = "tags"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)
    posts: Mapped[list["Post"]] = relationship(secondary=post_tags, back_populates="tags")

class Post(Base):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    tags: Mapped[list[Tag]] = relationship(secondary=post_tags, back_populates="posts")

# Usage
post = session.get(Post, 1)
tag = Tag(name="python")
post.tags.append(tag)
session.commit()
```

For many-to-many with **extra data on the association** (e.g., a `role` field on the join), use an **association object**:

```python
class UserGroup(Base):
    __tablename__ = "user_groups"
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), primary_key=True)
    group_id: Mapped[int] = mapped_column(ForeignKey("groups.id"), primary_key=True)
    role: Mapped[str] = mapped_column(String(50))   # extra data

    user: Mapped["User"] = relationship(back_populates="group_associations")
    group: Mapped["Group"] = relationship(back_populates="user_associations")
```

---

## 9. Querying — Legacy vs Modern Style

SQLAlchemy has **two query APIs**. You will encounter both in the wild.

### Legacy (1.x) Query API — Still Works, But Avoid for New Code

```python
# The old way
users = session.query(User).filter(User.is_active == True).all()
user = session.query(User).filter_by(username="alice").first()
count = session.query(User).count()
```

### Modern (2.0) `select()` API — Use This

```python
from sqlalchemy import select, func, and_, or_, not_

# Basic
stmt = select(User).where(User.is_active == True)
users = session.scalars(stmt).all()

# Multiple conditions
stmt = select(User).where(
    and_(User.is_active == True, User.created_at > some_date)
)

# OR condition
stmt = select(User).where(
    or_(User.username == "alice", User.username == "bob")
)

# IN clause
stmt = select(User).where(User.id.in_([1, 2, 3]))

# NOT IN
stmt = select(User).where(User.id.not_in([1, 2, 3]))

# LIKE
stmt = select(User).where(User.username.like("ali%"))
stmt = select(User).where(User.username.ilike("ali%"))  # case-insensitive

# IS NULL / IS NOT NULL
stmt = select(User).where(User.deleted_at.is_(None))
stmt = select(User).where(User.deleted_at.is_not(None))

# ORDER BY
stmt = select(User).order_by(User.created_at.desc())
stmt = select(User).order_by(User.username.asc())

# LIMIT / OFFSET
stmt = select(User).limit(10).offset(20)

# COUNT
count = session.scalar(select(func.count()).select_from(User))

# Aggregate functions
stmt = select(func.max(User.id), func.min(User.id), func.avg(User.id))

# Distinct
stmt = select(User.email).distinct()

# Exists
from sqlalchemy import exists
stmt = select(User).where(
    exists(select(Post).where(Post.user_id == User.id))
)
```

### Joins

```python
# INNER JOIN (explicit)
stmt = (
    select(Post, User)
    .join(User, Post.user_id == User.id)
)

# INNER JOIN using relationship
stmt = select(Post).join(Post.author)

# LEFT OUTER JOIN
stmt = select(User).outerjoin(User.posts)

# JOIN and filter
stmt = (
    select(User)
    .join(User.posts)
    .where(Post.title.like("%python%"))
)

# Join multiple tables
stmt = (
    select(Post)
    .join(Post.author)
    .join(Post.tags)
    .where(Tag.name == "python")
    .where(User.is_active == True)
)
```

### Subqueries

```python
from sqlalchemy import subquery

# Scalar subquery
subq = select(func.count(Post.id)).where(Post.user_id == User.id).scalar_subquery()
stmt = select(User, subq.label("post_count"))

# Subquery in WHERE
active_user_ids = select(User.id).where(User.is_active == True).subquery()
stmt = select(Post).where(Post.user_id.in_(select(active_user_ids.c.id)))
```

---

## 10. Migrations with Alembic

Django has `makemigrations` + `migrate`. SQLAlchemy uses **Alembic**, a separate but official tool.

### Setup

```bash
alembic init alembic
```

Edit `alembic.ini` to set your database URL, or better — edit `alembic/env.py` to pull from your config:

```python
# alembic/env.py
from app.db.base import Base
from app.db.engine import DATABASE_URL
import app.db.models  # import all models so they're registered on Base.metadata

config.set_main_option("sqlalchemy.url", DATABASE_URL)
target_metadata = Base.metadata
```

### Common Commands

```bash
# Create a migration (auto-detect changes from models — like Django's makemigrations)
alembic revision --autogenerate -m "add users table"

# Apply migrations (like Django's migrate)
alembic upgrade head

# Downgrade one step
alembic downgrade -1

# View history
alembic history

# View current revision
alembic current

# Upgrade to specific revision
alembic upgrade abc123
```

### A Generated Migration File

```python
# alembic/versions/abc123_add_users_table.py

def upgrade() -> None:
    op.create_table(
        "users",
        sa.Column("id", sa.Integer(), nullable=False),
        sa.Column("username", sa.String(length=50), nullable=False),
        sa.Column("email", sa.String(length=255), nullable=False),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint("username"),
    )

def downgrade() -> None:
    op.drop_table("users")
```

> **Important:** Always review auto-generated migrations. Alembic compares your model metadata to the DB state — it can detect column additions, removals, and type changes, but **cannot detect** renames (it sees a drop + add). Always check before applying to production.

---

## 11. Async SQLAlchemy

This is where FastAPI developers need to pay close attention. Async SQLAlchemy uses a completely different set of objects.

### Sync vs Async — The Object Map

| Sync | Async | Purpose |
|------|-------|---------|
| `create_engine` | `create_async_engine` | Creates the engine |
| `Engine` | `AsyncEngine` | Engine object |
| `Session` | `AsyncSession` | The session |
| `sessionmaker` | `async_sessionmaker` | Session factory |
| `Connection` | `AsyncConnection` | Low-level connection |

### Async Engine Setup

```python
# db/engine.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

# Note: driver is asyncpg, not psycopg2
DATABASE_URL = "postgresql+asyncpg://postgres:password@localhost:5432/myapp"

async_engine = create_async_engine(
    DATABASE_URL,
    pool_size=10,
    max_overflow=20,
    echo=False,
)

AsyncSessionLocal = async_sessionmaker(
    bind=async_engine,
    class_=AsyncSession,
    expire_on_commit=False,  # Important for async — explained below
)
```

### Async CRUD

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

# Create
async def create_user(session: AsyncSession, username: str, email: str) -> User:
    user = User(username=username, email=email)
    session.add(user)
    await session.commit()
    await session.refresh(user)
    return user

# Read
async def get_user(session: AsyncSession, user_id: int) -> User | None:
    return await session.get(User, user_id)

async def list_active_users(session: AsyncSession) -> list[User]:
    result = await session.execute(select(User).where(User.is_active == True))
    return result.scalars().all()

# Update
async def deactivate_user(session: AsyncSession, user_id: int) -> None:
    user = await session.get(User, user_id)
    if user:
        user.is_active = False
        await session.commit()

# Delete
async def delete_user(session: AsyncSession, user_id: int) -> None:
    user = await session.get(User, user_id)
    if user:
        await session.delete(user)
        await session.commit()
```

### Why `expire_on_commit=False` for Async

In sync SQLAlchemy, when you commit, objects are "expired" — next access triggers a SELECT. In async context, if you return an ORM object after a commit and try to access its attributes **outside the session context**, SQLAlchemy would need to fire an async SELECT — but there's no session anymore, so it fails.

```python
# Problem scenario
async def create_user(session: AsyncSession, ...) -> User:
    user = User(...)
    session.add(user)
    await session.commit()
    # With expire_on_commit=True (default), user.id is now "expired"
    return user  # caller accesses user.id → needs a SELECT → but session might be closed!

# Solution 1: expire_on_commit=False (most common)
# Solution 2: await session.refresh(user) before returning
```

### Async Relationships — Lazy Loading Doesn't Work!

This is the **biggest gotcha** in async SQLAlchemy. Lazy loading fires a synchronous SQL query under the hood. In async mode, this raises an error.

```python
# THIS WILL RAISE MissingGreenlet error in async context
user = await session.get(User, 1)
posts = user.posts  # ERROR! Lazy load not allowed in async

# CORRECT: Use eager loading at query time
stmt = select(User).where(User.id == 1).options(selectinload(User.posts))
result = await session.execute(stmt)
user = result.scalars().one()
posts = user.posts  # Fine — already loaded

# OR: Use AsyncSession.run_sync for one-off lazy loads (advanced)
# OR: Use lazy="raise" on relationship to get explicit errors
class User(Base):
    posts: Mapped[list["Post"]] = relationship(
        back_populates="author",
        lazy="raise"  # Will raise immediately if lazy-loaded — good for catching bugs
    )
```

---

## 12. FastAPI Integration Pattern

### The Dependency Injection Pattern

```python
# db/deps.py
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.engine import AsyncSessionLocal


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise


# In your router:
from fastapi import Depends, APIRouter
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.deps import get_db

router = APIRouter()

@router.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@router.post("/users")
async def create_user(data: UserCreate, db: AsyncSession = Depends(get_db)):
    user = User(username=data.username, email=data.email)
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return user
```

### The Repository Pattern (Recommended for Larger Apps)

Separating database logic from route handlers makes testing much easier.

```python
# repositories/user_repo.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

class UserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, user_id: int) -> User | None:
        return await self.session.get(User, user_id)

    async def get_by_username(self, username: str) -> User | None:
        result = await self.session.execute(
            select(User).where(User.username == username)
        )
        return result.scalar_one_or_none()

    async def list_active(self, limit: int = 100, offset: int = 0) -> list[User]:
        result = await self.session.execute(
            select(User)
            .where(User.is_active == True)
            .limit(limit)
            .offset(offset)
            .order_by(User.created_at.desc())
        )
        return result.scalars().all()

    async def create(self, username: str, email: str) -> User:
        user = User(username=username, email=email)
        self.session.add(user)
        await self.session.flush()   # Write to DB, don't commit yet
        return user


# Use in router with Depends
async def get_user_repo(db: AsyncSession = Depends(get_db)) -> UserRepository:
    return UserRepository(db)

@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    repo: UserRepository = Depends(get_user_repo)
):
    user = await repo.get_by_id(user_id)
    if not user:
        raise HTTPException(404, "Not found")
    return user
```

### Startup and Shutdown

```python
# main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.db.engine import async_engine
from app.db.base import Base

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: create tables (dev only — use Alembic in production)
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    # Shutdown: close connections
    await async_engine.dispose()

app = FastAPI(lifespan=lifespan)
```

---

## 13. The Unit of Work Pattern & Session Lifecycle

Understanding this deeply will save you from subtle bugs.

### What the Session Tracks

The Session maintains an **identity map**: a dictionary mapping `(ClassName, primary_key)` → Python object. This means:

```python
user1 = session.get(User, 1)
user2 = session.get(User, 1)
assert user1 is user2  # True! Same Python object
```

This is different from Django where each queryset always fetches a new Python object.

### Flush vs Commit

```python
with SessionLocal() as session:
    user = User(username="alice")
    session.add(user)

    # flush(): writes to DB within current transaction, but doesn't commit
    # Use this when you need the generated ID before committing
    session.flush()
    print(user.id)  # Now available because INSERT was executed

    post = Post(user_id=user.id, title="Hello")
    session.add(post)

    # commit(): makes everything permanent
    session.commit()
```

### When autoflush Matters

`autoflush=True` (default) means the session flushes pending changes before executing any SELECT. This prevents reading stale data:

```python
user = User(username="alice")
session.add(user)

# Because autoflush=True, this triggers a flush first:
count = session.scalar(select(func.count(User.id)))
# alice is now in the count, even before commit
```

### expire_on_commit and Detached Objects

After `session.commit()`, all loaded objects are **expired**. Accessing their attributes causes a lazy reload.

```python
user = session.get(User, 1)
session.commit()

# user.username is now "expired"
# Next access fires: SELECT * FROM users WHERE id = 1
print(user.username)  # triggers reload — fine if session is still open

# After session.close():
session.close()
print(user.username)  # RAISES DetachedInstanceError!
```

This is the most common error for FastAPI developers. If you return SQLAlchemy objects from a session context and access them later, you get `DetachedInstanceError`. Solutions:

1. Use `expire_on_commit=False` (async standard practice)
2. Use Pydantic schemas to serialize before returning
3. Call `session.refresh(obj)` before closing the session

---

## 14. Tricky But Valuable Topics

### The N+1 Problem in Depth

```python
# Bug: loads users, then fires one SELECT per user to get posts
users = session.scalars(select(User)).all()
for user in users:
    print(f"{user.username}: {len(user.posts)} posts")  # N queries!

# Fix with selectinload
users = session.scalars(
    select(User).options(selectinload(User.posts))
).all()
for user in users:
    print(f"{user.username}: {len(user.posts)} posts")  # 2 queries total
```

Enable SQL logging during development to catch this:
```python
engine = create_engine(DATABASE_URL, echo=True)
```

### Hybrid Properties

Like Django's `@property`, but also usable in SQL queries.

```python
from sqlalchemy.ext.hybrid import hybrid_property

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    first_name: Mapped[str] = mapped_column(String(50))
    last_name: Mapped[str] = mapped_column(String(50))

    @hybrid_property
    def full_name(self) -> str:
        return f"{self.first_name} {self.last_name}"

    @full_name.expression
    @classmethod
    def full_name(cls):
        return func.concat(cls.first_name, " ", cls.last_name)


# Use in Python
user.full_name  # "Alice Smith"

# Use in SQL query — this is what makes hybrid special
stmt = select(User).where(User.full_name == "Alice Smith")
# Generates: WHERE concat(first_name, ' ', last_name) = 'Alice Smith'
```

> **Django parallel:** Django properties work in Python only. Hybrid properties work in both Python and SQL — a significant advantage.

### `column_property` for Computed Columns

```python
from sqlalchemy.orm import column_property

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)

    post_count = column_property(
        select(func.count(Post.id))
        .where(Post.user_id == id)
        .correlate_except(Post)
        .scalar_subquery()
    )

# Now every User query includes a subquery for post count
users = session.scalars(select(User)).all()
users[0].post_count  # Available without extra query
```

### Soft Deletes

A common pattern — mark rows as deleted instead of removing them.

```python
from sqlalchemy import event

class SoftDeleteMixin:
    deleted_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), default=None)

    @property
    def is_deleted(self) -> bool:
        return self.deleted_at is not None

    def soft_delete(self):
        self.deleted_at = datetime.now(timezone.utc)


class Post(SoftDeleteMixin, Base):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))


# Always filter in queries — or use a custom query class / filtered relationship
stmt = select(Post).where(Post.deleted_at.is_(None))
```

### Custom Base with Timestamps

```python
from sqlalchemy.orm import DeclarativeBase, declared_attr
from sqlalchemy import DateTime, func
from datetime import datetime

class Base(DeclarativeBase):
    @declared_attr.directive
    def __tablename__(cls) -> str:
        return cls.__name__.lower() + "s"  # Auto-generate table names


class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now()
    )


class User(TimestampMixin, Base):
    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(50))
    # __tablename__ is auto-generated as "users"
```

### Transactions and Savepoints

```python
# Nested transaction with savepoint (rollback part of work)
with session.begin():
    user = User(username="alice")
    session.add(user)

    with session.begin_nested():  # SAVEPOINT
        post = Post(title="Hello", user_id=user.id)
        session.add(post)
        # If this raises, only the post is rolled back, not the user
        raise ValueError("oops")

    # user is still saved, post is not
```

### Inspecting What SQLAlchemy Will Execute

```python
# See the SQL without executing it
stmt = select(User).where(User.is_active == True)
print(stmt.compile(dialect=postgresql.dialect(), compile_kwargs={"literal_binds": True}))
# Output: SELECT users.id, users.username FROM users WHERE users.is_active = true
```

### Type Decorators — Custom Column Types

```python
from sqlalchemy import TypeDecorator, String
import json

class JSONEncodedList(TypeDecorator):
    """Stores a Python list as a JSON string in a VARCHAR column."""
    impl = String
    cache_ok = True

    def process_bind_param(self, value, dialect):
        if value is not None:
            return json.dumps(value)
        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            return json.loads(value)
        return value


class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    permissions: Mapped[list | None] = mapped_column(JSONEncodedList)

user.permissions = ["read", "write"]
session.commit()
# Stored as: '["read", "write"]' in DB
# Retrieved as: ["read", "write"] in Python
```

### `selectinload` vs `joinedload` — When to Use Which

| | `joinedload` | `selectinload` |
|---|---|---|
| Query count | 1 (with JOIN) | 2 (main + IN) |
| Best for | many-to-one, one-to-one | one-to-many, many-to-many |
| Risk | Cartesian product if loading many-to-many | None |
| Works async | Yes | Yes |

```python
# Loading 100 users with their 10 posts each:
# joinedload → 1 query returning 1000 rows (100×10 cartesian product)
# selectinload → 2 queries: SELECT users (100 rows) + SELECT posts WHERE user_id IN (...)
# selectinload is much more efficient here
```

### Using `with_expression` for Ad-hoc Annotations

```python
from sqlalchemy.orm import with_expression, query_expression

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(50))
    post_count: Mapped[int | None] = query_expression()  # placeholder


# Annotate dynamically
post_count_expr = (
    select(func.count(Post.id))
    .where(Post.user_id == User.id)
    .correlate(User)
    .scalar_subquery()
)

stmt = select(User).options(with_expression(User.post_count, post_count_expr))
users = session.scalars(stmt).all()
users[0].post_count  # Available!
```

---

## 15. Performance & Query Optimization

### Connection Pool Tuning

```python
engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,          # Permanent connections (tune to DB max_connections / workers)
    max_overflow=10,       # Burst connections
    pool_timeout=30,       # How long to wait for a connection from pool
    pool_recycle=1800,     # Recycle after 30 min (avoids "server closed connection")
    pool_pre_ping=True,    # Test connection before using (handles dropped connections)
)
```

### Bulk Operations

```python
# Slow: N individual INSERTs
for data in items:
    session.add(Model(**data))
await session.commit()

# Fast: Single bulk INSERT
await session.execute(insert(Model), list_of_dicts)
await session.commit()

# Faster for PostgreSQL: INSERT ... ON CONFLICT (upsert)
from sqlalchemy.dialects.postgresql import insert as pg_insert

stmt = pg_insert(User).values(list_of_dicts)
stmt = stmt.on_conflict_do_update(
    index_elements=["username"],
    set_={"email": stmt.excluded.email}
)
await session.execute(stmt)
```

### Only Load What You Need

```python
# Load only specific columns
stmt = select(User.id, User.username).where(User.is_active == True)
rows = session.execute(stmt).all()
# Returns Row objects, not User ORM objects

# Raiseload on columns you never need
from sqlalchemy.orm import raiseload, defer, load_only

stmt = select(User).options(
    load_only(User.id, User.username),    # Only load these columns
    defer(User.bio),                       # Load bio lazily when accessed
)

# Or exclude heavy columns
stmt = select(User).options(defer(User.large_binary_field, raiseload=True))
```

### Indexing in Models

```python
from sqlalchemy import Index

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), index=True)  # simple index
    username: Mapped[str] = mapped_column(String(50))

    # Composite index or custom index
    __table_args__ = (
        Index("ix_user_username_active", "username", "is_active"),
        Index("ix_user_email_lower", func.lower(email)),  # functional index
    )
```

---

## 16. Quick Reference Cheatsheet

### Setup

```python
# Sync
engine = create_engine("postgresql+psycopg2://...")
SessionLocal = sessionmaker(bind=engine)

# Async
engine = create_async_engine("postgresql+asyncpg://...")
AsyncSessionLocal = async_sessionmaker(bind=engine, expire_on_commit=False)
```

### Model Template

```python
class MyModel(Base):
    __tablename__ = "my_models"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    optional: Mapped[str | None] = mapped_column(String(100))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
```

### Query Patterns

```python
# Get by PK
obj = session.get(Model, pk)                           # sync
obj = await session.get(Model, pk)                     # async

# All
objs = session.scalars(select(Model)).all()
objs = (await session.execute(select(Model))).scalars().all()

# Filter
stmt = select(Model).where(Model.field == value)

# First or None
obj = session.scalars(stmt).first()

# One or exception
obj = session.scalars(stmt).one()

# Count
n = session.scalar(select(func.count()).select_from(Model))
```

### Common Gotchas Summary

| Gotcha | Cause | Fix |
|--------|-------|-----|
| `DetachedInstanceError` | Accessing expired attr outside session | `expire_on_commit=False` or Pydantic serialization |
| `MissingGreenlet` | Lazy load in async context | Use `selectinload`/`joinedload` eagerly |
| N+1 queries | Default lazy loading on relationships | Use `selectinload`/`joinedload` |
| Server defaults not populated | DB sets them, not Python | `await session.refresh(obj)` |
| Relationship not refreshed | Stale in-memory object | `session.expire(obj)` or `session.refresh(obj)` |
| Migration misses rename | Alembic sees drop+add | Edit migration manually |

---

*SQLAlchemy rewards understanding its internals. The more you understand the identity map, the Unit of Work, and the difference between flush and commit, the more confidently you'll write correct, performant database code.*
