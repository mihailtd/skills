---
name: python-fastapi-dependency-injection
description: Instructs the agent on leveraging FastAPI's Depends class to create reusable dependency functions and inject them into path operation function arguments to enforce runtime conditions like database session preparation and authentication.
---

# FastAPI Dependency Injection Guidelines

You are an expert Python developer specializing in FastAPI. When asked to implement shared logic, mandate pre-route conditions, handle database sessions, or secure routes, you must heavily leverage FastAPI's Dependency Injection system using the `Depends` class. Adhere to the following rules:

## 1. Creating Reusable Dependency Functions

Dependency injection is a pattern where a path operation function receives an instance variable (or fulfills a condition) needed for its execution [2].

- Define dependencies as standalone functions or classes that the framework will execute at runtime [3].
- **Database Sessions:** When preparing a service like a database connection, write a dependency function that sets up the database and yields the session (e.g., `yield session`) [4, 5].
- **Authentication/Authorization:** Write dependency functions that extract headers, verify JSON Web Tokens (JWT), and return the validated user [3, 6].

## 2. Injecting Dependencies with `Depends`

Dependencies are dormant until explicitly injected into their place of use [7].

- Always import the `Depends` class from the `fastapi` library [8].
- Pass your truth source (the dependency function) as an argument to `Depends` (e.g., `Depends(get_session)`) [9].
- Inject this directly into the path operation function's arguments [10]. This automatically mandates that the dependency condition is fully satisfied before the main route logic is allowed to execute [9].
- Doing this eliminates the need to manually instantiate database sessions or decode tokens inside every single route body [3].

---

## Code Examples

Below are best-practice examples demonstrating how to use `Depends` to prepare an async database session and enforce authentication checks.

**dependencies.py (Defining the Dependency Functions)**

```python
from collections.abc import AsyncGenerator

from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import jwt, JWTError
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker

# Database Setup (async PostgreSQL)
engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db", echo=False)
async_session = async_sessionmaker(engine, expire_on_commit=False)

# 1. Dependency for yielding an async database session
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        yield session

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/user/signin")
SECRET_KEY = "YOUR_SUPER_SECRET_KEY"

# 2. Dependency for authenticating a user
async def authenticate(token: str = Depends(oauth2_scheme)) -> str:
    if not token:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Sign in for access",
        )
    try:
        decoded_token = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return decoded_token["user"]
    except JWTError:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Invalid token",
        )
```

**routes.py (Injecting the Dependencies into Routes)**

```python
from fastapi import APIRouter, Depends
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from dependencies import get_db, authenticate
from models import Event, EventCreate, EventResponse

event_router = APIRouter()

# Injecting get_db to prepare the async database session
@event_router.get("/event", response_model=list[EventResponse])
async def retrieve_all_events(db: AsyncSession = Depends(get_db)) -> list[EventResponse]:
    result = await db.execute(select(Event))
    events = result.scalars().all()
    return events

# Injecting 'authenticate' (to mandate header checks) AND 'get_db'
@event_router.post("/event/new", response_model=dict)
async def create_event(
    new_event: EventCreate,
    user: str = Depends(authenticate),
    db: AsyncSession = Depends(get_db),
) -> dict:
    # This logic only runs if the 'authenticate' dependency succeeds
    event = Event(**new_event.model_dump(), creator=user)
    db.add(event)
    await db.commit()
    await db.refresh(event)
    return {"message": "Event created successfully"}
```
