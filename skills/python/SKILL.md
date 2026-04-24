---
name: python
description: >
  General Python programming approach and standards for this project.
  Defines the core stack, tooling, style philosophy, and code conventions.
  USE THIS SKILL for any Python task. It references specialized domain skills
  for deeper guidance on specific topics (FastAPI, Alembic, async, functional
  programming, dependency management).
---

# Python Programming — General Approach

This is the master Python reference skill. It captures the non-negotiable defaults
and the "functional-lite" philosophy used across all Python code in this project.
Read this first; follow the linked domain skills for deeper guidance.

---

## Core Stack

| Concern | Choice |
|---|---|
| Package manager & task runner | `uv` |
| Packaging manifest | `pyproject.toml` only |
| Linter / formatter | `ruff` |
| Type checker | `ty` |
| Backend framework | FastAPI |
| Database | PostgreSQL |
| ORM | SQLAlchemy (async) |
| Migrations | Alembic |
| Runtime model | `asyncio` — always async |
| Programming style | Functional-lite |

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
See the **upgrade-dependencies** skill for the full upgrade workflow:
`.github/skills/upgrade-dependencies/SKILL.md`
For library-specific guidance, see `.github/skills/python-library-creation/SKILL.md`.

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
FastAPI domain skills:

- `.github/skills/python-fastapi-project-structuring/SKILL.md`
- `.github/skills/python-fastapi-routing-validation/SKILL.md`
- `.github/skills/python-fastapi-dependency-injection/SKILL.md`
- `.github/skills/python-fastapi-response-error-handling/SKILL.md`
- `.github/skills/python-fastapi-security-authentication/SKILL.md`
- `.github/skills/python-fastapi-security-attack-resistance/SKILL.md`
- `.github/skills/python-fastapi-api-testing/SKILL.md`
- `.github/skills/python-fastapi-openapi-documentation/SKILL.md`

---

## Database — PostgreSQL + SQLAlchemy (async)

PostgreSQL is the only supported database. SQLAlchemy is the ORM.

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

---

## Database Migrations — Alembic

Alembic manages all schema changes. Never mutate the database schema manually.

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

For full migration authoring rules and RLS enforcement see:
`.github/skills/python-alembic-migrations/SKILL.md`

For rebasing local-only migration branches see:
`.github/skills/python-alembic-migration-rebase/SKILL.md`

---

## Async — Always

Every I/O operation **must** be async. There is no place for blocking calls in
the request path.

- Use `asyncio.gather()` for concurrent independent coroutines.
- Use `asyncio.TaskGroup` (Python 3.11+) when structured concurrency and error
  propagation matter.
- Use `asyncio.Queue` or `asyncio.Semaphore` for back-pressure and rate limiting.
- Never call `time.sleep()` — use `await asyncio.sleep()`.
- Never use `requests` — use `httpx.AsyncClient` or `aiohttp`.

Deeper guidance:

- `.github/skills/python-asyncio/asyncio-concurrent-web-requests/SKILL.md`
- `.github/skills/python-concurrent-processing/SKILL.md` (CPU-bound work → `ProcessPoolExecutor` + asyncio integration)
- `.github/skills/python-asyncio/asyncio-synchronization/SKILL.md`

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
| Strict immutability everywhere | SQLAlchemy models and async state are inherently mutable — fight that only where it matters. |
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

Deeper guidance — consolidated and specialized functional skills:

- `.github/skills/python-higher-order-functools/SKILL.md` (map, filter, max/min, @cache, partial, reduce, @singledispatch)
- `.github/skills/python-lazy-evaluation-pipelines/SKILL.md` (yield, yield from, generator pipelines, map/filter/reduce)
- `.github/skills/python-immutable-data-design/SKILL.md` (NamedTuple, frozen dataclass, pyrsistent, context managers)
- `.github/skills/python-functional-programming/python-decorator-design/SKILL.md`
- `.github/skills/python-functional-programming/python-functional-web-services/SKILL.md` (FastAPI functional handlers)
- `.github/skills/python-functional-programming/python-toolz-composition/SKILL.md`
- `.github/skills/python-functional-programming/python-itertools-combinatorics/SKILL.md`
- `.github/skills/python-functional-programming/python-recursion-tco/SKILL.md`
- `.github/skills/python-functional-programming/python-pymonad-strict-evaluation/SKILL.md` ← opt-in only

---

## General Code Conventions

- **Python 3.11+** minimum. Use modern syntax: `match`, `TypeAlias`, `Self`, etc.
- **Type-annotate everything** — function signatures, variables, return types.
- **No `Any`** unless integrating with an untyped third-party boundary.
- Prefer `dataclass(frozen=True)` or Pydantic `BaseModel` for value objects.
- Keep functions small and single-purpose. If a function needs a comment to
  explain *what* it does (not *why*), split it.
- Avoid global mutable state. Pass dependencies explicitly (constructor or `Depends()`).
- Use `logging` — not `print`. Configure at app startup; never call
  `basicConfig()` inside library code.
  See `.github/skills/python-configuration-observability/SKILL.md` for full
  logging, tracing, and configuration guidance.

---

## Skill Map

```
python/SKILL.md  ← you are here (master reference)
│
├── Backend Framework
│   ├── python-fastapi-project-structuring/SKILL.md
│   ├── python-fastapi-routing-validation/SKILL.md
│   ├── python-fastapi-dependency-injection/SKILL.md
│   ├── python-fastapi-response-error-handling/SKILL.md
│   ├── python-fastapi-security-authentication/SKILL.md
│   ├── python-fastapi-security-attack-resistance/SKILL.md
│   ├── python-fastapi-api-testing/SKILL.md
│   └── python-fastapi-openapi-documentation/SKILL.md
│
├── Database & Migrations
│   ├── python-alembic-migrations/SKILL.md
│   └── python-alembic-migration-rebase/SKILL.md
│
├── Async Patterns
│   ├── python-asyncio/asyncio-concurrent-web-requests/SKILL.md
│   └── python-asyncio/asyncio-synchronization/SKILL.md
│
├── Consolidated Core Skills
│   ├── python-configuration-observability/SKILL.md   (config + logging + tracing)
│   ├── python-data-structures-type-system/SKILL.md   (collections + types + Pydantic)
│   ├── python-lazy-evaluation-pipelines/SKILL.md     (generators + map/filter/reduce)
│   ├── python-higher-order-functools/SKILL.md        (HOFs + cache + partial + singledispatch)
│   ├── python-immutable-data-design/SKILL.md         (NamedTuple + frozen DC + context mgrs)
│   └── python-concurrent-processing/SKILL.md         (ProcessPool + asyncio integration)
│
├── Functional Programming (specialized)
│   ├── python-functional-programming/python-decorator-design/SKILL.md
│   ├── python-functional-programming/python-functional-web-services/SKILL.md
│   ├── python-functional-programming/python-toolz-composition/SKILL.md
│   ├── python-functional-programming/python-itertools-combinatorics/SKILL.md
│   ├── python-functional-programming/python-recursion-tco/SKILL.md
│   └── python-functional-programming/python-pymonad-strict-evaluation/SKILL.md ← opt-in
│
├── Library Creation
│   └── python-library-creation/SKILL.md
│
├── Standalone Skills
│   ├── python-architectural-fitness-functions/SKILL.md
│   ├── python-pattern-matching/SKILL.md
│   ├── python-restful-api-design/SKILL.md
│   ├── python-safe-io-file-parsing/SKILL.md
│   └── python-testing-mocking/SKILL.md
│
└── upgrade-dependencies/SKILL.md   Dependency upgrade workflow
```
