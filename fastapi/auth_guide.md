# Building a Secure Auth API from First Principles
### JWT Authentication · Role-Based Access Control · FastAPI + SQLAlchemy

---

## What You'll Build

A production-style User Profile API with:
- User registration and login
- JWT-based authentication
- Role-based access control (admin vs. regular user)
- Secure password hashing
- SQLite database via SQLAlchemy (async)

By the end you'll understand **why** each piece exists, not just how to copy it.

---

## Mental Model First: How Auth Actually Works

Before writing a single line, understand the flow at an industry level.

### The Problem Auth Solves

HTTP is **stateless** — every request is isolated. The server has no memory of you. So after you log in, how does the server know the *next* request is still you?

**Two classic solutions:**

| Approach | How it works | Used by |
|---|---|---|
| **Sessions** | Server stores your session in DB/Redis, gives you a cookie ID | Traditional web apps |
| **JWT** | Server gives you a self-contained signed token; no server-side storage needed | APIs, mobile apps, microservices |

You're building a JWT API. Here's the full flow visually:

```
REGISTRATION
  Client ──POST /register──► Server
                              ├─ Validate input
                              ├─ Hash password (bcrypt)
                              └─ Save user to DB

LOGIN
  Client ──POST /login──────► Server
                              ├─ Find user by email
                              ├─ Verify password hash
                              └─ Return signed JWT token

AUTHENTICATED REQUEST
  Client ──GET /profile──────► Server
   (sends JWT in header)       ├─ Decode & verify JWT signature
                               ├─ Extract user_id from token
                               ├─ Load user from DB
                               └─ Return protected data
```

### What is a JWT?

A JWT is three Base64-encoded parts joined by dots:

```
eyJhbGciOiJIUzI1NiJ9  .  eyJ1c2VyX2lkIjoxfQ  .  SflKxwRJSMeKKF2QT4
      HEADER                    PAYLOAD              SIGNATURE
```

- **Header**: algorithm used (`HS256`)
- **Payload**: claims — data you put in (e.g. `user_id`, `role`, `exp`)
- **Signature**: `HMAC(header + payload, SECRET_KEY)` — proves it wasn't tampered with

> **Key insight**: The server never stores the token. It just verifies the signature on every request using the same `SECRET_KEY`. If the payload was tampered with, the signature won't match → rejected.

### What is Password Hashing?

You **never** store passwords as plain text. You store a one-way hash.

```
"mypassword123"  ──bcrypt──►  "$2b$12$abc...xyz"   ← stored in DB

Login attempt:
bcrypt.verify("mypassword123", "$2b$12$abc...xyz")  ──► True ✓
bcrypt.verify("wrongpassword", "$2b$12$abc...xyz")  ──► False ✗
```

bcrypt is slow *by design* — it makes brute-force attacks expensive.

### What is RBAC?

Role-Based Access Control ties permissions to roles, not individual users.

```
User has role: "admin"  ──► can access /admin/users
User has role: "user"   ──► GET /profile (own data only)
No token at all         ──► 401 Unauthorized
Wrong role              ──► 403 Forbidden
```

In FastAPI, this is implemented as a **dependency** — a function that runs before your route handler and either passes or raises an exception.

---

## Project Structure

```
auth_api/
├── main.py          ← FastAPI app, routes
├── database.py      ← SQLAlchemy engine + session
├── models.py        ← ORM User model
├── schemas.py       ← Pydantic request/response models
├── auth.py          ← JWT creation/decoding, password hashing
├── dependencies.py  ← Reusable FastAPI dependencies (get_current_user, require_admin)
└── requirements.txt
```

---

## Step 1: Install Dependencies

```bash
pip install fastapi uvicorn sqlalchemy aiosqlite passlib[bcrypt] python-jose[cryptography] python-dotenv
```

**What each package does:**

| Package | Purpose |
|---|---|
| `passlib[bcrypt]` | Password hashing with bcrypt |
| `python-jose` | JWT encode/decode |
| `sqlalchemy + aiosqlite` | Async ORM + SQLite driver |
| `python-dotenv` | Load `.env` for secrets |

---

## Step 2: Database Setup (`database.py`)

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase
from typing import AsyncGenerator

DATABASE_URL = "sqlite+aiosqlite:///./auth.db"

engine = create_async_engine(DATABASE_URL, echo=True)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with SessionLocal() as session:
        yield session
```

Nothing new here from your todo app — this is the same pattern.

---

## Step 3: User Model (`models.py`)

```python
from sqlalchemy import String, Boolean
from sqlalchemy.orm import Mapped, mapped_column
from database import Base

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False, index=True)
    username: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    is_admin: Mapped[bool] = mapped_column(Boolean, default=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
```

**Notice:**
- We store `hashed_password`, never the raw password
- `is_admin` is a simple boolean flag — in larger systems this becomes a `roles` relationship
- `index=True` on email because every login does a lookup by email

---

## Step 4: Schemas (`schemas.py`)

Pydantic schemas are your **contract** with the outside world. Separate from ORM models intentionally — you control exactly what goes in and out.

```python
from pydantic import BaseModel, EmailStr, field_validator
from typing import Optional

# ── Registration ──────────────────────────────────────────────
class UserRegister(BaseModel):
    email: EmailStr
    username: str
    password: str

    @field_validator("password")
    @classmethod
    def password_strength(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError("Password must be at least 8 characters")
        return v

    @field_validator("username")
    @classmethod
    def username_clean(cls, v: str) -> str:
        if len(v) < 3:
            raise ValueError("Username must be at least 3 characters")
        return v.strip().lower()


# ── Login ──────────────────────────────────────────────────────
class UserLogin(BaseModel):
    email: EmailStr
    password: str


# ── Token Response ─────────────────────────────────────────────
class Token(BaseModel):
    access_token: str
    token_type: str = "bearer"


# ── Profile Responses ──────────────────────────────────────────
class UserProfileResponse(BaseModel):
    id: int
    email: str
    username: str
    is_active: bool
    # Note: is_admin is NOT here — regular users can't see this field

    model_config = {"from_attributes": True}


class AdminUserResponse(UserProfileResponse):
    is_admin: bool  # Only admins can see this field


# ── Profile Update ─────────────────────────────────────────────
class UserProfileUpdate(BaseModel):
    username: Optional[str] = None
    email: Optional[EmailStr] = None
```

**Key design decision**: `UserProfileResponse` deliberately omits `is_admin`. The `AdminUserResponse` extends it with the sensitive field. This is RBAC at the schema level — admins and users get different shapes of data.

---

## Step 5: Auth Utilities (`auth.py`)

This is the heart of the system. Read it carefully.

```python
from datetime import datetime, timedelta, timezone
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
import os

# ── Config ─────────────────────────────────────────────────────
SECRET_KEY = os.getenv("SECRET_KEY", "CHANGE-THIS-IN-PRODUCTION-USE-DOTENV")
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# ── Password Hashing ───────────────────────────────────────────
# CryptContext handles the bcrypt work for us
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(plain_password: str) -> str:
    """Turn 'mypassword123' into '$2b$12$...' """
    return pwd_context.hash(plain_password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Returns True if plain matches the stored hash"""
    return pwd_context.verify(plain_password, hashed_password)


# ── JWT ────────────────────────────────────────────────────────
def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """
    Creates a signed JWT token.

    'data' is your payload — typically {"sub": str(user_id), "role": "admin"}
    'sub' (subject) is a JWT standard claim meaning "who this token is about"
    """
    to_encode = data.copy()

    # Always set an expiry — tokens without expiry are a security risk
    expire = datetime.now(timezone.utc) + (
        expires_delta or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    to_encode["exp"] = expire

    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def decode_access_token(token: str) -> Optional[dict]:
    """
    Verifies the signature and returns the payload.
    Returns None if the token is invalid or expired.
    """
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except JWTError:
        return None
```

**Why `sub` (subject)?**
It's a JWT standard claim. Convention is to put the user's ID here as a string. You'll see `sub` everywhere in OAuth2/OIDC systems.

---

## Step 6: Dependencies (`dependencies.py`)

Dependencies are FastAPI's superpower. They're just functions that run before your route.

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from database import get_db
from models import User
from auth import decode_access_token

# This tells FastAPI to look for: Authorization: Bearer <token>
bearer_scheme = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    """
    Dependency that:
    1. Extracts the Bearer token from the Authorization header
    2. Decodes and verifies the JWT
    3. Loads the user from DB
    4. Returns the User object — or raises 401

    Any route that Depends() on this is automatically protected.
    """
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    token = credentials.credentials
    payload = decode_access_token(token)

    if payload is None:
        raise credentials_exception

    user_id: str = payload.get("sub")
    if user_id is None:
        raise credentials_exception

    user = await db.get(User, int(user_id))
    if user is None or not user.is_active:
        raise credentials_exception

    return user


async def require_admin(current_user: User = Depends(get_current_user)) -> User:
    """
    Dependency that layers on top of get_current_user.
    First validates the token (via get_current_user),
    then checks the admin role.

    This is dependency chaining — FastAPI resolves the tree automatically.
    """
    if not current_user.is_admin:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Admin privileges required",
        )
    return current_user
```

**The dependency chain for an admin route:**

```
Request arrives
    │
    ▼
bearer_scheme  ──── extracts token from header
    │
    ▼
get_current_user ── decodes JWT, loads User from DB
    │
    ▼
require_admin ───── checks user.is_admin == True
    │
    ▼
Route handler ───── receives User object, does its work
```

If any step fails, FastAPI short-circuits and returns the exception immediately.

---

## Step 7: Main Application (`main.py`)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, HTTPException, status, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from database import engine, Base, get_db
from models import User
from schemas import (
    UserRegister, UserLogin, Token,
    UserProfileResponse, AdminUserResponse, UserProfileUpdate,
)
from auth import hash_password, verify_password, create_access_token
from dependencies import get_current_user, require_admin


@asynccontextmanager
async def lifespan(app: FastAPI):
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield


app = FastAPI(title="Auth API", lifespan=lifespan)


# ── Registration ───────────────────────────────────────────────
@app.post("/auth/register", response_model=UserProfileResponse, status_code=201)
async def register(data: UserRegister, db: AsyncSession = Depends(get_db)):
    # Check for duplicate email
    result = await db.execute(select(User).where(User.email == data.email))
    if result.scalar_one_or_none():
        raise HTTPException(status.HTTP_409_CONFLICT, detail="Email already registered")

    # Check for duplicate username
    result = await db.execute(select(User).where(User.username == data.username))
    if result.scalar_one_or_none():
        raise HTTPException(status.HTTP_409_CONFLICT, detail="Username already taken")

    user = User(
        email=data.email,
        username=data.username,
        hashed_password=hash_password(data.password),  # ← never store raw password
    )
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return user


# ── Login ──────────────────────────────────────────────────────
@app.post("/auth/login", response_model=Token)
async def login(data: UserLogin, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.email == data.email))
    user = result.scalar_one_or_none()

    # Always run verify_password even if user not found (prevents timing attacks)
    # If user is None we pass a dummy hash — it will fail but take the same time
    dummy_hash = "$2b$12$dummy.hash.to.prevent.timing.attack.XXXXXXXXXX"
    password_valid = verify_password(
        data.password,
        user.hashed_password if user else dummy_hash
    )

    if not user or not password_valid:
        # Intentionally vague — don't reveal whether email exists
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid email or password",
        )

    token = create_access_token(data={"sub": str(user.id)})
    return Token(access_token=token)


# ── Profile (any authenticated user) ──────────────────────────
@app.get("/profile/me", response_model=UserProfileResponse)
async def get_my_profile(current_user: User = Depends(get_current_user)):
    # get_current_user already did all the auth work
    # This route just returns the user object
    return current_user


@app.patch("/profile/me", response_model=UserProfileResponse)
async def update_my_profile(
    data: UserProfileUpdate,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    updated = data.model_dump(exclude_unset=True)

    # Check uniqueness if email/username is being changed
    if "email" in updated:
        result = await db.execute(select(User).where(User.email == updated["email"]))
        if result.scalar_one_or_none():
            raise HTTPException(status.HTTP_409_CONFLICT, detail="Email already in use")

    if "username" in updated:
        result = await db.execute(select(User).where(User.username == updated["username"]))
        if result.scalar_one_or_none():
            raise HTTPException(status.HTTP_409_CONFLICT, detail="Username already taken")

    for key, value in updated.items():
        setattr(current_user, key, value)

    await db.commit()
    await db.refresh(current_user)
    return current_user


# ── Admin Routes ───────────────────────────────────────────────
@app.get("/admin/users", response_model=list[AdminUserResponse])
async def list_all_users(
    _admin: User = Depends(require_admin),  # enforces admin role
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(select(User))
    return result.scalars().all()


@app.get("/admin/users/{user_id}", response_model=AdminUserResponse)
async def get_user_by_id(
    user_id: int,
    _admin: User = Depends(require_admin),
    db: AsyncSession = Depends(get_db),
):
    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status.HTTP_404_NOT_FOUND, detail="User not found")
    return user


@app.patch("/admin/users/{user_id}/promote", response_model=AdminUserResponse)
async def promote_user(
    user_id: int,
    _admin: User = Depends(require_admin),
    db: AsyncSession = Depends(get_db),
):
    """Only admins can grant admin rights to others."""
    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status.HTTP_404_NOT_FOUND, detail="User not found")

    user.is_admin = True
    await db.commit()
    await db.refresh(user)
    return user
```

---

## Step 8: Create Your First Admin User

Since every new user defaults to `is_admin=False`, you need to seed one admin manually. Add this script:

```python
# seed_admin.py
import asyncio
from database import engine, Base, SessionLocal
from models import User
from auth import hash_password

async def create_admin():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    async with SessionLocal() as db:
        admin = User(
            email="admin@example.com",
            username="admin",
            hashed_password=hash_password("AdminPass123"),
            is_admin=True,
        )
        db.add(admin)
        await db.commit()
        print("Admin created: admin@example.com / AdminPass123")

asyncio.run(create_admin())
```

```bash
python seed_admin.py
```

---

## Step 9: Run and Test

```bash
uvicorn main:app --reload
```

Then open `http://localhost:8000/docs` — FastAPI auto-generates Swagger UI.

### Testing the full flow manually (curl):

**1. Register a user**
```bash
curl -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "bob@example.com", "username": "bob", "password": "mypassword123"}'
```

**2. Login and get a token**
```bash
curl -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "bob@example.com", "password": "mypassword123"}'

# Response: {"access_token": "eyJ...", "token_type": "bearer"}
```

**3. Use the token to access your profile**
```bash
curl http://localhost:8000/profile/me \
  -H "Authorization: Bearer eyJ..."
```

**4. Try an admin route as a regular user (should 403)**
```bash
curl http://localhost:8000/admin/users \
  -H "Authorization: Bearer eyJ..."   # Bob's token

# Response: {"detail": "Admin privileges required"}
```

**5. Login as admin and access it**
```bash
# Get admin token first, then:
curl http://localhost:8000/admin/users \
  -H "Authorization: Bearer <admin_token>"

# Response: list of all users WITH is_admin field
```

---

## What You Now Understand

| Concept | Where it lives in your code |
|---|---|
| Password hashing | `auth.py → hash_password / verify_password` |
| JWT creation | `auth.py → create_access_token` |
| JWT verification | `auth.py → decode_access_token` |
| Auth enforcement | `dependencies.py → get_current_user` |
| Role enforcement | `dependencies.py → require_admin` |
| Schema-level RBAC | `schemas.py → UserProfileResponse vs AdminUserResponse` |
| Route protection | `Depends(get_current_user)` or `Depends(require_admin)` |
| Timing attack prevention | `login()` — always runs `verify_password` even if user not found |

---

## Industry-Level Notes

**Things done right here that real systems do:**
- Vague error messages on login ("Invalid email or password", not "Email not found")
- Timing-safe password verification (always runs bcrypt even on missing user)
- Tokens expire (30 min default)
- `SECRET_KEY` from environment variable, not hardcoded
- `is_admin` not exposed in regular user responses

**What you'd add in a production system:**
- **Refresh tokens** — short-lived access token + long-lived refresh token stored in DB
- **Token revocation** — store invalidated JTI (JWT ID) claims in Redis
- **Rate limiting** on `/auth/login` to prevent brute force
- **Email verification** on registration
- **HTTPS only** — JWT over HTTP is insecure
- **Alembic** for database migrations instead of `create_all`
- **Audit logging** — record every login attempt, admin action

---

## Quick Reference: HTTP Status Codes for Auth

| Code | Meaning | When to use |
|---|---|---|
| `200 OK` | Success | GET, PATCH responses |
| `201 Created` | Resource created | Registration, POST |
| `401 Unauthorized` | Not authenticated | Missing/invalid/expired token |
| `403 Forbidden` | Authenticated but not authorized | Valid token, wrong role |
| `404 Not Found` | Resource missing | User ID not found |
| `409 Conflict` | Duplicate resource | Email/username already taken |
