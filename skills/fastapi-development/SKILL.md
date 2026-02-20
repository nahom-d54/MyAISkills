---
name: fastapi-development
description: Build high-performance FastAPI applications with async routes, validation, dependency injection, security, and automatic API documentation. Use when developing modern Python APIs with async support, automatic OpenAPI documentation, and high performance requirements.
---

# FastAPI Development

Build modern Python APIs with FastAPI. **Always write tests for every code component created.**

## Project Structure

```
app/
├─ api/v1/          # API routes by version
├─ core/            # Core utilities (auth, config)
├─ models/          # Pydantic schemas
├─ services/        # Business logic
├─ db/              # Database models and sessions
│  ├─ migrations/   # Alembic migrations
├─ tests/           # Test suite
│  ├─ unit/
│  ├─ integration/
├─ main.py
```

## Core Components

### 1. Application Setup (main.py)

**Key features:** Lifecycle management, middleware, exception handlers, health checks

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request, status
from fastapi.exceptions import RequestValidationError
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
import logging
import time

logger = logging.getLogger(__name__)

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Manage startup/shutdown."""
    logger.info("Starting up")
    # Initialize Redis, Sentry, Prometheus
    yield
    logger.info("Shutting down")

app = FastAPI(
    title="API",
    lifespan=lifespan,
    docs_url="/api/docs"
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.middleware("http")
async def add_process_time(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    response.headers["X-Process-Time"] = str(time.time() - start)
    return response

@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=422,
        content={"error": "Validation failed", "details": exc.errors()}
    )

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

**Tests to write:**
- Test lifespan startup/shutdown
- Test middleware adds process time header
- Test validation error handler formats errors correctly
- Test health endpoint returns 200

### 2. Pydantic Models (models.py)

**Key features:** Validation, field constraints, custom validators

```python
from pydantic import BaseModel, EmailStr, Field, field_validator
from typing import Optional
from datetime import datetime
from enum import Enum

class UserRole(str, Enum):
    ADMIN = "admin"
    USER = "user"

class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(..., min_length=8)
    first_name: str = Field(..., max_length=100)
    last_name: str = Field(..., max_length=100)

    @field_validator('password')
    @classmethod
    def validate_password(cls, v):
        if not any(c.isupper() for c in v):
            raise ValueError('Must contain uppercase')
        if not any(c.isdigit() for c in v):
            raise ValueError('Must contain digit')
        return v

class UserResponse(BaseModel):
    id: str
    email: str
    first_name: str
    last_name: str
    role: UserRole = UserRole.USER
    created_at: datetime

    class Config:
        from_attributes = True
```

**Tests to write:**
- Test valid user creation succeeds
- Test password validation rejects weak passwords
- Test email validation and normalization
- Test field constraints (min/max length)

### 3. Database Models (db/base.py)

**Key features:** Async SQLAlchemy, type annotations, relationships

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import declarative_base, Mapped, mapped_column
from sqlalchemy import String, DateTime, Enum as SQLEnum
from datetime import datetime
import uuid

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/db"

engine = create_async_engine(DATABASE_URL, echo=False)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
Base = declarative_base()

class User(Base):
    __tablename__ = "users"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    first_name: Mapped[str] = mapped_column(String(100))
    last_name: Mapped[str] = mapped_column(String(100))
    password_hash: Mapped[str] = mapped_column(String(255))
    role: Mapped[str] = mapped_column(SQLEnum(UserRole))
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

async def get_db() -> AsyncSession:
    async with async_session() as session:
        yield session
```

**Tests to write:**
- Test database connection
- Test user model creation and constraints
- Test unique email constraint
- Test default values (id, created_at, role)

### 4. Authentication (core/security.py)

**Key features:** JWT tokens, password hashing, dependencies

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import JWTError, jwt
from passlib.context import CryptContext
from datetime import datetime, timedelta
from typing import Annotated

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_HOURS = 24

pwd_context = CryptContext(schemes=["argon2"], deprecated="auto")
security = HTTPBearer()

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(user_id: str) -> str:
    expire = datetime.utcnow() + timedelta(hours=ACCESS_TOKEN_EXPIRE_HOURS)
    return jwt.encode({"sub": user_id, "exp": expire}, SECRET_KEY, ALGORITHM)

async def get_current_user(creds: Annotated[HTTPAuthorizationCredentials, Depends(security)]) -> str:
    try:
        payload = jwt.decode(creds.credentials, SECRET_KEY, algorithms=[ALGORITHM])
        user_id = payload.get("sub")
        if not user_id:
            raise HTTPException(401, "Invalid token")
        return user_id
    except JWTError:
        raise HTTPException(401, "Invalid token")
```

**Tests to write:**
- Test password hashing and verification
- Test JWT token creation and validation
- Test expired token rejection
- Test invalid token rejection
- Test get_current_user dependency

### 5. Service Layer (services/user_service.py)

**Key features:** Business logic separation, async queries, error handling

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from db.base import User
from models import UserCreate
from core.security import hash_password, verify_password
from typing import Optional
import logging

logger = logging.getLogger(__name__)

class UserService:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def create_user(self, data: UserCreate) -> User:
        user = User(
            email=data.email.lower(),
            password_hash=hash_password(data.password),
            first_name=data.first_name,
            last_name=data.last_name
        )
        self.db.add(user)
        await self.db.commit()
        await self.db.refresh(user)
        return user

    async def get_by_email(self, email: str) -> Optional[User]:
        result = await self.db.execute(select(User).where(User.email == email.lower()))
        return result.scalar_one_or_none()

    async def authenticate(self, email: str, password: str) -> Optional[User]:
        user = await self.get_by_email(email)
        if user and verify_password(password, user.password_hash):
            return user
        return None
```

**Tests to write:**
- Test create_user saves to database
- Test get_by_email finds existing user
- Test get_by_email returns None for non-existent
- Test authenticate with valid credentials
- Test authenticate with invalid credentials

### 6. API Routes (api/v1/auth.py)

**Key features:** Async endpoints, dependency injection, proper status codes

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession
from db.base import get_db
from models import UserCreate, UserResponse
from services.user_service import UserService
from core.security import create_access_token, get_current_user
from typing import Annotated

router = APIRouter(prefix="/auth")

@router.post("/register", response_model=UserResponse, status_code=201)
async def register(data: UserCreate, db: Annotated[AsyncSession, Depends(get_db)]):
    service = UserService(db)
    if await service.get_by_email(data.email):
        raise HTTPException(409, "Email already registered")
    return await service.create_user(data)

@router.post("/login")
async def login(email: str, password: str, db: Annotated[AsyncSession, Depends(get_db)]):
    service = UserService(db)
    user = await service.authenticate(email, password)
    if not user:
        raise HTTPException(401, "Invalid credentials")
    return {
        "access_token": create_access_token(str(user.id)),
        "token_type": "bearer"
    }

@router.get("/me", response_model=UserResponse)
async def get_me(user_id: Annotated[str, Depends(get_current_user)], db: Annotated[AsyncSession, Depends(get_db)]):
    service = UserService(db)
    user = await service.get_by_id(user_id)
    if not user:
        raise HTTPException(404, "User not found")
    return user
```

**Tests to write:**
- Test POST /register creates user (201)
- Test POST /register rejects duplicate email (409)
- Test POST /login returns token for valid credentials
- Test POST /login rejects invalid credentials (401)
- Test GET /me returns user with valid token
- Test GET /me rejects invalid token (401)

## Testing Setup

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from app.main import app
from app.db.base import Base

TEST_DB = "sqlite+aiosqlite:///:memory:"

@pytest.fixture
async def db_session():
    engine = create_async_engine(TEST_DB)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    
    async with AsyncSession(engine) as session:
        yield session
    
    await engine.dispose()

@pytest.fixture
async def client(db_session):
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac
```

## Best Practices

**DO:**
- ✅ Write tests for every component (models, services, routes)
- ✅ Use async/await for I/O operations
- ✅ Leverage Pydantic validation
- ✅ Use dependency injection for services
- ✅ Return proper HTTP status codes
- ✅ Use type hints everywhere
- ✅ Separate business logic into services
- ✅ Use environment variables for config

**DON'T:**
- ❌ Use synchronous database operations
- ❌ Trust user input without validation
- ❌ Store secrets in code
- ❌ Return database models directly in responses
- ❌ Implement business logic in route handlers