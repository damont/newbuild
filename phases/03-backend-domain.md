# Phase 03: Backend Domain

Add your app's database models, DTOs, and resource routes. This phase covers the patterns for building out the actual business logic of your application.

**Prerequisite:** Phase 02 complete (FastAPI app running, auth working).

---

## Database Models (Beanie Documents)

Place in `api/schemas/orm/`. Each document maps to a MongoDB collection.

```python
from datetime import datetime, timezone
from typing import Optional
from beanie import Document, Indexed
from pydantic import BaseModel, Field

class Step(BaseModel):
    """Nested model — stored inside the parent document, not its own collection."""
    id: str
    description: str
    completed: bool = False

class Thing(Document):
    name: str
    user_id: Indexed(str)
    description: Optional[str] = None
    steps: list[Step] = []
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    updated_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))

    class Settings:
        name = "things"  # MongoDB collection name
```

Key patterns:
- Use `Indexed()` for fields you query on frequently
- Nest related data (steps, references, history entries) as `BaseModel` lists inside the document rather than separate collections
- Keep `created_at` and `updated_at` on every document
- The inner `Settings` class with `name` sets the MongoDB collection name

**Important:** Register every new Document model in `api/main.py`'s `init_beanie()` call:

```python
from api.schemas.orm.thing import Thing

# In lifespan:
await init_beanie(
    database=client[settings.mongodb_db_name],
    document_models=[User, Thing],  # Add new models here
)
```

---

## DTOs (Request/Response Models)

Place in `api/schemas/dto/`. Separate from ORM models so the API shape can differ from the database shape.

```python
from pydantic import BaseModel
from typing import Optional

class ThingCreate(BaseModel):
    name: str
    description: Optional[str] = None

class ThingUpdate(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None

class ThingResponse(BaseModel):
    id: str
    name: str
    description: Optional[str]
    created_at: str
```

Pattern: one `Create`, one `Update` (all fields optional), one `Response` per resource.

---

## Routes

One file per resource in `api/routes/`. Use FastAPI's dependency injection for auth.

```python
from fastapi import APIRouter, Depends, HTTPException
from api.schemas.orm.thing import Thing
from api.schemas.dto.thing import ThingCreate, ThingUpdate, ThingResponse
from api.utils.auth import get_current_user

router = APIRouter()

def thing_to_response(thing: Thing) -> ThingResponse:
    return ThingResponse(
        id=str(thing.id),
        name=thing.name,
        description=thing.description,
        created_at=thing.created_at.isoformat(),
    )

@router.get("/", response_model=list[ThingResponse])
async def list_things(user=Depends(get_current_user), skip: int = 0, limit: int = 20):
    things = await Thing.find(Thing.user_id == str(user.id)).skip(skip).limit(limit).to_list()
    return [thing_to_response(t) for t in things]

@router.post("/", response_model=ThingResponse, status_code=201)
async def create_thing(data: ThingCreate, user=Depends(get_current_user)):
    thing = Thing(name=data.name, description=data.description, user_id=str(user.id))
    await thing.insert()
    return thing_to_response(thing)

@router.get("/{thing_id}", response_model=ThingResponse)
async def get_thing(thing_id: str, user=Depends(get_current_user)):
    thing = await Thing.get(thing_id)
    if not thing or thing.user_id != str(user.id):
        raise HTTPException(status_code=404, detail="Not found")
    return thing_to_response(thing)

@router.put("/{thing_id}", response_model=ThingResponse)
async def update_thing(thing_id: str, data: ThingUpdate, user=Depends(get_current_user)):
    thing = await Thing.get(thing_id)
    if not thing or thing.user_id != str(user.id):
        raise HTTPException(status_code=404, detail="Not found")
    update_data = data.model_dump(exclude_unset=True)
    if update_data:
        await thing.set(update_data)
    return thing_to_response(thing)

@router.delete("/{thing_id}", status_code=204)
async def delete_thing(thing_id: str, user=Depends(get_current_user)):
    thing = await Thing.get(thing_id)
    if not thing or thing.user_id != str(user.id):
        raise HTTPException(status_code=404, detail="Not found")
    await thing.delete()
```

Conventions:
- All routes are prefixed in `main.py` (e.g., `/api/things`)
- Return DTOs, not raw Beanie documents
- Use `Depends(get_current_user)` on every protected route
- Standard HTTP methods: GET (list/detail), POST (create), PUT (update), DELETE (delete)
- Pagination via `?skip=0&limit=20` query params on list endpoints

Register the new router in `api/main.py`:

```python
from api.routes import auth, things

app.include_router(things.router, prefix="/api/things", tags=["things"])
```

---

## Services Layer

Use a services layer when route handlers get complex (multi-step operations, external API calls, shared business logic). For simple CRUD, routes can call Beanie directly.

```python
# api/services/thing_service.py
import logging
from api.schemas.orm.thing import Thing

logger = logging.getLogger(__name__)

async def complete_all_steps(thing: Thing) -> Thing:
    """Example: business logic that doesn't belong in a route handler."""
    for step in thing.steps:
        step.completed = True
    await thing.save()
    logger.info("All steps completed for thing %s", thing.id)
    return thing
```

---

## Logging

Logging is already configured in `api/main.py` (Phase 02). In any module that needs logging, create a module-level logger:

```python
import logging
logger = logging.getLogger(__name__)
```

### What to log

- **User actions that change state**: created, updated, deleted, completed resources
- **Authentication events**: login, failed login, registration
- **Authorization failures**: user tried to access something they shouldn't
- **External service errors**: if the app calls another API and it fails
- **Startup/shutdown**: app started, connected to database, shutting down

```python
# Examples:
logger.info("Task %s marked complete by user %s", task_id, user.id)
logger.warning("User %s attempted to access task %s owned by another user", user.id, task_id)
logger.error("Failed to send notification for task %s: %s", task_id, str(e))

# Unexpected exceptions (include stack trace):
try:
    await some_operation()
except Exception:
    logger.exception("Unexpected error during some_operation")
    raise
```

### What NOT to log

- **Every request** — uvicorn already does this
- **Request/response bodies** — can contain passwords or personal data
- **Successful reads** — too noisy, no diagnostic value
- **Anything at DEBUG level in production** — keep log level at INFO

---

## Checklist

- [ ] Database models created in `api/schemas/orm/`
- [ ] All Document models registered in `api/main.py` `init_beanie()` call
- [ ] DTOs created in `api/schemas/dto/` (Create, Update, Response per resource)
- [ ] Routes created in `api/routes/` (one file per resource)
- [ ] Routes registered in `api/main.py` with `app.include_router()`
- [ ] All routes use `Depends(get_current_user)` for auth
- [ ] CRUD operations testable via curl
- [ ] Users can only access their own data
- [ ] Logging added for significant state changes
