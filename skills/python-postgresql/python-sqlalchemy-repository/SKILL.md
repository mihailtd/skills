---
name: python-sqlalchemy-repository

description: Instructs the agent on building an asynchronous data layer using SQLAlchemy 2.0 syntax (Mapped, mapped_column), handling database relationships and query optimization, and strictly encapsulating all database access behind the Repository pattern — as a closure-based factory function per python-clean-architecture-drivers, not an ABC-implementing class, when the project uses the python-clean-architecture package.
---

# Python SQLAlchemy Repository & Data Layer Guidelines

You are an expert Python developer specializing in Clean Architecture, database performance, and modern SQLAlchemy 2.0. When asked to define database models, construct queries, or interact with the database, you must adhere to the following highly detailed rules:

## 1. The Repository Pattern (Clean Architecture)
Do not leak SQLAlchemy implementation details (like `Session`, `select`, or ORM models) into the core business logic (Application/Domain layers).
*   **The Interface (Domain/Application Layer):** the required database operations (`get_by_id`, `save`, ...) are expressed as `Callable` type aliases (e.g., `GetOrder = Callable[[int], Order | None]`), not an abstract base class — see python-clean-architecture-dependency-inversion. There is no `class OrderRepository(ABC)` in this repo's style.
*   **The Adapter (Infrastructure Layer):** implemented as a factory function that opens over the `AsyncSession` (or a session-provider callable) and returns plain functions matching those `Callable` types — see python-clean-architecture-drivers for the closure-based pattern. **Not** a class implementing the ABC (`class PostgresOrderRepository(OrderRepository)`, shown below in section "Code Examples" as the *classic* reference pattern, is the shape to translate away from, not to copy directly, in a `python-clean-architecture` project).
*   This distinction matters specifically because it changes how the repository is tested: a closure-returned function needs only a stand-in function to test against (see python-clean-architecture-test-doubles), never `Mock(spec=OrderRepository)`.

## 2. Modern SQLAlchemy 2.0 Model Definition
Define your database schema using SQLAlchemy's modern, strongly-typed 2.0 declarative syntax.
*   **Base Class:** Inherit from `sqlalchemy.orm.DeclarativeBase` rather than using the older `declarative_base()` function.
*   **Columns:** Strictly use type-hinted `Mapped[T]` and assign them using `mapped_column()`.
*   Example: `id: Mapped[int] = mapped_column(primary_key=True, index=True)`. To allow nulls, use the union syntax `Mapped[str | None]`.

## 3. Relationship Patterns
Define relationships explicitly using the `relationship()` directive and `ForeignKey`.
*   **One-to-Many / Many-to-One:** Use `ForeignKey("parent_table.id")` on the child model. Define `Mapped[Parent]` and `Mapped[list[Child]]` on the respective models, using the `back_populates="attribute_name"` parameter to link them bidirectionally.
*   **Many-to-Many:** Do not rely on hidden tables. Explicitly define an association model/table (containing foreign keys to both parent tables). On the primary models, define the relationship using the `secondary="association_table_name"` argument.

## 4. Async Session Patterns
Strictly use asynchronous drivers (like `asyncpg` or `aiosqlite`) to avoid blocking the event loop during I/O-bound database operations.
*   Use `create_async_engine()` and `sessionmaker(class_=AsyncSession, ...)` to configure the connection.
*   Use the `async with` context manager when acquiring sessions or managing transactions (e.g., `async with session.begin():`).
*   Execute queries using the modern 2.0 execution style: `result = await session.execute(query)` followed by `result.scalars().all()` or `result.scalars().first()`.

## 5. Query Composition and Optimization
Always optimize queries to reduce memory footprint and database round-trips.
*   **Eager Loading (Avoid N+1):** Never loop over a result set to fetch related data (the N+1 query problem). Use `.options(joinedload(Model.relationship))` in your `select()` statement to fetch parent and child records efficiently in a single trip.
*   **Targeted Fetching:** Use `.options(load_only(Model.col1, Model.col2))` to fetch only specific columns when you do not need the entire row, saving memory.
*   **Explicit Joins:** When aggregating data across tables without needing the full ORM objects, compose explicit joins using `.join(OtherModel, Model.id == OtherModel.fk_id)`.

---

## Code Examples

Below are best-practice examples demonstrating how to combine SQLAlchemy 2.0, asynchronous streaming, and the Repository pattern.

**1. Modern Models & Relationships (Infrastructure Layer)**
```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class EventModel(Base):
    __tablename__ = "events"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    
    # Many-to-Many Relationship using a secondary association table
    sponsors: Mapped[list["SponsorModel"]] = relationship(
        secondary="sponsorships",
        back_populates="events"
    )

class SponsorModel(Base):
    __tablename__ = "sponsors"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(unique=True)
    
    events: Mapped[list["EventModel"]] = relationship(
        secondary="sponsorships",
        back_populates="sponsors"
    )

# Association Table for Many-to-Many
class SponsorshipModel(Base):
    __tablename__ = "sponsorships"
    
    event_id: Mapped[int] = mapped_column(ForeignKey("events.id"), primary_key=True)
    sponsor_id: Mapped[int] = mapped_column(ForeignKey("sponsors.id"), primary_key=True)
    amount: Mapped[float] = mapped_column(default=0.0)
```

**2. Query Optimization: Joins and Eager Loading**

```python
from sqlalchemy.future import select
from sqlalchemy.orm import joinedload, load_only
from sqlalchemy.ext.asyncio import AsyncSession

async def get_events_with_sponsors(db_session: AsyncSession) -> list[EventModel]:
    # Eager loading: joinedload prevents the N+1 query problem
    query = select(EventModel).options(joinedload(EventModel.sponsors))
    
    async with db_session as session:
        result = await session.execute(query)
        return result.scalars().all()

async def get_sponsor_contributions(db_session: AsyncSession, event_id: int):
    # Explicit Join: Selecting specific columns efficiently
    query = (
        select(SponsorModel.name, SponsorshipModel.amount)
        .join(SponsorshipModel, SponsorshipModel.sponsor_id == SponsorModel.id)
        .where(SponsorshipModel.event_id == event_id)
        .order_by(SponsorshipModel.amount.desc())
    )
    
    async with db_session as session:
        result = await session.execute(query)
        return result.fetchall()
```

**3. The Repository Pattern Implementation**

**3a. Classic pattern (ABC + concrete class)** — this is the pattern most Clean Architecture reference material shows. Recognize it, but do not copy it into a `python-clean-architecture` project:

```python
from abc import ABC, abstractmethod
from typing import Optional
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.future import select

# -- DOMAIN LAYER (No SQLAlchemy imports here) --
class EventEntity:
    def __init__(self, id: int, name: str):
        self.id = id
        self.name = name

class EventRepository(ABC):
    @abstractmethod
    async def get_by_id(self, event_id: int) -> Optional[EventEntity]:
        pass

# -- INFRASTRUCTURE LAYER (SQLAlchemy is isolated here) --
class PostgresEventRepository(EventRepository):
    def __init__(self, session: AsyncSession):
        self.session = session
        
    async def get_by_id(self, event_id: int) -> Optional[EventEntity]:
        query = select(EventModel).where(EventModel.id == event_id)
        result = await self.session.execute(query)
        db_model = result.scalars().first()
        
        if not db_model:
            return None
            
        # Map the ORM model back to the pure Domain Entity
        return EventEntity(id=db_model.id, name=db_model.name)
```

**3b. This repo's pattern (`Callable` type + closure)** — the same behavior, no `ABC`, no repository class, per python-clean-architecture-drivers:

```python
from dataclasses import dataclass
from typing import Callable
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.future import select

# -- DOMAIN LAYER (No SQLAlchemy imports here) --
@dataclass(frozen=True)
class EventEntity:
    id: int
    name: str

# The "port" — a Callable type, not an ABC
GetEvent = Callable[[int], "EventEntity | None"]

# -- INFRASTRUCTURE LAYER (SQLAlchemy isolated inside the closure) --
def make_sqlalchemy_event_repository(session: AsyncSession) -> GetEvent:
    async def get_event(event_id: int) -> EventEntity | None:
        query = select(EventModel).where(EventModel.id == event_id)
        result = await session.execute(query)
        db_model = result.scalars().first()
        if not db_model:
            return None
        # Map the ORM model back to the pure Domain Entity
        return EventEntity(id=db_model.id, name=db_model.name)
    return get_event

# Usage at the composition root (see python-clean-architecture-composition-root):
# get_event = make_sqlalchemy_event_repository(session)
```

---

## Related guidance

For the full functional-lite repository/driver pattern this skill's section 1 and 3b apply (Callable-typed ports, closure-based factory functions, no ABC, testing with stand-in functions instead of `Mock(spec=...)`), see the `python-clean-architecture` package:

- python-clean-architecture-drivers
- python-clean-architecture-dependency-inversion
- python-clean-architecture-composition-root
- python-clean-architecture-test-doubles
