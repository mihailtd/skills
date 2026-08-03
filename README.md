# skills

[![Install via skills.sh](https://img.shields.io/badge/skills.sh-install-green)](https://skills.sh/mihailtd/skills)

A personal collection of [Agent Skills](https://agentskills.io/) for AI coding agents (GitHub Copilot, Claude Code, Cursor, Codex, and more), organized into stack-specific **packages** so you can install only what your project needs instead of all 278 skills at once.

## Installation

Each package lives under `skills/<package>/` and can be installed on its own by pointing `npx skills add` at that subpath:

```bash
npx skills add mihailtd/skills/skills/<package> --all
```

You'll be prompted for which agent(s) to install to. Add `-g` to install globally (user-level) instead of per-project — sensible for the cross-cutting packages (`architecture`, `project-management`, `code-review`, `writing`, `business-analysis`) that aren't tied to one codebase.

Omit `--all` to pick interactively, or use `--skill <name>` (repeatable) to install specific skills from a package without the rest:

```bash
npx skills add mihailtd/skills/skills/python-fastapi --skill python-fastapi-routing-validation --skill python-fastapi-security-authentication
```

To install everything from every package in one go (the old flat behavior):

```bash
npx skills add mihailtd/skills --all -g
```

## Packages

| Package | Skills | What it covers |
|---|---|---|
| [`python-core`](skills/python-core/README.md) | 33 | General Python, start with the `python` master skill: stack decisions (PostgreSQL vs MongoDB, Polars vs pandas, Polars/DuckDB vs Spark), functional-lite/dataclasses-over-OOP style, project setup, testing, logging, error handling, date/time modeling/requirements-clarity/testability, functional programming, async/concurrency (stdlib, so it lives here), lazy-vs-eager startup initialization, dark-mode/light-mode rollout verification, notebooks (marimo, not Jupyter). Pair with a stack package below. |
| [`python-fastapi`](skills/python-fastapi/README.md) | 10 | REST APIs built on FastAPI: routing, DI, auth, caching, testing, OpenAPI, PATCH-style partial updates. |
| [`python-polars`](skills/python-polars/README.md) | 36 | Data processing / ETL with Polars: expressions, I/O, aggregation, lazy API. |
| [`python-langgraph`](skills/python-langgraph/README.md) | 13 | LLM agent architectures with LangGraph: multi-agent design, state, memory, RAG, the workflow-vs-agent design/suitability decisions that precede implementation, and context engineering (Generation/Retrieval/Write/Reduce/Isolate). |
| [`python-fastmcp`](skills/python-fastmcp/README.md) | 14 | MCP (Model Context Protocol) server development with FastMCP: tools (`@mcp.tool()`), resource providers (`@mcp.resource()`), prompt providers (`@mcp.prompt()`), security & discovery, multi-agent orchestration, RAG, LangGraph integration, enterprise EKM, personalization, multimodal AI, evaluation/benchmarking, performance tuning, manual testing (MCP Inspector), and client integration. |
| [`python-clean-architecture`](skills/python-clean-architecture/README.md) | 25 | Clean/Onion Architecture, SOLID, and DDD in Python, functional-lite style — no class hierarchies: Functional Core/Imperative Shell, the Dependency Rule, dependency inversion, SRP/OCP/ISP/LSP, domain modeling, aggregates, factories, use cases, controllers/presenters, the composition root, drivers, testing, legacy migration, and the FastAPI/Pydantic boundary. Growing — more skills to follow. |
| [`python-postgresql`](skills/python-postgresql/README.md) | 3 | Python + PostgreSQL: SQLAlchemy 2.0 async ORM/Repository pattern, Alembic migrations. Driver/ORM layer — pair with `database-postgresql` for engine-level concepts. |
| [`python-mongodb`](skills/python-mongodb/README.md) | 2 | Python + MongoDB: Beanie async ODM — documents, CRUD, Link/BackLink relations, schema migrations. ODM layer — pair with `database-mongodb` for engine-level concepts. |
| [`database-postgresql`](skills/database-postgresql/README.md) | 50 | PostgreSQL *database* concepts: indexing, advanced SQL, partitioning, security, JSON/JSONB, performance, temporal modeling, idempotency keys. Engine-level, language-agnostic. |
| [`database-mongodb`](skills/database-mongodb/README.md) | 34 | MongoDB *database* concepts: aggregation, indexing, sharding, schema design, transactions, temporal modeling. Engine-level, language-agnostic. |
| [`database-redis`](skills/database-redis/README.md) | 39 | Redis *database* concepts: data modeling, RediSearch indexing, JSON, time series, probabilistic structures, geospatial, messaging/Streams, VSS, programmability, caching, rate limiting, sessions, leaderboards, fraud detection, microservices patterns. Covers plain Redis and Redis Stack. |
| [`database-duckdb`](skills/database-duckdb/README.md) | 20 | DuckDB *database* concepts: in-process OLAP engine, the CLI, DDL/CRUD/joins/aggregation/views/analytics patterns, geospatial queries, the relational (method-chaining) API, columnar/vectorized performance design, SQL against files/DataFrames without a load step, CSV/Parquet/Excel/JSON import-export, remote HTTP(S)/Hugging Face dataset access, pulling from a live PostgreSQL server, the full DuckDB-vs/plus-Polars-and-PostgreSQL decision guide (including Polars' own SQLContext), and DuckDB-in-marimo notebooks. Python client examples. |
| [`database-general`](skills/database-general/README.md) | 1 | Cross-database fundamentals, engine-agnostic. |
| [`sql-antipatterns`](skills/sql-antipatterns/README.md) | 31 | Relational schema/query design mistakes from *SQL Antipatterns*, one skill per antipattern — engine-agnostic. *(cross-cutting)* |
| [`devops-pulumi`](skills/devops-pulumi/README.md) | 15 | Infrastructure-as-code with Pulumi: stacks, environments, secrets, CI/CD. |
| [`architecture`](skills/architecture/README.md) | 65 | Solution/enterprise architecture: diagramming, risk management, resilient design, platform strategy, standards adoption, change management, simplicity, incremental delivery, change proposals/process, decomposition/composition, parallelism, code-duplication/inheritance-coupling/exception-design/flexibility-complexity/configuration-surface/third-party-library/idempotency/delivery-semantics/versioning/network-API-versioning/data-storage-schema-evolution/trend-adoption tradeoffs (Python + JS/TS), refactoring (scope classification, degradation diagnosis, decision criteria, evidence gathering, plan structure, verified-behavior rollout), backlog/review practices, estimation. *(cross-cutting)* |
| [`project-management`](skills/project-management/README.md) | 51 | Agile/lean delivery: story mapping, kanban, discovery, planning. *(cross-cutting)* |
| [`code-review`](skills/code-review/README.md) | 14 | Structured review checklists: architecture, structure, concurrency, security, integration-code quality/hygiene, cognitive-science-backed naming, code smells/linguistic antipatterns, and Clean Architecture conformance (boundaries + functional-lite style). *(cross-cutting)* |
| [`business-analysis`](skills/business-analysis/README.md) | 4 | Domain storytelling and scope definition. *(cross-cutting)* |
| [`writing`](skills/writing/README.md) | 11 | Verb-driven prose editing. *(cross-cutting)* |

Each package folder has its own `README.md` with the full skill list and description.

**Database packages come in pairs**: `database-<engine>` covers the database's own concepts (indexing, sharding, security — true regardless of what language talks to it), while `python-<engine>` covers how *Python specifically* talks to it (ORM/ODM, migrations, async driver patterns). Install both halves for the datastore your project actually uses.

## Recipes by project type

**FastAPI + PostgreSQL service**
```bash
npx skills add mihailtd/skills/skills/python-core --all
npx skills add mihailtd/skills/skills/python-fastapi --all
npx skills add mihailtd/skills/skills/python-postgresql --all
npx skills add mihailtd/skills/skills/database-postgresql --all
```

**FastAPI + MongoDB service**
```bash
npx skills add mihailtd/skills/skills/python-core --all
npx skills add mihailtd/skills/skills/python-fastapi --all
npx skills add mihailtd/skills/skills/python-mongodb --all
npx skills add mihailtd/skills/skills/database-mongodb --all
```

**Polars data pipeline**
```bash
npx skills add mihailtd/skills/skills/python-core --all
npx skills add mihailtd/skills/skills/python-polars --all
```

**Polars + DuckDB analytics pipeline** (SQL-first exploration/reporting over files, Polars for pipeline code)
```bash
npx skills add mihailtd/skills/skills/python-core --all
npx skills add mihailtd/skills/skills/python-polars --all
npx skills add mihailtd/skills/skills/database-duckdb --all
```

**LLM agent service (LangGraph + MCP tools)**
```bash
npx skills add mihailtd/skills/skills/python-core --all
npx skills add mihailtd/skills/skills/python-langgraph --all
npx skills add mihailtd/skills/skills/python-fastmcp --all
```

**Async worker / background-job service (Celery-style)**
```bash
npx skills add mihailtd/skills/skills/python-core --all
```
`python-core` includes the asyncio/concurrency skills (asyncio is standard library, not a separate stack choice) — no Celery-specific skill exists yet, this is the closest match.

**Just the cross-cutting skills, installed once for all your projects**
```bash
npx skills add mihailtd/skills/skills/architecture -g --all
npx skills add mihailtd/skills/skills/project-management -g --all
npx skills add mihailtd/skills/skills/code-review -g --all
```

## Agents

The [`agents/`](agents/) directory holds Claude Code subagent definitions (`.agent.md`) that consume this library's skills to perform a specific, scoped audit or review task. Each stays out of the others' territory deliberately:

- **[`code-reviewer`](agents/code-reviewer.agent.md)** — line-level code review (structure, data structures, architecture, concurrency/performance, security, integration hygiene, naming, Clean Architecture conformance) using the `code-review` package.
- **[`project-auditor`](agents/project-auditor.agent.md)** — high-level house-standards audit (`uv`/`ruff`/`ty`/pre-commit, functional-lite style, async-by-default, approved stack, migration hygiene). Not a code review.
- **[`clean-architecture-auditor`](agents/clean-architecture-auditor.agent.md)** — Clean Architecture conformance audit specifically (layer structure, the Dependency Rule, functional-lite reformulation of entities/use cases/drivers) using the `python-clean-architecture` package. Not a general code review, not the house-standards audit.
- **[`dependency-upgrader`](agents/dependency-upgrader.agent.md)** — dependency upgrade workflow using `python-upgrade-dependencies`.
- **[`refactoring-auditor`](agents/refactoring-auditor.agent.md)** — assesses whether, what, why, and when to refactor: measures the repo's current state (git-churn hotspots, test safety net), identifies patterns/antipatterns and technique/technology use vs. misuse across this library's skills, and drafts a prioritized refactoring plan. Not a code review, not the house-standards audit, not a Clean Architecture conformance check. A living skeleton, extended incrementally as *Refactoring at Scale* book content is fed in.

## Creating a new skill

Scaffold a new skill with:

```bash
npx skills init my-skill
```

Every skill lives in its own directory containing a `SKILL.md` with valid YAML frontmatter (`name` matching the directory name, and `description`). Place it under the `skills/<package>/` directory that matches its stack, or start a new package folder if it doesn't fit an existing one — then add or update that package's `README.md`.

## License

MIT
