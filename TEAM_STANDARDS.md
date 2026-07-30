# Engineering Standards — Python Projects

This document summarizes the team's Python engineering standards: tooling, stack
choices, database conventions, and coding style. It's derived from this skills
repository.

This doc is meant to be read standalone — you don't need the skills installed
to use it. Where a topic goes deeper than fits here, it names the skill/package
to install (see [Where to Go Deeper](#where-to-go-deeper)).

## At a glance

| Concern                       | Choice                                                                         |
| ----------------------------- | ------------------------------------------------------------------------------ |
| Package manager & task runner | `uv`                                                                           |
| Packaging manifest            | `pyproject.toml` only                                                          |
| Linter / formatter            | `ruff`                                                                         |
| Type checker                  | `ty`                                                                           |
| Data processing library       | `polars` (not `pandas`)                                                        |
| Notebooks                     | `marimo` (not Jupyter)                                                         |
| Backend framework             | FastAPI                                                                        |
| Data validation & config      | Pydantic (v2) — models for data/API shapes, `pydantic-settings` for env config |
| Async job queues              | Celery + Redis                                                                 |
| Database                      | PostgreSQL or MongoDB — see [decision guide](#databases-postgresql-vs-mongodb) |
| Relational path               | SQLAlchemy (async) + Alembic                                                   |
| Document path                 | Beanie (default) or PyMongo's async driver                                     |
| Runtime model                 | `asyncio` — always async unless there's a specific reason not to be            |
| Programming style             | Functional-lite — data classes + functions, not OOP                            |

---

## Architecture

### Project archetypes

The team builds three kinds of projects, and each has its own conventions
elsewhere in this document:

- **HTTP REST APIs** — FastAPI (see below, and the [Databases](#databases-postgresql-vs-mongodb) section).
- **AI MCP servers** — Model Context Protocol servers exposing tools/resources
  to AI agents. There's no dedicated skill for this yet in the skills
  repository — treat the FastAPI/async/functional-lite conventions elsewhere
  in this doc as the baseline until one exists.
- **Background/workflow task processing** — Celery workers consuming from
  queues. See [Background task processing](#background-task-processing-celery) below.

### Databases: PostgreSQL vs. MongoDB

Pick **one datastore path per project** (or per bounded context in a larger
system) — don't mix ORMs/ODMs within the same service.

**Use PostgreSQL + SQLAlchemy + Alembic when:**

- Data is naturally relational — entities reference each other and you need real joins.
- You need transactions spanning multiple entities/tables with strong consistency.
- The schema is relatively stable and you want it enforced and versioned.
- You'll run reporting/analytics SQL directly against the primary store.

**Use MongoDB when:**

- Data is naturally document-shaped — nested, variable, or maps closely to your API's JSON payloads.
- The schema evolves frequently per-document and you don't want a migration gate on every change.
- You want application-transparent horizontal sharding built into the database itself, without adopting a separate scaling layer.
- Cross-entity transactions are rare-to-none in your access patterns.

PostgreSQL scales to very high write throughput and large datasets too — via
partitioning, read replicas, and (if needed) horizontal-scaling extensions
like Citus. The distinguishing factor for MongoDB here isn't that Postgres
can't scale, it's whether you want that sharding behavior built into the
database by default versus layered on deliberately.

**Decision heuristic** — if the task is:

| Situation                                                    | Choice                                                    |
| -------------------------------------------------------------- | ------------------------------------------------------------ |
| Strict relationships, referential integrity                  | PostgreSQL                                                |
| Nested/variable JSON-like records (events, logs, catalogs)   | MongoDB                                                   |
| Multi-table transactions or complex reporting SQL             | PostgreSQL                                                |
| Fast document-shape iteration without a migration per field  | MongoDB                                                   |
| Uncertain / mixed requirements                                | Default to PostgreSQL — the safer general-purpose choice |

#### Relational path: PostgreSQL + SQLAlchemy (async)

- Always use the async engine and session (`AsyncEngine`, `AsyncSession`) — never call sync SQLAlchemy APIs from async code.
- Use `select()`/`insert()`/`update()`/`delete()` — not raw SQL strings.
- Define models with `DeclarativeBase` and `Mapped[T]` typed columns (SQLAlchemy 2.x style).

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

#### Document path: MongoDB

Choose the driver layer within MongoDB:

- **Beanie** (default) — Pydantic-model ergonomics; `Document` subclasses feel like SQLAlchemy declarative models, with validation and indexes colocated. Prefer this unless you have a specific reason not to.
- **PyMongo's async driver** — thin, direct control for aggregation-heavy or bulk work where Beanie's abstraction gets in the way.

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

### Automated database migrations — mandatory, not optional

**If a project has a database, it must have automated, versioned migrations.**
An app running against PostgreSQL or MongoDB with no migration mechanism is a
standards violation — schema/document shape changes happening without a
versioned, auditable path is a real production risk (drift between
environments, no rollback story, no history of _why_ a field changed).

**PostgreSQL → Alembic.**

```bash
uv run alembic revision --autogenerate -m "add_items_table"
uv run alembic upgrade head
```

Migrations are immutable once applied outside local dev, and should be
idempotent where possible. Never mutate schema by hand.

**MongoDB (Beanie) → Beanie's own migration framework.** This is a common
misconception worth correcting explicitly: "MongoDB is schemaless" does **not**
mean "MongoDB needs no migrations." If you're on Beanie, it ships a real
migration system — `@iterative_migration()` for per-document transforms
(the common case), `@free_fall_migration()` for anything that doesn't fit a
per-document shape, and a `beanie migrate` CLI with forward/backward support.
Use it the same way Alembic is mandatory on the relational side.

```bash
beanie new-migration -n add_active_flag -p migrations/
beanie migrate -uri "mongodb+srv://..." -db app_db -p migrations/
```

If you're on raw PyMongo without Beanie, there genuinely is no off-the-shelf
equivalent — version document shape changes with explicit migration scripts
and/or a `schema_version` field on documents, rather than untracked one-off edits.

### Should you use an ORM/ODM?

**This is a recommendation, not a hard requirement** — unlike migrations above.

Adopt SQLAlchemy+Alembic or Beanie (matching your database) when the codebase
shows signs it would benefit:

- Query logic is scattered and duplicated across multiple route handlers/modules.
- The schema/document shape is non-trivial and growing.
- You want repository-pattern encapsulation between business logic and the data layer.

It's fine to skip the ORM/ODM when:

- The project is small and simple, with a handful of well-contained query call-sites.
- Raw queries are already consistent and easy to follow.

Don't force an ORM onto a small service just because it's the default for
larger ones — but if you do use one, keep DB access behind a repository layer
rather than scattering ORM calls through route handlers/business logic.

### Background task processing (Celery)

Celery is the approved way to run background/workflow work: anything that
shouldn't block an HTTP response, runs on a schedule, or is naturally a
multi-step pipeline. This section is about the _philosophy_ behind that
architecture, not just the tool.

#### The core concept: an asynchronous task queue

A task queue decouples **asking for work to happen** from **doing the work**.
A producer — an API handler, another task, a scheduled job — doesn't execute
the task itself. It serializes a message (which task, with what arguments)
onto a named queue. That message sits in the queue (Redis, in our stack) until
a worker picks it up. The producer moves on immediately; it doesn't wait for
the work to finish.

**Workers watch specific queues, not everything.** Each worker process is
configured to consume from one queue (or a small, deliberate set of related
queues) — never "all queues by default." A worker built for image processing
watches the image-processing queue; a worker built for sending notifications
watches the notifications queue. They don't see, and can't accidentally pick
up, each other's tasks. This is what makes independent scaling possible: the
image-processing workload and the notification workload never compete with
each other for capacity, because they're not competing for the same queue.

**Within one queue, replicas _do_ compete — and that's the point.** If you run
five replicas of the same worker (five pods in Kubernetes, watching the same
queue), each replica independently connects to the broker and pulls the next
available task. The broker hands any given task to exactly one consumer —
never to two workers at once, and never broadcast to all of them. This
competing-consumers pattern is the entire scaling mechanism: to increase
throughput on a queue, add more replicas of the worker that watches it. No
code changes, no coordination logic to write — the broker handles it.

So the two mechanics together give you real independent scaling:

- **Different worker types → different queues → independently sized.** The
  video-transcoding worker can run 10 replicas while the email worker runs 2,
  because they're entirely separate deployments with separate autoscaling
  policies (e.g. scaled on that specific queue's backlog/depth), each in its
  own container.
- **Same worker type → same queue → horizontally scaled as one pool.**
  Replicas of one worker type share the load on their queue automatically;
  scaling that workload is just changing a replica count.

This is exactly why each worker gets its own container and its own
`Dockerfile.<worker-name>`, per [Each app owns its own Dockerfile](#each-app-owns-its-own-dockerfile)
above — a worker watching one queue is architecturally its own independently
deployable unit, with its own resource profile and its own scaling policy. It
should never share an image, a scaling policy, or a queue with a worker doing
unrelated work.

#### Task decomposition — think in workflows, not scripts

The most common mistake is writing one large task function that does
everything end-to-end — download a file, parse it, transform it, validate it,
write it to the database, send a notification, update a search index — all in
a single task body. Don't do this. It has real, compounding costs:

- **Retries redo everything.** If step 6 of 7 fails, retrying the task re-runs
  steps 1–5 too, including anything expensive or already-applied.
- **You lose observability.** You can only see "the task failed," not which
  step failed or how long each step actually took.
- **Nothing is reusable.** A "resize an image" step buried inside one giant
  task can't be reused by a different workflow that also needs to resize images.
- **You can't scale a bottleneck independently.** If the "transform" step is
  the slow part, you can't give it more workers without also scaling the
  cheap steps around it.

Instead, **decompose the work into small, single-purpose tasks and compose
them into an explicit workflow** using Celery's canvas primitives:

- **`chain`** — a sequential pipeline, where each task's output feeds the next.
- **`group`** — independent tasks run in parallel (fan-out).
- **`chord`** — a `group` followed by a callback that runs once every parallel
  task in it has completed (fan-out, then aggregate).

Before writing a task, think about the workflow it belongs to: what are the
discrete steps, which of them can run in parallel, which of them might need to
retry independently, and which of them might be reused elsewhere. Design that
shape explicitly rather than writing a linear script and calling it a task.

---

## Stack

Only the following are approved. Anything else in a dependency list is a
deviation worth a conversation, not an automatic rewrite.

| Category            | Approved                                | Notes                                                                                                |
| ------------------- | --------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| REST HTTP API       | **FastAPI**                             | Async route handlers, `Depends()` for DI, typed Pydantic response models — never raw dicts.          |
| Async job queues    | **Celery**, broker/backend on **Redis** | Redis is approved as a Celery broker/cache — not as a project's primary datastore.                   |
| Relational database | **PostgreSQL**                          | Via SQLAlchemy (async) + Alembic — see below.                                                        |
| Document database   | **MongoDB**                             | Via Beanie (default) or PyMongo's async driver — see below.                                          |
| Dataframes          | **Polars**, not pandas                  | `import pandas` anywhere in application code is a standards violation.                               |
| Notebooks           | **marimo**, not Jupyter                 | Reactive, git-diffable, no hidden execution-order state. Any `.ipynb` file in a repo is a violation. |

**Known gaps**: there is no dedicated Celery skill or marimo skill in the skill
library yet — the async/concurrency skills are the closest guidance for Celery
task code today. If your team leans heavily on either, authoring those skills
is a reasonable next step.

---

## Code organization

### The toolchain is `uv` + `ruff` + `ty` — nothing else

One manifest file (`pyproject.toml`), one package manager, one linter/formatter,
one type checker. This isn't a preference, it's a constraint: **never** use
`pip`, `pip-tools`, `poetry`, `pipenv`, `conda`, `venv`, or `pyenv` directly.
No `requirements.txt`/`requirements-dev.txt` in new projects.

```bash
uv init my-project        # scaffold a new project
uv sync                   # install dependencies
uv add <package>          # add a runtime dependency
uv add --dev <package>    # add a dev dependency
uv run <command>          # run anything inside the project's environment
uv lock --upgrade         # upgrade dependencies
```

All production dependencies are pinned exactly (`==`) for reproducible builds.

### Runtime vs. dev dependencies — keep them separate

Every dependency is either something the application needs to _run_, or
something the team needs to _develop_ it (linters, type checkers, test
frameworks, pre-commit itself). Getting this split wrong has real cost:

- A dependency added as a plain runtime dependency when it's actually
  dev-only (e.g. `pytest`, `ruff`, `ty`) gets installed into every production
  environment — bigger images, a larger attack surface, and slower deploys
  for tooling nobody runs in prod.
- Conversely, a genuine runtime dependency added as `--dev` will work on a
  developer's machine and then break in production, where dev dependencies
  aren't installed.

Add runtime dependencies with `uv add <package>`; add anything the app itself
doesn't need to run with `uv add --dev <package>` — this keeps them in a
separate `[dependency-groups] dev = [...]` block in `pyproject.toml`, so a
production install (`uv sync --no-dev`, or however your deploy pipeline is
configured) never pulls in tooling it doesn't need. When in doubt — "does this
package get imported by the running application, or only used to build/test/lint
it?" — answer that before deciding which flag to use.

### Recommended layout

```text
my-project/
├── .python-version
├── README.md
├── pyproject.toml
├── uv.lock
├── src/
│   └── my_project/
│       ├── __init__.py
│       └── main.py
└── tests/
```

### `pyproject.toml` is the single manifest

There is exactly one configuration file per project. Beyond the standard
`[project]` metadata and `[build-system]`, `pyproject.toml` also holds every
tool's settings as its own section — `[tool.ruff]` (and `[tool.ruff.lint]`,
`[tool.ruff.format]`) for the linter/formatter, `[tool.ty.*]` for the type
checker, `[tool.pytest.ini_options]` for test discovery, and
`[dependency-groups]` to separate runtime from dev dependencies (see below).
Nothing gets its own separate config file (`ruff.toml`, `pytest.ini`, etc.) —
if a tool supports living inside `pyproject.toml`, it lives there.

### Monorepo layout

For a repo hosting several independently deployable apps, use a `uv` workspace
instead of one flat project: a root `pyproject.toml` declares the workspace,
each app gets its own subdirectory under `apps/` with its own `pyproject.toml`,
and the whole workspace shares one `uv.lock`.

```text
project/
├── pyproject.toml        # workspace root: [tool.uv.workspace], shared dev deps
├── uv.lock                # one shared lock file for every app
├── apps/
│   ├── api/
│   │   ├── pyproject.toml
│   │   ├── Dockerfile.api
│   │   └── src/
│   └── worker/
│       ├── pyproject.toml
│       ├── Dockerfile.worker
│       └── src/
└── tests/
```

Because every app shares one lock file, they **must** agree on the version of
any dependency they have in common — two apps pinning different versions of
the same package will fail to resolve. Check every workspace member before
bumping a shared dependency, and update all of them together.

### Pre-commit: auto-fix, then block

Every repo must enforce lint/format/type-check **before** a commit lands — not
just in CI after the fact. The chain has two stages, and both are required:

1. **Auto-fix first.** Formatting and auto-fixable lint issues are corrected
   automatically as part of the commit hook, so nobody has to manually chase
   trivial style issues.
2. **Then block.** After auto-fix runs, a second, blocking check re-verifies
   formatting, linting, and type-checking. If anything still fails — an issue
   auto-fix couldn't resolve, or a type error — the commit is rejected.

A hook chain that only formats and never blocks doesn't satisfy this
standard — the blocking step is what actually prevents bad code from entering
history, not just tidies it on the way in. This can be wired up either through
the `pre-commit` framework or a plain Git hook script; the enforcement is what
matters, not the specific mechanism.

---

## Deployment

### Each app owns its own Dockerfile

Every app under `apps/` is independently deployable, so every app gets its own
Dockerfile — named `Dockerfile.<app-name>` (e.g. `Dockerfile.api`,
`Dockerfile.worker`) rather than a single shared `Dockerfile` at the repo root.
This keeps each app's image minimal (it only installs what that app actually
needs) and lets apps be built, versioned, and deployed independently of each
other. Only fall back to a single shared Dockerfile if there's a specific
reason multiple apps must ship as one image — that should be the exception,
not the default.

### Three environments

Every project has exactly three environments, each with a distinct purpose
and a distinct risk tolerance — don't treat them as interchangeable, and don't
apply production rigor to dev or dev looseness to production.

- **`dev`** — the continuous-integration target. Changes are pushed and
  integrated here continuously, including experiments. **Dev can sometimes be
  broken** — that's an acceptable cost of it being the environment where
  things get tried. Don't be afraid to push early, incomplete work here.
- **`staging`** — user acceptance testing (UAT), QA, and integration testing
  against other systems' lower environments (their dev/staging, not their
  prod). Staging should be stable enough to actually test against — it's not
  the place for half-finished experiments the way dev is.
- **`prod`** — rigorous and tightly protected. Deployments require approval
  (change management), review, and thorough testing beforehand. This should
  not feel like an obstacle: **if the process was actually followed** —
  changes went through dev, then staging, with review and testing at each
  step — production approval is a formality confirming that, not a new gate
  you're meeting for the first time.

### Local development

- Set up with `uv sync` and install the pre-commit hooks (`uv run pre-commit install`)
  before writing code — catching lint/format/type issues locally is faster
  than catching them in CI or in review.
- Run the relevant test suite locally before pushing, not just at PR time.
- Local `.env` files (or equivalent) hold environment-specific config and
  secrets and are never committed — see [Code hygiene](#code-hygiene) above
  for what belongs in an environment variable in the first place.

### Pull requests & merges

- Every change to a shared branch goes through a PR — no direct pushes to
  `main`/`dev`. A PR must pass CI (the same `ruff format` / `ruff check` /
  `ty check` / tests that pre-commit runs locally — CI is the backstop for
  anyone whose local hooks were skipped or bypassed) and get at least one
  review before merging.
- Merging to the main integration branch is what deploys to **dev** — this is
  the continuous-integration part of the flow, and it's expected to happen
  often.
- Promotion to **staging** and **prod** is a deliberate, separate step, not an
  automatic side effect of every merge — a project's release process decides
  when a dev-validated set of changes is promoted onward, and production
  promotion is where the approval/change-management step in the section above
  applies.

---

## Coding standards

### Functional-lite methodology

Apply functional techniques **where they reduce complexity and improve
clarity** — this is not strict FP purism.

**What we do:** pure functions as the default for business logic, immutable
data (frozen dataclasses / Pydantic models), higher-order functions (`map`,
`filter`, `functools.reduce`) over imperative loops for data pipelines,
generators for large/streaming data, `functools` utilities (`lru_cache`,
`partial`, `wraps`) used freely, small composable functions chained into
pipelines, cross-cutting concerns (auth, caching, retries) as decorators.

**What we don't force:** monads (`Maybe`/`Either`/`Result` — use exceptions +
typed returns instead), point-free style, strict immutability where a
framework (SQLAlchemy, Beanie) is inherently mutable, third-party FP libraries
imposed on shared code without team agreement.

#### Data classes + functions, not OOP classes

The default _shape_ of business logic is **a `@dataclass` (or Pydantic model)
plus free functions that operate on it** — not a class with instance methods
and mutable `self` state.

```python
# ✅ Preferred
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
```

**Real OOP classes are still the right tool** where the framework requires
them or you're modeling genuine encapsulated state — not business logic:
SQLAlchemy `DeclarativeBase` models, Beanie `Document`s, Pydantic
`BaseModel`/FastAPI dependency-provider classes, custom `Exception` types, or
objects that truly own mutable state across calls (a connection pool, a
stateful parser).

### Async by default

Every I/O operation must be async — there's no place for blocking calls in a
request or task path, regardless of which database path you're on.

- `asyncio.gather()` for concurrent independent coroutines; `asyncio.TaskGroup` (3.11+) when structured concurrency/error propagation matters.
- `asyncio.Queue`/`asyncio.Semaphore` for back-pressure and rate limiting.
- Never `time.sleep()` — `await asyncio.sleep()`.
- Never `requests` — `httpx.AsyncClient` or `aiohttp`.

**The exception**: genuinely CPU-bound work (not I/O) is a legitimate reason to
not be async inline — but it should be offloaded via `ProcessPoolExecutor` and
integrated back into the event loop with `run_in_executor`, not just left as a
blocking call in request-handling code.

### Code hygiene

Small conventions that consistently prevent real bugs — cheap to follow,
expensive to clean up after the fact.

#### Naming — explicit, complete, no shortcuts

Naming conventions are **mandatory, not a style preference**.

- **No cryptic abbreviations or shortcuts.** Variable and parameter names must
  be explicit and complete — spell out the domain term. `checkResult`, not
  `chkRes`; `userConfiguration`, not `usrCfg`; `databaseConnection`, not
  `dbConn`. Optimize for the next reader, not for typing speed. (A trivial,
  fully-local loop variable in a short comprehension is the one reasonable
  exception — anything that lives beyond a couple of lines gets a real name.)
- **Function names must be verbs.** A function *does* something, so its name
  should say what: `calculate_total`, `send_notification`, `validate_order` —
  not `total`, `notification`, or `order_validator`. If you can't put a verb
  on a function's name, that's a signal it's either scoped wrong or should be
  a value/property instead of a function call.

See the `code-review-quality-and-hygiene` skill for the full guidance and more examples.

#### Integration code: safe URL construction

- **Build URLs with `urllib.parse`, never string concatenation.** Use
  `urljoin()` to combine a base URL with a path, and `urlencode()` for query
  parameters. Manual concatenation (`base_url + "/path"`) reliably produces
  missing slashes, double slashes, or broken escaping.
- **One `BASE_URL` per integration, and only the base.** Scheme + host (+ port)
  only — no path segments, no query string baked in. If switching environments
  means updating several related URL variables at once instead of one, that's
  the smell: consolidate to a single base and compose endpoints from it.

#### Configuration: Pydantic Settings & environment variables

Load and validate configuration through `pydantic-settings`'
`BaseSettings` — not scattered `os.environ.get(...)` calls. A `Settings` class
gives you one documented, type-checked, IDE-discoverable place that declares
every config value a service needs, with validation and defaults built in:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class ApiSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="API_", env_file=".env")

    database_url: str
    log_level: str = "INFO"
```

**Environment variable vs. hardcoded value** — a value belongs in `Settings`
(backed by an environment variable) if any of these are true: it differs per
environment (dev/staging/prod), it needs to change without a redeploy, or it's
a secret (credentials, tokens, sensitive identifiers). If none apply,
hardcoding it is fine — don't reach for an environment variable reflexively
where a constant would do.

**In a monorepo, prefixing environment variables is mandatory** — with one
deliberate exception. Every app's `Settings` class sets its own `env_prefix`
(`API_`, `WORKER_`, etc., as above). Without this, two apps that happen to
declare a same-named variable (`DATABASE_URL`, `TIMEOUT_SECONDS`, `LOG_LEVEL`)
will either silently share a value neither app intended to share, or one
app's config change will unintentionally affect another app's runtime.

- **Prefix per-app** when two apps could legitimately need *different* values
  for a variable with the same conceptual name — e.g. `API_DATABASE_URL` vs.
  `WORKER_DATABASE_URL` if they point at different databases or connection
  pools.
- **Leave it unprefixed** only when the variable is genuinely shared by
  design — every app must always see the exact same value. Typical examples:
  a `REDIS_URL` every app uses to reach the same broker, or `ENVIRONMENT`
  indicating which of dev/staging/prod is running.
- **The test**: would two apps in this monorepo ever legitimately need a
  *different* value for this variable? If yes, prefix it. If no — the value
  must always be identical across every app by design — leave it unprefixed;
  prefixing it would just create multiple copies of "the same" value that can
  silently drift out of sync.

---

## Where to go deeper

Every topic above is backed by a skill in this skills repository. Install what
your project needs — see the repo README for the full package catalog and
per-project-type install recipes.

| Topic                               | Package               | Key skills                                                                                                                             |
| ----------------------------------- | --------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Everything above, in one place      | `python-core`         | `python` (master reference — start here)                                                                                               |
| Project setup / tooling detail      | `python-core`         | `python-project-setup`                                                                                                                 |
| FastAPI conventions                 | `python-fastapi`      | routing, DI, security, testing, OpenAPI                                                                                                |
| PostgreSQL + SQLAlchemy/Alembic     | `python-postgresql`   | `python-sqlalchemy-repository`, `python-alembic-migrations`, `python-alembic-migration-rebase`                                         |
| MongoDB + Beanie                    | `python-mongodb`      | `python-beanie-documents`, `python-beanie-migrations`                                                                                  |
| PostgreSQL engine internals         | `database-postgresql` | indexing, partitioning, security, JSON/JSONB, performance (48 skills)                                                                  |
| MongoDB engine internals            | `database-mongodb`    | aggregation, indexing, sharding, schema design, transactions (33 skills)                                                               |
| Redis                               | `database-redis`      | caching strategy                                                                                                                       |
| Polars                              | `python-polars`       | expressions, I/O, aggregation, lazy API (36 skills)                                                                                    |
| Functional-lite / dataclasses       | `python-core`         | `python-immutable-data-design`, `python-higher-order-functools`, `python-lazy-evaluation-pipelines`, `python-functional-programming-*` |
| Async/concurrency                   | `python-core`         | `python-asyncio-concurrent-web-requests`, `python-asyncio-synchronization`, `python-concurrent-processing`                             |
| Background task processing (Celery) | —                     | No dedicated skill yet — the async/concurrency skills above are the closest available guidance.                                        |
| Dependency upgrades                 | `python-core`         | `python-upgrade-dependencies`                                                                                                          |

```bash
# Install everything a FastAPI + PostgreSQL project needs
npx skills add <skills-repo>/skills/python-core --all
npx skills add <skills-repo>/skills/python-fastapi --all
npx skills add <skills-repo>/skills/python-postgresql --all
npx skills add <skills-repo>/skills/database-postgresql --all
```

Replace `<skills-repo>` with wherever this skills repository lives (a GitHub
path, or an internal Git host).
