# Phase 02: Backend Foundation

Build the FastAPI app skeleton with authentication and a health check. By the end of this phase, you can register a user, log in, and call authenticated endpoints.

**Prerequisite:** Phase 01 complete (project scaffolded, dependencies installed, config in place).

---

## FastAPI App Entry Point

Create `api/main.py`:

```python
import logging
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from motor.motor_asyncio import AsyncIOMotorClient
from beanie import init_beanie

from api.config import get_settings
from api.schemas.orm.user import User
from api.routes import auth

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
logger = logging.getLogger(__name__)

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    client = AsyncIOMotorClient(settings.mongodb_url)
    await init_beanie(
        database=client[settings.mongodb_db_name],
        document_models=[User],  # Add all Beanie documents here as they're created
    )
    logger.info("Connected to MongoDB database: %s", settings.mongodb_db_name)
    yield
    client.close()
    logger.info("Disconnected from MongoDB")

app = FastAPI(lifespan=lifespan)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(auth.router, prefix="/api/auth", tags=["auth"])

@app.get("/api/health")
async def health_check():
    return {"status": "ok"}
```

Key points:
- Logging is configured once here, before the app is created
- Beanie is initialized during the lifespan — register every Document model in `document_models`
- Add new routers with `app.include_router()` as resources are created in Phase 03

---

## User Model

Create `api/schemas/orm/user.py`:

```python
from datetime import datetime, timezone
from typing import Optional
from beanie import Document, Indexed
from pydantic import Field

class User(Document):
    username: Indexed(str, unique=True)
    email: Indexed(str, unique=True)
    hashed_password: str
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    updated_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))

    class Settings:
        name = "users"
```

Ensure `api/schemas/__init__.py` and `api/schemas/orm/__init__.py` exist (empty files).

---

## Auth Utilities

Create `api/utils/auth.py`:

```python
from datetime import datetime, timedelta, timezone
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError
from api.config import get_settings
from api.schemas.orm.user import User

ph = PasswordHasher()
security = HTTPBearer()

def hash_password(password: str) -> str:
    return ph.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    try:
        ph.verify(hashed_password, plain_password)
        return True
    except VerifyMismatchError:
        return False

def create_access_token(subject: str) -> str:
    settings = get_settings()
    expire = datetime.now(timezone.utc) + timedelta(minutes=settings.jwt_expire_minutes)
    payload = {"sub": subject, "exp": expire}
    return jwt.encode(payload, settings.jwt_secret, algorithm=settings.jwt_algorithm)

async def get_current_user(credentials: HTTPAuthorizationCredentials = Depends(security)):
    settings = get_settings()
    try:
        payload = jwt.decode(
            credentials.credentials, settings.jwt_secret,
            algorithms=[settings.jwt_algorithm]
        )
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")
    user = await User.get(user_id)
    if user is None:
        raise HTTPException(status_code=401, detail="User not found")
    return user
```

Key points:
- **Argon2** for password hashing (not bcrypt/passlib — bcrypt 5.x has compatibility issues)
- `get_current_user` is a FastAPI dependency used on every protected route

Ensure `api/utils/__init__.py` exists (empty file).

---

## Auth DTOs

Create `api/schemas/dto/auth.py`:

```python
from pydantic import BaseModel

class RegisterRequest(BaseModel):
    username: str
    email: str
    password: str

class LoginRequest(BaseModel):
    username: str
    password: str

class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"

class UserResponse(BaseModel):
    id: str
    username: str
    email: str
```

Ensure `api/schemas/dto/__init__.py` exists (empty file).

---

## Auth Routes

Create `api/routes/auth.py`:

```python
from fastapi import APIRouter, Depends, HTTPException
from api.schemas.orm.user import User
from api.schemas.dto.auth import RegisterRequest, LoginRequest, TokenResponse, UserResponse
from api.utils.auth import hash_password, verify_password, create_access_token, get_current_user

router = APIRouter()

@router.post("/register", response_model=UserResponse, status_code=201)
async def register(data: RegisterRequest):
    existing = await User.find_one(User.username == data.username)
    if existing:
        raise HTTPException(status_code=409, detail="Username already taken")
    existing_email = await User.find_one(User.email == data.email)
    if existing_email:
        raise HTTPException(status_code=409, detail="Email already registered")
    user = User(
        username=data.username,
        email=data.email,
        hashed_password=hash_password(data.password),
    )
    await user.insert()
    return UserResponse(id=str(user.id), username=user.username, email=user.email)

@router.post("/login", response_model=TokenResponse)
async def login(data: LoginRequest):
    user = await User.find_one(User.username == data.username)
    if not user or not verify_password(data.password, user.hashed_password):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    token = create_access_token(str(user.id))
    return TokenResponse(access_token=token)

@router.get("/me", response_model=UserResponse)
async def get_me(user=Depends(get_current_user)):
    return UserResponse(id=str(user.id), username=user.username, email=user.email)
```

Ensure `api/routes/__init__.py` exists (empty file).

---

## Verify

Start the dev server:

```bash
uv run uvicorn api.main:app --reload --port 8020
```

Test the endpoints:

```bash
# Health check
curl http://localhost:8020/api/health

# Register
curl -X POST http://localhost:8020/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","email":"test@example.com","password":"testpass123"}'

# Login
curl -X POST http://localhost:8020/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"testpass123"}'

# Get current user (use the token from login response)
curl http://localhost:8020/api/auth/me \
  -H "Authorization: Bearer <token>"
```

---

## Checklist

- [ ] `api/main.py` created with lifespan, CORS, health check
- [ ] `api/schemas/orm/user.py` — User document model
- [ ] `api/utils/auth.py` — password hashing, JWT, get_current_user
- [ ] `api/schemas/dto/auth.py` — request/response models
- [ ] `api/routes/auth.py` — register, login, /me
- [ ] All `__init__.py` files in place
- [ ] `curl /api/health` returns `{"status": "ok"}`
- [ ] Can register a user, login, and call `/api/auth/me` with the token
