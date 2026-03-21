# Phase 10: Agent Authentication & API Access

Make the app accessible to AI agents, scripts, and MCP servers. Adds a dedicated agent token endpoint with long-lived JWTs, exposes the OpenAPI schema for agent discovery, and documents the agent workflow.

**Prerequisite:** Phase 02 complete (auth system in place). Can be done at any point after Phase 02.

---

## Overview

Agents authenticate using the same JWT system as browser users, but with a dedicated endpoint that issues long-lived tokens (up to 1 year). This avoids needing a separate auth mechanism — agents get a Bearer token and use the exact same API endpoints as the frontend.

| | Browser Users | Agents |
|---|---|---|
| **Token endpoint** | `POST /api/auth/login` | `POST /api/auth/agent-token` |
| **Token format** | JWT (Bearer) | JWT (Bearer) — same format |
| **Token lifetime** | 7 days (default) | 1-365 days (configurable per request) |
| **Auth header** | `Authorization: Bearer <token>` | Same |
| **API access** | All endpoints | All endpoints — same routes |

---

## Agent Token DTOs

Add to `api/schemas/dto/auth.py`:

```python
from pydantic import BaseModel, Field

class AgentTokenRequest(BaseModel):
    username: str
    password: str
    expires_in_days: int = Field(default=30, ge=1, le=365)

class AgentTokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    expires_in_days: int
```

---

## Update Token Creation

Update `create_access_token` in `api/utils/auth.py` to accept an optional `expires_delta`:

```python
from datetime import datetime, timedelta, timezone
from typing import Optional

def create_access_token(subject: str, expires_delta: Optional[timedelta] = None) -> str:
    settings = get_settings()
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(minutes=settings.jwt_expire_minutes)
    payload = {"sub": subject, "exp": expire}
    return jwt.encode(payload, settings.jwt_secret, algorithm=settings.jwt_algorithm)
```

The browser login path continues to use the default expiry. The agent endpoint passes a custom `expires_delta`.

---

## Agent Token Endpoint

Add to `api/routes/auth.py`:

```python
from datetime import timedelta
from api.schemas.dto.auth import AgentTokenRequest, AgentTokenResponse

@router.post("/agent-token", response_model=AgentTokenResponse)
async def agent_token(data: AgentTokenRequest):
    user = await User.find_one(User.username == data.username)
    if not user or not verify_password(data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid username or password",
        )

    access_token = create_access_token(
        str(user.id), expires_delta=timedelta(days=data.expires_in_days)
    )

    return AgentTokenResponse(
        access_token=access_token,
        expires_in_days=data.expires_in_days,
    )
```

Key points:
- Authenticates with username + password (same credentials as browser login)
- Returns a JWT with configurable lifetime (default 30 days, max 365)
- The token is a standard Bearer JWT — `get_current_user` works identically
- All operations are scoped to the authenticated user

---

## Expose OpenAPI Schema for Agents

Update the FastAPI app in `api/main.py` to expose the API docs at an agent-friendly URL:

```python
app = FastAPI(
    title="MyApp API",
    description="MyApp API — used by the frontend and AI agents",
    version="0.1.0",
    lifespan=lifespan,
    docs_url="/api/agent",           # Swagger UI for agents
    openapi_url="/api/openapi.json", # Machine-readable schema
    redoc_url=None,                  # Disable redoc (one docs UI is enough)
)
```

Add a convenience endpoint that returns the schema directly:

```python
@app.get("/api/schema")
async def get_schema():
    return app.openapi()
```

This gives agents three ways to discover the API:
- **`/api/openapi.json`** — standard OpenAPI 3.x schema (machine-readable)
- **`/api/schema`** — same schema via a simple GET (useful for `curl` or `fetch`)
- **`/api/agent`** — interactive Swagger UI (useful for debugging)

---

## Agent Workflow

### 1. Get a token

```bash
curl -X POST https://myapp.example.com/api/auth/agent-token \
  -H "Content-Type: application/json" \
  -d '{"username": "agentuser", "password": "agentpass", "expires_in_days": 90}'
```

Response:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "expires_in_days": 90
}
```

### 2. Discover the API

```bash
curl https://myapp.example.com/api/openapi.json
```

Or visit `https://myapp.example.com/api/agent` in a browser for interactive docs.

### 3. Call endpoints

```bash
# List things
curl https://myapp.example.com/api/things \
  -H "Authorization: Bearer <access_token>"

# Create a thing
curl -X POST https://myapp.example.com/api/things \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "New thing", "description": "Created by agent"}'
```

### 4. Refresh when needed

Tokens don't auto-refresh. When a token expires, the agent requests a new one from `/api/auth/agent-token`. For long-running agents, request a 365-day token.

---

## Agent User Account

Create a dedicated user account for agents rather than sharing a human user's credentials:

```bash
curl -X POST https://myapp.example.com/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "claude-agent", "email": "agent@example.com", "password": "a-strong-random-password"}'
```

Store the credentials securely (e.g., in `.env` or a secrets manager). The agent's data is scoped to this user account — it can't see or modify other users' data.

---

## Checklist

- [ ] `AgentTokenRequest` and `AgentTokenResponse` DTOs added to `api/schemas/dto/auth.py`
- [ ] `create_access_token` updated to accept optional `expires_delta`
- [ ] `POST /api/auth/agent-token` endpoint added to `api/routes/auth.py`
- [ ] FastAPI app configured with `docs_url="/api/agent"` and `openapi_url="/api/openapi.json"`
- [ ] `GET /api/schema` convenience endpoint added
- [ ] Agent can get a long-lived token via `/api/auth/agent-token`
- [ ] Agent can call all API endpoints with the Bearer token
- [ ] OpenAPI schema accessible at `/api/openapi.json`
- [ ] Swagger UI accessible at `/api/agent`
