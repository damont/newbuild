# Phase 04: Backend Testing

Set up pytest with async fixtures and write integration tests for all API endpoints.

**Prerequisite:** Phase 03 complete (models, routes, and business logic in place).

---

## Test Structure

```
<appname>/
├── api/
├── tests/
│   ├── conftest.py          # Fixtures: app client, test DB, auth helpers
│   ├── test_auth.py         # Auth endpoint tests
│   └── test_<resource>.py   # One file per resource, matching api/routes/
├── pyproject.toml
```

---

## conftest.py

This is the most important file — it wires up the test database, creates the async HTTP client, and provides auth helpers.

```python
import pytest
import os
from httpx import AsyncClient, ASGITransport
from motor.motor_asyncio import AsyncIOMotorClient
from beanie import init_beanie

from api.main import app
from api.schemas.orm.user import User
from api.schemas.orm.thing import Thing  # Import ALL your Beanie documents
from api.utils.auth import hash_password, create_access_token

# Test database name — always suffixed with _test to avoid touching real data.
TEST_DB_NAME = "myapp_test"
MONGODB_URL = os.environ.get("MONGODB_URL", "mongodb://localhost:27017")


@pytest.fixture(scope="session")
def anyio_backend():
    return "asyncio"


@pytest.fixture(autouse=True)
async def setup_test_db():
    """Initialize Beanie with a test database before each test, drop it after."""
    client = AsyncIOMotorClient(MONGODB_URL)
    await init_beanie(
        database=client[TEST_DB_NAME],
        document_models=[User, Thing],  # ALL Beanie documents
    )
    yield
    await client.drop_database(TEST_DB_NAME)
    client.close()


@pytest.fixture
async def client():
    """Async HTTP client pointed at the FastAPI app."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c


@pytest.fixture
async def authenticated_client(client: AsyncClient):
    """Client with a valid auth token for a test user."""
    user = User(
        username="testuser",
        email="test@example.com",
        hashed_password=hash_password("testpass123"),
    )
    await user.insert()
    token = create_access_token(str(user.id))
    client.headers["Authorization"] = f"Bearer {token}"
    yield client
```

Key points:
- `TEST_DB_NAME` is hardcoded with a `_test` suffix — never accidentally wipes a real database
- `setup_test_db` is `autouse=True` — runs before every test and drops the entire test database afterward
- `authenticated_client` creates a real user and real JWT, exercising the actual auth pipeline

---

## Example Tests

### test_auth.py

```python
import pytest

@pytest.mark.anyio
async def test_health_check(client):
    res = await client.get("/api/health")
    assert res.status_code == 200
    assert res.json() == {"status": "ok"}

@pytest.mark.anyio
async def test_register(client):
    res = await client.post("/api/auth/register", json={
        "username": "newuser",
        "email": "new@example.com",
        "password": "password123",
    })
    assert res.status_code == 201
    data = res.json()
    assert data["username"] == "newuser"
    assert "id" in data

@pytest.mark.anyio
async def test_register_duplicate_username(client):
    await client.post("/api/auth/register", json={
        "username": "dup",
        "email": "dup1@example.com",
        "password": "password123",
    })
    res = await client.post("/api/auth/register", json={
        "username": "dup",
        "email": "dup2@example.com",
        "password": "password123",
    })
    assert res.status_code == 409

@pytest.mark.anyio
async def test_login(client):
    await client.post("/api/auth/register", json={
        "username": "loginuser",
        "email": "login@example.com",
        "password": "password123",
    })
    res = await client.post("/api/auth/login", json={
        "username": "loginuser",
        "password": "password123",
    })
    assert res.status_code == 200
    assert "access_token" in res.json()

@pytest.mark.anyio
async def test_login_bad_password(client):
    await client.post("/api/auth/register", json={
        "username": "badpw",
        "email": "bad@example.com",
        "password": "password123",
    })
    res = await client.post("/api/auth/login", json={
        "username": "badpw",
        "password": "wrongpassword",
    })
    assert res.status_code == 401

@pytest.mark.anyio
async def test_me(authenticated_client):
    res = await authenticated_client.get("/api/auth/me")
    assert res.status_code == 200
    assert res.json()["username"] == "testuser"
```

### test_things.py (resource example)

```python
import pytest

@pytest.mark.anyio
async def test_create_thing(authenticated_client):
    res = await authenticated_client.post("/api/things", json={
        "name": "Test thing",
        "description": "A test",
    })
    assert res.status_code == 201
    data = res.json()
    assert data["name"] == "Test thing"
    assert "id" in data

@pytest.mark.anyio
async def test_list_things(authenticated_client):
    await authenticated_client.post("/api/things", json={"name": "Item 1"})
    await authenticated_client.post("/api/things", json={"name": "Item 2"})
    res = await authenticated_client.get("/api/things")
    assert res.status_code == 200
    assert len(res.json()) == 2

@pytest.mark.anyio
async def test_list_things_requires_auth(client):
    res = await client.get("/api/things")
    assert res.status_code == 401 or res.status_code == 403

@pytest.mark.anyio
async def test_users_only_see_own_things(authenticated_client, client):
    # Create a thing as the authenticated user
    await authenticated_client.post("/api/things", json={"name": "Private"})

    # Second user shouldn't see it
    from api.schemas.orm.user import User
    from api.utils.auth import hash_password, create_access_token

    user2 = User(
        username="other",
        email="other@example.com",
        hashed_password=hash_password("pass"),
    )
    await user2.insert()
    token2 = create_access_token(str(user2.id))
    client.headers["Authorization"] = f"Bearer {token2}"

    res = await client.get("/api/things")
    assert res.status_code == 200
    assert len(res.json()) == 0
```

---

## What to Test

Focus on route-level integration tests — they cover routing, auth, validation, and database logic in one shot:

- **Auth flow**: register, login, token validation, bad credentials rejected
- **CRUD per resource**: create, list, get by ID, update, delete
- **Authorization**: user A can't access user B's data
- **Business logic edge cases**: ordering, status transitions, etc.

Skip testing Pydantic validation in isolation (FastAPI handles that) and skip testing individual database queries (the route tests cover them).

---

## pytest Configuration

Add to `pyproject.toml`:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

This avoids needing `@pytest.mark.anyio` on every test — all async test functions are treated as async automatically.

---

## Running Tests

```bash
uv run pytest              # run all tests
uv run pytest -v           # verbose output
uv run pytest tests/test_auth.py   # single file
uv run pytest -x           # stop on first failure
```

If MongoDB is not on localhost:
```bash
MONGODB_URL=mongodb://192.168.1.50:27017 uv run pytest
```

---

## Checklist

- [ ] `tests/conftest.py` created with test DB, client, and authenticated_client fixtures
- [ ] Test files created for auth and each resource
- [ ] Auth tests cover: register, login, bad credentials, /me
- [ ] Resource tests cover: CRUD operations, auth required, user isolation
- [ ] `pyproject.toml` has `asyncio_mode = "auto"`
- [ ] `uv run pytest` passes all tests
