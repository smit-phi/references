# FastAPI — Comprehensive Guide
> *A complete reference for Django/DRF developers transitioning to FastAPI*

---

## Table of Contents

1. [FastAPI vs Django — The Mental Model Shift](#1-fastapi-vs-django--the-mental-model-shift)
2. [Django → FastAPI Equivalents Cheatsheet](#2-django--fastapi-equivalents-cheatsheet)
3. [Installation & Project Setup](#3-installation--project-setup)
4. [Core Concepts — The FastAPI Way](#4-core-concepts--the-fastapi-way)
5. [Path Operations (Routes)](#5-path-operations-routes)
6. [Request Handling](#6-request-handling)
7. [Pydantic & Data Validation](#7-pydantic--data-validation)
8. [Dependency Injection](#8-dependency-injection)
9. [Database Integration (SQLAlchemy + Alembic)](#9-database-integration-sqlalchemy--alembic)
10. [Authentication & Security](#10-authentication--security)
11. [Middleware](#11-middleware)
12. [Background Tasks](#12-background-tasks)
13. [WebSockets](#13-websockets)
14. [File Uploads & Static Files](#14-file-uploads--static-files)
15. [Testing](#15-testing)
16. [Async Programming in FastAPI](#16-async-programming-in-fastapi)
17. [Application Structure & Best Practices](#17-application-structure--best-practices)
18. [Deployment](#18-deployment)

---

## 1. FastAPI vs Django — The Mental Model Shift

### Philosophy

| Dimension | Django | FastAPI |
|---|---|---|
| **Design Philosophy** | "Batteries included" — everything built-in | "Do one thing well" — minimal core, composable |
| **Primary Paradigm** | Synchronous-first (async added later) | Async-first (sync supported) |
| **Data Layer** | Tightly coupled ORM (Django ORM) | ORM-agnostic (you bring SQLAlchemy, Tortoise, etc.) |
| **Validation** | Django Forms / DRF Serializers | Pydantic (integrated at the type-hint level) |
| **API Style** | DRF ViewSets, Serializers, Routers | Path operation functions + Pydantic schemas |
| **Admin Panel** | Built-in (`/admin`) | None by default (use SQLAdmin, etc.) |
| **Template Engine** | Jinja2 (built-in) | None by default (use Jinja2 separately) |
| **Auto-generated Docs** | drf-yasg / drf-spectacular (add-on) | Built-in (Swagger UI + ReDoc) |
| **Performance** | Good | Excellent (one of the fastest Python frameworks) |
| **Learning Curve** | Steep (many conventions to learn) | Moderate (leverages Python type hints you already know) |

---

### Where They Sit in the Backend Ecosystem

```
Django Stack                       FastAPI Stack
─────────────────────────          ────────────────────────────
Browser / Client                   Browser / Client
      │                                   │
Django (WSGI/ASGI)                 Uvicorn/Gunicorn (ASGI server)
      │                                   │
Django ORM                         FastAPI Application
      │                                   │
Django Models                      SQLAlchemy / Tortoise ORM
      │                                   │
PostgreSQL / MySQL                 PostgreSQL / MySQL / MongoDB
```

Django is a **full-stack web framework** — it handles URL routing, views, templates, ORM, admin, sessions, auth, and more in one package.

FastAPI is a **modern API framework** — it handles routing, request/response parsing, validation, and async I/O at exceptional speed. Everything else (ORM, auth strategy, background jobs) is wired in separately. This means more decisions, but also much less magic.

---

### Key Conceptual Differences

**1. Type Hints Are First-Class Citizens**

In Django/DRF, you define serializers as classes with fields. In FastAPI, you write Python type hints and Pydantic handles the rest automatically — validation, serialization, and OpenAPI schema generation all happen from the same type annotation.

**2. No ORM — You Choose Your Own**

Django ships with its own ORM. FastAPI has no opinion on data access. SQLAlchemy (with async support via `sqlalchemy[asyncio]`) is the most common choice, but Tortoise ORM, Beanie (MongoDB), and raw SQL (via `databases` library) are all valid.

**3. Async Is Native**

FastAPI is built on Starlette (ASGI framework) and supports `async/await` natively in every route handler. This means you can handle thousands of concurrent requests without spawning threads — critical for high-performance APIs.

**4. Dependency Injection Is Explicit**

Django uses class-based mixins, middleware, and `request.user` accessed globally. FastAPI uses a formal DI system — you declare dependencies as function parameters and FastAPI resolves them. It's more verbose but far more testable and composable.

**5. OpenAPI Docs Are Automatic**

FastAPI generates `/docs` (Swagger UI) and `/redoc` (ReDoc) out of the box — no extra packages, no configuration. Every route, parameter, and schema is documented automatically from your code.

---

## 2. Django → FastAPI Equivalents Cheatsheet

| Django / DRF Concept | FastAPI Equivalent | Notes |
|---|---|---|
| `urls.py` | `APIRouter` + `app.include_router()` | Modular routing |
| `views.py` (FBV) | Path operation function (`@app.get(...)`) | Direct mapping |
| `views.py` (CBV) | Class-based routes (via `APIRouter` + methods) | Less common in FastAPI |
| `ViewSet` (DRF) | `APIRouter` with grouped path operations | No automatic CRUD generation |
| `Serializer` (DRF) | `Pydantic BaseModel` | Validation + serialization in one |
| `ModelSerializer` | Pydantic model mirroring SQLAlchemy model | Must be written manually or via `sqlmodel` |
| `request.data` | Function parameter with `Body(...)` or Pydantic model | Type-safe |
| `request.query_params` | Function parameter declared as non-path variable | Auto-parsed from URL |
| `request.user` | `Depends(get_current_user)` | Explicit DI |
| `permissions.py` | `Depends(require_role)` or custom dependency | Composable |
| `authentication.py` | `OAuth2PasswordBearer` + JWT utils | Manual implementation |
| `middleware.py` | Starlette `Middleware` or `@app.middleware("http")` | Similar concept |
| `signals.py` | Background Tasks or event hooks | `BackgroundTasks` class |
| `settings.py` | Pydantic `BaseSettings` (from `pydantic-settings`) | Env-based config |
| `manage.py migrate` | Alembic (`alembic upgrade head`) | External migration tool |
| `manage.py shell` | Regular Python REPL or `ipython` | No special shell |
| `admin.py` | SQLAdmin / admin-fastapi (third party) | Not built-in |
| `tests.py` | `pytest` + `TestClient` (from Starlette) | Similar pattern |
| `Response` object | `JSONResponse`, `Response`, or return dict | Automatic serialization |
| `status_code` | `status_code=` param in decorator | Explicit |
| `@action` (DRF ViewSet) | Additional route on `APIRouter` | Manual |
| `throttling.py` | `slowapi` library | Third-party |
| `pagination.py` | Custom dependency or `fastapi-pagination` lib | Third-party |

---

## 3. Installation & Project Setup

### Install FastAPI

```bash
pip install fastapi "uvicorn[standard]" pydantic
```

`uvicorn` is the ASGI server (like Gunicorn for WSGI). The `[standard]` extra includes `watchfiles` for auto-reload and `websockets` support.

### Minimal Application

```python
# main.py
from fastapi import FastAPI

app = FastAPI(
    title="My API",
    description="A sample FastAPI application",
    version="1.0.0",
)

@app.get("/")
def root():
    return {"message": "Hello World"}
```

### Run the Server

```bash
# Development (with auto-reload)
uvicorn main:app --reload

# Production
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

Visit:
- `http://127.0.0.1:8000/docs` → Swagger UI
- `http://127.0.0.1:8000/redoc` → ReDoc
- `http://127.0.0.1:8000/openapi.json` → Raw OpenAPI schema

---

## 4. Core Concepts — The FastAPI Way

FastAPI is built on top of **Starlette** (routing, middleware, WebSockets, static files) and **Pydantic** (data validation). Everything flows through these two foundations.

```
FastAPI
├── Starlette (ASGI framework)
│   ├── Routing
│   ├── Middleware
│   ├── WebSockets
│   ├── Static Files
│   └── Background Tasks
└── Pydantic (data layer)
    ├── Schema validation
    ├── Serialization
    └── Settings management
```

### The Request Lifecycle

```
Client Request
      │
Uvicorn (ASGI server receives raw HTTP)
      │
Starlette Middleware Stack (CORS, Auth, Logging, etc.)
      │
FastAPI Router (matches path + method)
      │
Dependency Injection Resolution
      │
Path Operation Function (your code)
      │
Pydantic Response Validation
      │
HTTP Response → Client
```

---

## 5. Path Operations (Routes)

Path operations are the FastAPI equivalent of Django URL patterns + view functions combined.

### HTTP Methods

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items")
def list_items():
    return [{"id": 1}]

@app.post("/items")
def create_item():
    return {"id": 2}

@app.put("/items/{item_id}")
def update_item(item_id: int):
    return {"id": item_id}

@app.patch("/items/{item_id}")
def partial_update(item_id: int):
    return {"id": item_id}

@app.delete("/items/{item_id}")
def delete_item(item_id: int):
    return {"deleted": True}
```

### Path Parameters

```python
@app.get("/users/{user_id}/posts/{post_id}")
def get_post(user_id: int, post_id: int):
    # FastAPI automatically validates and converts types
    return {"user_id": user_id, "post_id": post_id}
```

### Query Parameters

```python
from typing import Optional

@app.get("/items")
def list_items(
    skip: int = 0,
    limit: int = 10,
    search: Optional[str] = None,
    is_active: bool = True,
):
    return {"skip": skip, "limit": limit, "search": search}
```

Any function parameter **not in the path** is automatically treated as a query parameter.

### Response Models & Status Codes

```python
from fastapi import status
from pydantic import BaseModel

class ItemOut(BaseModel):
    id: int
    name: str

@app.post(
    "/items",
    response_model=ItemOut,
    status_code=status.HTTP_201_CREATED,
    summary="Create a new item",
    tags=["items"],
)
def create_item():
    return {"id": 1, "name": "Widget", "secret_field": "hidden"}
    # "secret_field" is stripped because it's not in ItemOut
```

### APIRouter — Equivalent of Django's `include()`

```python
# routers/items.py
from fastapi import APIRouter

router = APIRouter(prefix="/items", tags=["items"])

@router.get("/")
def list_items():
    return []

@router.get("/{item_id}")
def get_item(item_id: int):
    return {"id": item_id}

# main.py
from fastapi import FastAPI
from routers import items

app = FastAPI()
app.include_router(items.router)
```

---

## 6. Request Handling

### Request Body

```python
from pydantic import BaseModel
from fastapi import Body

class ItemCreate(BaseModel):
    name: str
    price: float
    is_available: bool = True

@app.post("/items")
def create_item(item: ItemCreate):
    # FastAPI reads the JSON body and validates against ItemCreate
    print(item.name, item.price)
    return item
```

### Headers & Cookies

```python
from fastapi import Header, Cookie
from typing import Optional

@app.get("/protected")
def protected(
    authorization: Optional[str] = Header(None),
    session_id: Optional[str] = Cookie(None),
):
    return {"auth": authorization, "session": session_id}
```

### Form Data

```python
from fastapi import Form

@app.post("/login")
def login(username: str = Form(...), password: str = Form(...)):
    return {"username": username}
```

### Custom Request Object

```python
from fastapi import Request

@app.get("/debug")
async def debug(request: Request):
    print(request.headers)
    print(request.client.host)
    body = await request.body()
    return {"method": request.method}
```

### Custom Response

```python
from fastapi.responses import JSONResponse, HTMLResponse, RedirectResponse

@app.get("/redirect")
def redirect():
    return RedirectResponse(url="/items")

@app.get("/custom")
def custom():
    return JSONResponse(
        content={"error": "Something went wrong"},
        status_code=400,
        headers={"X-Custom-Header": "value"},
    )
```

---

## 7. Pydantic & Data Validation

Pydantic is the heart of FastAPI's data layer — analogous to DRF Serializers but driven entirely by Python type annotations.

### Basic Model

```python
from pydantic import BaseModel, Field, EmailStr
from typing import Optional, List
from datetime import datetime

class UserCreate(BaseModel):
    name: str = Field(..., min_length=2, max_length=100)
    email: EmailStr
    age: int = Field(..., ge=18, le=120)  # gte=18, lte=120
    bio: Optional[str] = None

class UserOut(BaseModel):
    id: int
    name: str
    email: str
    created_at: datetime

    class Config:
        from_attributes = True  # Pydantic v2 equivalent of orm_mode
```

### Nested Models

```python
class Address(BaseModel):
    street: str
    city: str
    country: str

class UserWithAddress(BaseModel):
    name: str
    address: Address
    tags: List[str] = []
```

### Validators

```python
from pydantic import field_validator, model_validator

class SignupForm(BaseModel):
    username: str
    password: str
    confirm_password: str

    @field_validator("username")
    @classmethod
    def username_alphanumeric(cls, v):
        if not v.isalnum():
            raise ValueError("Username must be alphanumeric")
        return v.lower()

    @model_validator(mode="after")
    def passwords_match(self):
        if self.password != self.confirm_password:
            raise ValueError("Passwords do not match")
        return self
```

### Pydantic Settings (replaces `settings.py`)

```bash
pip install pydantic-settings
```

```python
# config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    secret_key: str
    debug: bool = False
    allowed_hosts: list[str] = ["*"]

    class Config:
        env_file = ".env"

settings = Settings()
```

---

## 8. Dependency Injection

FastAPI's DI system is the most powerful feature separating it from simple frameworks. It's the equivalent of DRF's `get_permissions()`, `get_authenticators()`, and `get_queryset()` — but unified and composable.

### Basic Dependency

```python
from fastapi import Depends

def get_query_params(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}

@app.get("/items")
def list_items(pagination: dict = Depends(get_query_params)):
    return pagination
```

### Database Session Dependency (most common pattern)

```python
from sqlalchemy.orm import Session
from database import SessionLocal

def get_db():
    db = SessionLocal()
    try:
        yield db        # FastAPI runs code before yield, then resumes after
    finally:
        db.close()      # Always closes the session

@app.get("/users")
def list_users(db: Session = Depends(get_db)):
    return db.query(User).all()
```

This `yield`-based pattern is FastAPI's equivalent of Django's per-request database connection management.

### Auth Dependency

```python
from fastapi import HTTPException, status
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")

def get_current_user(token: str = Depends(oauth2_scheme)):
    user = verify_token(token)  # Your JWT decoding logic
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",
        )
    return user

def require_admin(current_user = Depends(get_current_user)):
    if current_user.role != "admin":
        raise HTTPException(status_code=403, detail="Not an admin")
    return current_user

@app.delete("/users/{user_id}")
def delete_user(user_id: int, admin = Depends(require_admin)):
    return {"deleted": user_id}
```

### Class-Based Dependencies

```python
class Pagination:
    def __init__(self, skip: int = 0, limit: int = 10):
        self.skip = skip
        self.limit = limit

@app.get("/posts")
def list_posts(pagination: Pagination = Depends()):
    return {"skip": pagination.skip, "limit": pagination.limit}
```

---

## 9. Database Integration (SQLAlchemy + Alembic)

FastAPI has no built-in ORM. SQLAlchemy is the most widely-used choice.

### Setup

```bash
pip install sqlalchemy alembic psycopg2-binary
```

### Database Models

```python
# models.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime, ForeignKey
from sqlalchemy.orm import relationship, declarative_base
from datetime import datetime

Base = declarative_base()

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True)
    name = Column(String)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)

    posts = relationship("Post", back_populates="author")

class Post(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True)
    title = Column(String)
    body = Column(String)
    author_id = Column(Integer, ForeignKey("users.id"))

    author = relationship("User", back_populates="posts")
```

### Database Connection

```python
# database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from config import settings

engine = create_engine(settings.database_url)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

### CRUD Operations

```python
# crud/users.py
from sqlalchemy.orm import Session
from models import User
from schemas import UserCreate

def get_user(db: Session, user_id: int):
    return db.query(User).filter(User.id == user_id).first()

def get_user_by_email(db: Session, email: str):
    return db.query(User).filter(User.email == email).first()

def create_user(db: Session, user: UserCreate):
    db_user = User(email=user.email, name=user.name)
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

def delete_user(db: Session, user_id: int):
    db.query(User).filter(User.id == user_id).delete()
    db.commit()
```

### Alembic Migrations (equivalent of Django migrations)

```bash
# Initialize Alembic
alembic init alembic

# Create a migration
alembic revision --autogenerate -m "create users table"

# Apply migrations
alembic upgrade head

# Rollback one step
alembic downgrade -1
```

In `alembic/env.py`, point to your `Base.metadata`:

```python
from models import Base
target_metadata = Base.metadata
```

---

## 10. Authentication & Security

FastAPI provides security utilities but you implement the actual logic — no `django.contrib.auth` equivalent.

### JWT Authentication (most common pattern)

```bash
pip install python-jose[cryptography] passlib[bcrypt]
```

```python
# auth/utils.py
from datetime import datetime, timedelta
from jose import jwt, JWTError
from passlib.context import CryptContext

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"])

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(data: dict) -> str:
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    return jwt.encode({**data, "exp": expire}, SECRET_KEY, algorithm=ALGORITHM)

def decode_token(token: str) -> dict:
    return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
```

```python
# auth/router.py
from fastapi import APIRouter, Depends, HTTPException
from fastapi.security import OAuth2PasswordRequestForm

router = APIRouter(prefix="/auth", tags=["auth"])

@router.post("/token")
def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db),
):
    user = authenticate_user(db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(status_code=400, detail="Invalid credentials")
    
    token = create_access_token({"sub": str(user.id)})
    return {"access_token": token, "token_type": "bearer"}
```

### OAuth2 with Scopes

```python
from fastapi.security import SecurityScopes

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="/auth/token",
    scopes={"read": "Read access", "write": "Write access", "admin": "Admin access"},
)

def get_current_user(
    security_scopes: SecurityScopes,
    token: str = Depends(oauth2_scheme),
):
    # Validate token and check scopes
    ...
```

### HTTPS / CORS

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myfrontend.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## 11. Middleware

Middleware in FastAPI works the same conceptually as Django middleware — wrapping the request/response cycle.

### Custom Middleware

```python
from fastapi import Request
import time

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    response.headers["X-Process-Time"] = str(time.time() - start)
    return response
```

### Starlette-Style Middleware (for more complex cases)

```python
from starlette.middleware.base import BaseHTTPMiddleware

class LoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        print(f"→ {request.method} {request.url}")
        response = await call_next(request)
        print(f"← {response.status_code}")
        return response

app.add_middleware(LoggingMiddleware)
```

### Built-in Middleware Options

```python
from starlette.middleware.sessions import SessionMiddleware
from starlette.middleware.gzip import GZipMiddleware
from starlette.middleware.trustedhost import TrustedHostMiddleware

app.add_middleware(SessionMiddleware, secret_key="secret")
app.add_middleware(GZipMiddleware, minimum_size=1000)
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["example.com"])
```

---

## 12. Background Tasks

FastAPI's `BackgroundTasks` is the lightweight equivalent of Django signals or Celery for simple post-response work.

### Using BackgroundTasks

```python
from fastapi import BackgroundTasks
import smtplib

def send_welcome_email(email: str):
    # This runs AFTER the response is sent to the client
    print(f"Sending welcome email to {email}")

@app.post("/users")
def create_user(user: UserCreate, background_tasks: BackgroundTasks):
    new_user = save_user(user)
    background_tasks.add_task(send_welcome_email, user.email)
    return new_user  # Client gets this immediately, email sends after
```

### For Heavy Work — Use Celery

For tasks that are long-running, distributed, or need retry logic, integrate Celery just as you would with Django:

```python
# celery_app.py
from celery import Celery

celery = Celery("tasks", broker="redis://localhost:6379/0")

@celery.task
def process_video(video_id: int):
    ...

# In your route
@app.post("/videos/{video_id}/process")
def queue_processing(video_id: int):
    process_video.delay(video_id)
    return {"status": "queued"}
```

---

## 13. WebSockets

FastAPI supports WebSockets natively via Starlette.

### Basic WebSocket

```python
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: int):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo from {client_id}: {data}")
    except WebSocketDisconnect:
        print(f"Client {client_id} disconnected")
```

### Connection Manager (for broadcast/multi-client)

```python
class ConnectionManager:
    def __init__(self):
        self.active: list[WebSocket] = []

    async def connect(self, ws: WebSocket):
        await ws.accept()
        self.active.append(ws)

    def disconnect(self, ws: WebSocket):
        self.active.remove(ws)

    async def broadcast(self, message: str):
        for ws in self.active:
            await ws.send_text(message)

manager = ConnectionManager()

@app.websocket("/chat")
async def chat(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            msg = await websocket.receive_text()
            await manager.broadcast(msg)
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

---

## 14. File Uploads & Static Files

### File Upload

```python
from fastapi import UploadFile, File
import shutil

@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    with open(f"uploads/{file.filename}", "wb") as buffer:
        shutil.copyfileobj(file.file, buffer)
    
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": file.size,
    }

# Multiple files
@app.post("/upload-many")
async def upload_many(files: list[UploadFile] = File(...)):
    return [f.filename for f in files]
```

### Static Files

```python
from fastapi.staticfiles import StaticFiles

app.mount("/static", StaticFiles(directory="static"), name="static")
# Files in ./static/ are served at /static/filename.ext
```

---

## 15. Testing

FastAPI uses `TestClient` from Starlette, which wraps `httpx` — very similar to Django's test client.

```bash
pip install pytest httpx
```

### Basic Tests

```python
# test_main.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_root():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}

def test_create_item():
    response = client.post("/items", json={"name": "Widget", "price": 9.99})
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Widget"
```

### Overriding Dependencies in Tests

```python
from fastapi.testclient import TestClient
from main import app
from dependencies import get_db

def override_get_db():
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()

app.dependency_overrides[get_db] = override_get_db
client = TestClient(app)
```

### Async Tests

```bash
pip install pytest-asyncio
```

```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_async_endpoint():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/")
    assert response.status_code == 200
```

---

## 16. Async Programming in FastAPI

This is one of the biggest differences from Django. Understanding when to use `async def` vs `def` is critical.

### Rules

```python
# Use async def when your function calls async libraries
@app.get("/async-example")
async def async_handler():
    result = await some_async_db_call()   # await required
    return result

# Use regular def for CPU-bound or sync work
# FastAPI runs sync functions in a thread pool automatically
@app.get("/sync-example")
def sync_handler():
    result = regular_db_call()            # No await
    return result
```

### Async Database with SQLAlchemy

```bash
pip install sqlalchemy[asyncio] asyncpg
```

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/db"
engine = create_async_engine(DATABASE_URL)

async def get_db():
    async with AsyncSession(engine) as session:
        yield session

@app.get("/users")
async def list_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    return result.scalars().all()
```

### Lifespan Events (replaces `AppConfig.ready()`)

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: runs before the app starts accepting requests
    await database.connect()
    print("DB connected")
    
    yield  # App is running here
    
    # Shutdown: runs when app is stopping
    await database.disconnect()
    print("DB disconnected")

app = FastAPI(lifespan=lifespan)
```

---

## 17. Application Structure & Best Practices

### Recommended Project Structure

```
my_project/
├── app/
│   ├── __init__.py
│   ├── main.py               # FastAPI app factory
│   ├── config.py             # Pydantic settings
│   ├── database.py           # DB engine & session
│   ├── dependencies.py       # Shared dependencies (get_db, get_current_user)
│   │
│   ├── models/               # SQLAlchemy ORM models
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── post.py
│   │
│   ├── schemas/              # Pydantic schemas (request/response)
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── post.py
│   │
│   ├── crud/                 # Database operations
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── post.py
│   │
│   ├── routers/              # APIRouter modules
│   │   ├── __init__.py
│   │   ├── users.py
│   │   ├── posts.py
│   │   └── auth.py
│   │
│   └── services/             # Business logic
│       ├── email.py
│       └── storage.py
│
├── alembic/                  # Database migrations
│   ├── versions/
│   └── env.py
├── tests/
│   ├── conftest.py
│   ├── test_users.py
│   └── test_posts.py
├── alembic.ini
├── requirements.txt
└── .env
```

### App Factory Pattern

```python
# app/main.py
from fastapi import FastAPI
from routers import users, posts, auth
from middleware import setup_middleware
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    yield
    # shutdown

def create_app() -> FastAPI:
    app = FastAPI(title="My API", lifespan=lifespan)
    setup_middleware(app)
    app.include_router(auth.router)
    app.include_router(users.router)
    app.include_router(posts.router)
    return app

app = create_app()
```

### Error Handling

```python
from fastapi import HTTPException
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

# Custom exception
class ItemNotFound(Exception):
    def __init__(self, item_id: int):
        self.item_id = item_id

# Register handler
@app.exception_handler(ItemNotFound)
def item_not_found_handler(request, exc: ItemNotFound):
    return JSONResponse(
        status_code=404,
        content={"detail": f"Item {exc.item_id} not found"},
    )

# Override default validation error response
@app.exception_handler(RequestValidationError)
def validation_error_handler(request, exc: RequestValidationError):
    return JSONResponse(
        status_code=422,
        content={"errors": exc.errors()},
    )
```

---

## 18. Deployment

### Docker

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Gunicorn + Uvicorn Workers (Production)

```bash
pip install gunicorn

# Multiple Uvicorn workers managed by Gunicorn
gunicorn app.main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000
```

### Environment Variables

```bash
# .env
DATABASE_URL=postgresql://user:pass@localhost/mydb
SECRET_KEY=supersecretkey
DEBUG=false
```

### Health Check Endpoint

```python
@app.get("/health", tags=["health"])
def health_check():
    return {"status": "ok"}
```

---

## Quick Reference Summary

| Task | Code Pattern |
|---|---|
| Create a GET route | `@app.get("/path")` |
| Path parameter | `def fn(item_id: int):` |
| Query parameter | `def fn(skip: int = 0):` |
| Request body | `def fn(body: MyModel):` |
| Database session | `db: Session = Depends(get_db)` |
| Auth guard | `user = Depends(get_current_user)` |
| Return 404 | `raise HTTPException(404, "Not found")` |
| Background task | `bg.add_task(fn, arg)` |
| Return specific status | `@app.post("/", status_code=201)` |
| Filter response fields | `response_model=MyOutputSchema` |

---

*This guide covers FastAPI up to v0.111+. Always check the [official FastAPI docs](https://fastapi.tiangolo.com) for the latest updates.*
