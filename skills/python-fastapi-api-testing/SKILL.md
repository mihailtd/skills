---
name: python-fastapi-api-testing
description: Instructs the agent on writing asynchronous unit tests for FastAPI REST API endpoints using pytest and httpx.AsyncClient, creating reusable test fixtures, and generating test coverage reports.
---

# FastAPI API Testing Guidelines

You are an expert Python developer specializing in testing FastAPI applications. When asked to write unit tests, mock databases, create test fixtures, or generate coverage reports, you must adhere to the following rules:

## 1. Asynchronous Testing Setup

Because FastAPI routes are often asynchronous, your tests must support async execution.

- Use `pytest` alongside the `pytest-asyncio` and `httpx` libraries.
- Instruct the user to create a `pytest.ini` file with `asyncio_mode = True` to automatically run tests in asynchronous mode.
- Decorate all asynchronous test functions with `@pytest.mark.asyncio`.

## 2. Eliminating Repetition with Fixtures

Do not redefine application instances, database setups, or token generation within every single test. Use Pytest fixtures (`@pytest.fixture`) to create reusable data.

- **Test Client:** Create a `session` scoped fixture (e.g., in a `conftest.py` file) that returns an `httpx.AsyncClient` instance connected to the FastAPI `app`.
- **Database Mocks:** Yield the client within the fixture, and add teardown code to roll back or clean the database after test sessions.
- **Access Tokens:** Create a `module` scoped fixture that generates a reusable JWT token using your application's `create_access_token` utility so protected routes can be easily tested without re-authenticating.

## 3. Testing REST API Endpoints

When writing tests for CRUD endpoints, ensure you pass the `httpx.AsyncClient` fixture and any necessary token or mock data fixtures.

- **Headers:** For protected routes, pass the token fixture into a `headers` dictionary as `{"Authorization": f"Bearer {access_token}"}`.
- **Assertions:** Always assert the HTTP status code (e.g., `assert response.status_code == 200`) and the expected JSON response properties (e.g., `assert response.json()["title"] == payload["title"]`).

## 4. Test Coverage

When a user asks how to measure test coverage, recommend the `coverage` Python module.

- Advise running tests with `coverage run -m pytest`.
- Instruct the user to view terminal reports using `coverage report`.
- Instruct the user to view web-based reports using `coverage html`.

---

## Code Examples

Below are best-practice examples demonstrating these concepts.

**conftest.py (Application Client & Database Fixtures)**

```python
import asyncio
from collections.abc import AsyncGenerator

import httpx
import pytest
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker

from main import app
from database import get_db, Base

# Async test engine using PostgreSQL
TEST_DATABASE_URL = "postgresql+asyncpg://test:test@localhost:5432/testdb"
test_engine = create_async_engine(TEST_DATABASE_URL, echo=False)
TestSessionLocal = async_sessionmaker(test_engine, expire_on_commit=False)

@pytest.fixture(scope="session")
def event_loop():
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture(scope="session", autouse=True)
async def setup_database():
    """Create tables before tests, drop after."""
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await test_engine.dispose()

@pytest.fixture()
async def db_session() -> AsyncGenerator[AsyncSession, None]:
    """Provide a transactional database session that rolls back after each test."""
    async with TestSessionLocal() as session:
        async with session.begin():
            yield session
            await session.rollback()

@pytest.fixture()
async def default_client(db_session: AsyncSession) -> AsyncGenerator[httpx.AsyncClient, None]:
    """Provide an httpx.AsyncClient with the test database session injected."""
    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    async with httpx.AsyncClient(app=app, base_url="http://test") as client:
        yield client
    app.dependency_overrides.clear()
```

**test_routes.py (Tokens and Endpoint Testing)**

```python
import pytest
import httpx
from auth.jwt_handler import create_access_token

# Reusable access token fixture
@pytest.fixture(scope="module")
async def access_token() -> str:
    return create_access_token("testuser@packt.com")

# Testing a POST (Create) Endpoint
@pytest.mark.asyncio
async def test_post_event(default_client: httpx.AsyncClient, access_token: str) -> None:
    payload = {
        "title": "FastAPI Book Launch",
        "description": "Discussing the contents of the FastAPI book.",
        "location": "Google Meet"
    }

    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {access_token}"
    }

    response = await default_client.post("/event/new", json=payload, headers=headers)

    assert response.status_code == 200
    assert response.json()["message"] == "Event created successfully"

# Testing a GET (Read) Endpoint
@pytest.mark.asyncio
async def test_get_events_count(default_client: httpx.AsyncClient) -> None:
    response = await default_client.get("/event/")
    events = response.json()

    assert response.status_code == 200
    assert len(events) > 0
```
