---
name: python
description: >
  General Python programming approach and standards for this project.
  Defines the core stack, tooling, the PostgreSQL vs MongoDB decision, and the
  "functional-lite, data classes over OOP" style philosophy.
  USE THIS SKILL for any Python task. It references specialized domain skills
  for deeper guidance on specific topics (FastAPI, Alembic, async, functional
  programming, dependency management).
---

# Python Programming — General Approach

This is the master Python reference skill. It captures the non-negotiable defaults,
the high-level architectural decisions (which database, which style), and the
"functional-lite" philosophy used across all Python code in this project.
Read this first; follow the linked domain skills for deeper guidance.

---

## Core Stack

| Concern | Choice |
|---|---|
| Package manager & task runner | `uv` |
| Packaging manifest | `pyproject.toml` only |
| Linter / formatter | `ruff` |
| Type checker | `ty` |
| Data processing library | `polars` (not `pandas`) |
| Backend framework | FastAPI |
| Database | PostgreSQL or MongoDB — see the decision guide below |
| Relational path | SQLAlchemy (async) + Alembic |
| Document path | Beanie (default) or PyMongo's async driver |
| Runtime model | `asyncio` — always async |
| Programming style | Functional-lite — data classes + functions, not OOP |

---

## Package Management — Always `uv`

**Never** use `pip`, `pip-tools`, `poetry`, or `pipenv` directly.
All installation, running, and scripting uses `uv` and its native commands.

```bash
# Install dependencies
uv sync

# Add a dependency
uv add <package>

# Add a dev dependency
uv add --dev <package>

# Run a script or module
uv run python script.py
uv run alembic upgrade head
uv run pytest

# Upgrade dependencies
uv lock --upgrade
```

Pin all production dependencies with `==` for reproducible builds.
Do not use `requirements.txt`, `requirements-dev.txt`, or any requirements directory for dependency management. All package metadata and dependency configuration belongs in `pyproject.toml`.
Use `uv` for every operation that touches packages, tools, or builds. Never run `pip`, `pip install`, `python -m pip`, `poetry`, or `pipenv` directly in this project.
See the **python-upgrade-dependencies** skill for the full upgrade workflow, and the **python-project-setup** skill for full project scaffolding (`pyproject.toml`, pre-commit, CI wiring).
For library-specific guidance, see the **python-library-creation** skill.

---

## Linting and Formatting — Astral Stack Mandatory

The Astral stack is mandatory for Python projects in this repository. `ruff` is the canonical linter and formatter.

- Add `ruff` as a dev dependency: `uv add --dev ruff`
- Use `uv run ruff format` to format code.
- Use `uv run ruff check --fix` to run linting and automatically fix issues where possible.
- Document these commands in your project scripts or README.

`ruff` is the single source of truth for formatting and style consistency across code, tests, and configuration.

---

## Type Checking — `ty` Preferred

Type annotations are required. `ty` is the preferred type checker for this repository.

- Add `ty` as a dev dependency: `uv add --dev ty`
- Run type checks with `uv run ty check`
- Use `uv run --with ty ty check` to validate the toolchain if you want to compare behaviors.

The repository may still experiment with other checkers such as `mypy` or `pyrefly`, but `ty` is the preferred primary checker and should be the one enforced in CI.

---

## Data Processing — Polars vs DuckDB

This repository prefers `polars` for in-process data engineering and `DuckDB` for SQL-first analytics and large file querying.

- Use **Polars** when the agent is acting as a **Speedster**:
  - Rust-powered, multithreaded DataFrame processing on a single machine.
  - Best for complex pipeline chaining, feature engineering, joins, window functions, and programmatic transformations.
  - Prioritizes throughput and compute speed, with stricter schema and type safety.
  - Excellent for model preprocessing and integration with Python ML tooling.

- Use **DuckDB** when the agent is acting as a **Librarian**:
  - In-process SQL database optimized for analytical OLAP queries.
  - SQL-first intent is more reliable for translating natural language to code.
  - Better for memory-constrained workloads: DuckDB enforces strict memory limits and can safely query very large datasets.
  - Ideal for zero-ETL direct cloud querying of remote files in S3 or Google Cloud via URL.

### Decision heuristic

If the task is:

- Exploring new files or cloud storage → **DuckDB**
  - Best at direct file scanning and SQL exploration.
- Filtering or sorting massive datasets → **Polars**
  - Often dominates in raw filtering and loading speed on single nodes.
- Collaborating or handoff → **DuckDB**
  - SQL is easier for humans and agents to maintain and debug.
- Model preprocessing → **Polars**
  - Better integration with Python ML tools and schema-safe pipelines.

---

## Backend — FastAPI

FastAPI is the only supported backend framework. Key conventions:

- **Always use `async def`** for route handlers and dependencies.
- Use **Pydantic v2** models for all request/response shapes.
- Declare dependencies with `Depends()` — never reach for global state inside handlers.
- Return typed response models; never return raw dicts from route handlers.
- Routers live in their own modules; `main.py` only mounts routers and configures middleware.

```python
# ✅ Correct pattern
@router.get("/items/{item_id}", response_model=ItemResponse)
async def get_item(item_id: int, db: AsyncSession = Depends(get_db)) -> ItemResponse:
    return await item_service.get(db, item_id)
```

For structuring, testing, security, and OpenAPI documentation refer to the
FastAPI domain skills: **python-fastapi-project-structuring**, **python-fastapi-routing-validation**,
**python-fastapi-dependency-injection**, **python-fastapi-response-error-handling**,
**python-fastapi-security-authentication**, **python-fastapi-security-attack-resistance**,
**python-fastapi-api-testing**, **python-fastapi-openapi-documentation**.

---

## Database — PostgreSQL vs MongoDB

Pick one datastore path per project (or per bounded context in a larger system) —
don't mix ORMs/ODMs within the same service.

- Use **PostgreSQL + SQLAlchemy + Alembic** when:
  - Data is naturally relational — entities reference each other and you need real joins.
  - You need transactions spanning multiple entities/tables with strong consistency guarantees.
  - The schema is relatively stable and you want it enforced and versioned (DDL + Alembic migrations).
  - You'll run reporting/analytics SQL directly against the primary store.

- Use **MongoDB** when:
  - Data is naturally document-shaped — nested, variable, or maps closely to your API's JSON payloads.
  - The schema evolves frequently per-document and you don't want a migration gate on every change.
  - You need high write throughput or horizontal scale via sharding (see the **database-mongodb** package for sharding/replication skills).
  - Cross-entity transactions are rare-to-none in your access patterns.

### Decision heuristic

If the task is:

- Modeling entities with strict relationships and referential integrity → **PostgreSQL**
- Storing nested/variable JSON-like records (events, logs, user-generated content, catalogs with per-item fields) → **MongoDB**
- Running multi-table transactions or complex reporting SQL → **PostgreSQL**
- Iterating fast on document shape without a migration for every field → **MongoDB**
- Uncertain / mixed → default to **PostgreSQL**; it's the safer general-purpose choice.

### Relational path — PostgreSQL + SQLAlchemy (async)

- **Always use the async engine and session** (`AsyncEngine`, `AsyncSession`).
- Never call synchronous SQLAlchemy APIs from async code.
- Use `select()`, `insert()`, `update()`, `delete()` from `sqlalchemy` — not raw SQL strings.
- Manage the session lifecycle with `async with AsyncSession(engine) as session`.
- Define models with `DeclarativeBase`; use `Mapped[T]` typed column annotations (SQLAlchemy 2.x style).

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

engine = create_async_engine("postgresql+asyncpg://...", echo=False)

class Base(DeclarativeBase):
    pass

class Item(Base):
    __tablename__ = "items"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
```

Schema changes go through Alembic — see **Database Migrations — Alembic** below.
For repository-layer patterns on top of SQLAlchemy, see the **python-sqlalchemy-repository**
skill (**python-postgresql** package).

### Document path — MongoDB

Within MongoDB, choose the driver layer:

- **Beanie** (default) when you want Pydantic-model ergonomics on documents —
  `Document` subclasses feel like SQLAlchemy declarative models, with automatic
  validation and index declarations colocated with the model. Closest ergonomic
  parity with the relational path above; prefer this unless you have a reason not to.
- **PyMongo's async driver** when you need thin, direct control over queries and
  aggregation pipelines, are doing bulk/aggregation-heavy work, or Beanie's
  abstraction gets in the way.

```python
# Beanie — Pydantic-model ergonomics over MongoDB
from beanie import Document, Indexed

class Item(Document):
    name: Indexed(str)
    active: bool = True

    class Settings:
        name = "items"

# await Item.find(Item.active == True).to_list()
```

```python
# PyMongo's async client — direct driver access
from pymongo import AsyncMongoClient

client = AsyncMongoClient("mongodb://...")
items = client["app"]["items"]

# await items.find({"active": True}).to_list()
```

MongoDB itself is schemaless and has no DDL migration tool — but if you're on
Beanie, it ships a real migration framework of its own (`@iterative_migration()`,
`@free_fall_migration()`, and a `beanie migrate` CLI), functionally equivalent
to Alembic for the document path. Use it the same way Alembic is mandatory for
the relational path — don't fall back to untracked one-off scripts when Beanie
already gives you versioned, forward/backward migrations. See the
**python-beanie-migrations** skill for the full workflow. (If you're on raw
PyMongo without Beanie, there genuinely is no equivalent tool — version document
shape changes with explicit migration scripts and/or a `schema_version` field
instead.)

For document modeling, `init_beanie` setup, CRUD, querying, and Link/BackLink
relations, see the **python-beanie-documents** skill (**python-mongodb** package).

For MongoDB *database* concepts (aggregation, indexing, sharding, schema design,
transactions), see the **database-mongodb** package.

For PostgreSQL *database* concepts (indexing, partitioning, security, JSON/JSONB,
performance), see the **database-postgresql** package.

---

## Database Migrations — Alembic (relational path only)

Applies when the project uses PostgreSQL + SQLAlchemy. Alembic manages all schema
changes. Never mutate the database schema manually.

Key rules:
- Migrations are **immutable** once applied to any non-local environment.
- Every migration must be **idempotent** where possible.
- Use `uv run alembic` for all Alembic commands.

```bash
uv run alembic revision --autogenerate -m "add_items_table"
uv run alembic upgrade head
uv run alembic history
uv run alembic current
```

For full migration authoring rules and RLS enforcement see the **python-alembic-migrations** skill.
For rebasing local-only migration branches see the **python-alembic-migration-rebase** skill.
(Both live in the **python-postgresql** package, alongside **python-sqlalchemy-repository**.)

---

## Async — Always

Every I/O operation **must** be async. There is no place for blocking calls in
the request path. This applies regardless of which database path you chose above.

- Use `asyncio.gather()` for concurrent independent coroutines.
- Use `asyncio.TaskGroup` (Python 3.11+) when structured concurrency and error
  propagation matter.
- Use `asyncio.Queue` or `asyncio.Semaphore` for back-pressure and rate limiting.
- Never call `time.sleep()` — use `await asyncio.sleep()`.
- Never use `requests` — use `httpx.AsyncClient` or `aiohttp`.

Deeper guidance: **python-asyncio-concurrent-web-requests**, **python-asyncio-synchronization**,
and **python-concurrent-processing** (CPU-bound work → `ProcessPoolExecutor` + asyncio integration).

---

## Functional-Lite Style

The codebase uses a "functional-lite" approach: apply functional techniques
**where they reduce complexity and improve clarity**, but do not enforce
theoretical purity for its own sake.

### What we DO

| Technique | When to use |
|---|---|
| Pure functions | Default for business logic. No side effects unless necessary. |
| Immutable data | Use frozen dataclasses / Pydantic models for value objects. |
| Higher-order functions | Prefer `map`, `filter`, `functools.reduce` over imperative loops for data pipelines. |
| Generators / lazy sequences | Large or streaming datasets — avoid building full lists in memory. |
| `functools` utilities | `lru_cache`, `cache`, `partial`, `wraps` — use freely. |
| Function composition | Chain small, focused functions in data transformation pipelines. |
| Descriptive decorators | Express cross-cutting concerns (auth, caching, retries) as decorators. |

### What we DON'T force

| Technique | Reason to avoid unless clearly beneficial |
|---|---|
| Monads (`Maybe`, `Either`, `Result`) | Adds cognitive overhead; use exceptions + typed returns instead. |
| Point-free style | Hurts readability for most teammates. |
| Strict immutability everywhere | SQLAlchemy/Beanie models and async state are inherently mutable — fight that only where it matters. |
| `pymonad` / `returns` library | Opt-in only; do not impose on shared code without team agreement. |

```python
# ✅ Functional-lite: pipeline of focused pure functions
from functools import reduce

def parse_items(raw: list[dict]) -> list[Item]:
    return [Item(**r) for r in raw if r.get("active")]

def enrich(item: Item, overrides: dict) -> Item:
    return item.model_copy(update=overrides.get(item.id, {}))

def process(raw: list[dict], overrides: dict) -> list[Item]:
    return [enrich(item, overrides) for item in parse_items(raw)]
```

### Data over objects — `@dataclass` + functions, not OOP classes

Functional-lite means the default *shape* of business logic is **data classes
(or Pydantic models) plus free functions that operate on them** — not classes
with instance methods and inheritance hierarchies.

- **Default to `@dataclass`** (frozen where practical — see **python-immutable-data-design**)
  or a Pydantic `BaseModel` at validated boundaries, for anything that's just a
  bag of state: value objects, DTOs, function inputs/outputs.
- **Attach behavior as free functions** that take the dataclass as a parameter
  and return a new value, rather than as methods on the class. This keeps
  functions independently testable and composable in pipelines (see above).
- **Avoid designing services or business logic as OOP classes** (`class OrderProcessor`
  with a dozen methods and mutable instance state) as the default shape. Prefer a
  module of functions operating on plain data.
- **Reserve real OOP classes** for cases where the framework requires it or you're
  modeling genuine encapsulated statefulness — not business logic:
  - Framework-mandated models: SQLAlchemy `DeclarativeBase` subclasses, Beanie `Document`
    subclasses, Pydantic `BaseModel`/FastAPI dependency-provider classes, custom `Exception` types.
  - Objects that truly own mutable, encapsulated state across calls: a connection
    pool, a stateful parser, a cache client.

```python
# ✅ Preferred: dataclass + functions
from dataclasses import dataclass, replace

@dataclass(frozen=True)
class Order:
    id: int
    total: float
    status: str

def apply_discount(order: Order, pct: float) -> Order:
    return replace(order, total=order.total * (1 - pct))

def mark_paid(order: Order) -> Order:
    return replace(order, status="paid")
```

```python
# ❌ Avoid: OOP class as the default shape for business logic
class OrderProcessor:
    def __init__(self, order: Order):
        self.order = order

    def apply_discount(self, pct: float) -> None:
        self.order.total *= (1 - pct)

    def mark_paid(self) -> None:
        self.order.status = "paid"
```

Deeper guidance — consolidated and specialized functional skills: **python-higher-order-functools**
(map, filter, max/min, `@cache`, `partial`, `reduce`, `@singledispatch`), **python-lazy-evaluation-pipelines**
(`yield`, `yield from`, generator pipelines), **python-immutable-data-design** (`NamedTuple`, frozen
dataclass, `pyrsistent`, context managers), **python-functional-programming-decorator-design**,
**python-functional-programming-web-services** (FastAPI functional handlers), **python-functional-programming-itertools-combinatorics**,
**python-functional-programming-recursion-tco**.

---

## General Code Conventions

- **Python 3.11+** minimum. Use modern syntax: `match`, `TypeAlias`, `Self`, etc.
- **Type-annotate everything** — function signatures, variables, return types.
- **No `Any`** unless integrating with an untyped third-party boundary.
- Data shapes default to `@dataclass(frozen=True)` or Pydantic `BaseModel` — see
  "Data over objects" above.
- Keep functions small and single-purpose. If a function needs a comment to
  explain *what* it does (not *why*), split it.
- Avoid global mutable state. Pass dependencies explicitly (constructor or `Depends()`).
- Use `logging` — not `print`. Configure at app startup; never call
  `basicConfig()` inside library code. See the **python-configuration-observability**
  skill for full logging, tracing, and configuration guidance.

---

## Skill Map

Every group below names the **package** it lives in — install that package to get it
(see the root repo README for install commands per package).

```
python/SKILL.md  ← you are here (master reference, package: python-core)
│
├── Backend Framework                            [package: python-fastapi]
│   ├── python-fastapi-project-structuring/SKILL.md
│   ├── python-fastapi-routing-validation/SKILL.md
│   ├── python-fastapi-dependency-injection/SKILL.md
│   ├── python-fastapi-response-error-handling/SKILL.md
│   ├── python-fastapi-security-authentication/SKILL.md
│   ├── python-fastapi-security-attack-resistance/SKILL.md
│   ├── python-fastapi-api-testing/SKILL.md
│   └── python-fastapi-openapi-documentation/SKILL.md
│
├── Relational Database & Migrations             [package: python-postgresql]
│   ├── python-sqlalchemy-repository/SKILL.md
│   ├── python-alembic-migrations/SKILL.md
│   └── python-alembic-migration-rebase/SKILL.md
│
├── Document Database (Beanie)                   [package: python-mongodb]
│   ├── python-beanie-documents/SKILL.md    (modeling, init_beanie, CRUD, Link/BackLink)
│   └── python-beanie-migrations/SKILL.md   (iterative/free-fall migrations, beanie CLI)
│
├── Async & Concurrency                          [package: python-core]
│   ├── python-asyncio-concurrent-web-requests/SKILL.md
│   ├── python-asyncio-synchronization/SKILL.md
│   └── python-concurrent-processing/SKILL.md   (ProcessPool + asyncio integration)
│
├── Consolidated Core Skills                     [package: python-core]
│   ├── python-configuration-observability/SKILL.md   (config + logging + tracing)
│   ├── python-data-structures-type-system/SKILL.md   (collections + types + Pydantic)
│   ├── python-lazy-evaluation-pipelines/SKILL.md     (generators + map/filter/reduce)
│   ├── python-higher-order-functools/SKILL.md        (HOFs + cache + partial + singledispatch)
│   └── python-immutable-data-design/SKILL.md         (NamedTuple + frozen DC + context mgrs)
│
├── Functional Programming (specialized)         [package: python-core]
│   ├── python-functional-programming-decorator-design/SKILL.md
│   ├── python-functional-programming-web-services/SKILL.md
│   ├── python-functional-programming-itertools-combinatorics/SKILL.md
│   └── python-functional-programming-recursion-tco/SKILL.md
│
├── Library Creation                             [package: python-core]
│   ├── python-library-creation/SKILL.md
│   └── python-project-setup/SKILL.md
│
├── Polars                                       [package: python-polars]
│   ├── python-polars-eager-api/SKILL.md
│   ├── python-polars-lazy-api/SKILL.md
│   ├── python-polars-attributes-methods/SKILL.md
│   ├── python-polars-aggregation-methods/SKILL.md
│   ├── python-polars-computation-methods/SKILL.md
│   ├── python-polars-descriptive-methods/SKILL.md
│   ├── python-polars-groupby-methods/SKILL.md
│   ├── python-polars-exporting-methods/SKILL.md
│   ├── python-polars-manipulation-methods/SKILL.md
│   ├── python-polars-reshaping/SKILL.md
│   ├── python-polars-miscellaneous-methods/SKILL.md
│   ├── python-polars-expressions-overview/SKILL.md
│   ├── python-polars-expression-selection/SKILL.md
│   ├── python-polars-column-operations/SKILL.md
│   ├── python-polars-expression-creation/SKILL.md
│   ├── python-polars-expression-combining/SKILL.md
│   ├── python-polars-expression-operations/SKILL.md
│   ├── python-polars-row-operations/SKILL.md
│   ├── python-polars-expression-filtering/SKILL.md
│   ├── python-polars-expression-groupby-aggregation/SKILL.md
│   ├── python-polars-expression-sorting/SKILL.md
│   ├── python-polars-expression-naming/SKILL.md
│   ├── python-polars-expression-ranges/SKILL.md
│   ├── python-polars-expression-idioms/SKILL.md
│   ├── python-polars-reading-writing-data/SKILL.md
│   ├── python-polars-csv-io/SKILL.md
│   ├── python-polars-excel-io/SKILL.md
│   ├── python-polars-parquet-io/SKILL.md
│   ├── python-polars-json-io/SKILL.md
│   ├── python-polars-database-io/SKILL.md
│   ├── python-polars-multi-file-io/SKILL.md
│   ├── python-polars-encoding-io/SKILL.md
│   └── python-polars-write-formats/SKILL.md
│
├── Standalone Skills                            [package: python-core]
│   ├── python-architectural-fitness-functions/SKILL.md
│   ├── python-pattern-matching/SKILL.md
│   ├── python-restful-api-design/SKILL.md
│   ├── python-safe-io-file-parsing/SKILL.md
│   └── python-testing-mocking/SKILL.md
│
└── python-upgrade-dependencies/SKILL.md   Dependency upgrade workflow   [package: python-core]
```
