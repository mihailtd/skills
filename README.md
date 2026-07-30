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
| [`python-core`](skills/python-core/README.md) | 25 | General Python, start with the `python` master skill: stack decisions (PostgreSQL vs MongoDB), functional-lite/dataclasses-over-OOP style, project setup, testing, logging, error handling, functional programming, async/concurrency (stdlib, so it lives here). Pair with a stack package below. |
| [`python-fastapi`](skills/python-fastapi/README.md) | 9 | REST APIs built on FastAPI: routing, DI, auth, caching, testing, OpenAPI. |
| [`python-polars`](skills/python-polars/README.md) | 36 | Data processing / ETL with Polars: expressions, I/O, aggregation, lazy API. |
| [`python-langgraph`](skills/python-langgraph/README.md) | 9 | LLM agent architectures with LangGraph: multi-agent design, state, memory, RAG. |
| [`python-postgresql`](skills/python-postgresql/README.md) | 3 | Python + PostgreSQL: SQLAlchemy 2.0 async ORM/Repository pattern, Alembic migrations. Driver/ORM layer — pair with `database-postgresql` for engine-level concepts. |
| [`python-mongodb`](skills/python-mongodb/README.md) | 2 | Python + MongoDB: Beanie async ODM — documents, CRUD, Link/BackLink relations, schema migrations. ODM layer — pair with `database-mongodb` for engine-level concepts. |
| [`database-postgresql`](skills/database-postgresql/README.md) | 48 | PostgreSQL *database* concepts: indexing, advanced SQL, partitioning, security, JSON/JSONB, performance. Engine-level, language-agnostic. |
| [`database-mongodb`](skills/database-mongodb/README.md) | 33 | MongoDB *database* concepts: aggregation, indexing, sharding, schema design, transactions. Engine-level, language-agnostic. |
| [`database-redis`](skills/database-redis/README.md) | 1 | Redis caching strategy. |
| [`database-general`](skills/database-general/README.md) | 1 | Cross-database fundamentals, engine-agnostic. |
| [`devops-pulumi`](skills/devops-pulumi/README.md) | 15 | Infrastructure-as-code with Pulumi: stacks, environments, secrets, CI/CD. |
| [`architecture`](skills/architecture/README.md) | 23 | Solution/enterprise architecture: diagramming, risk management, resilient design. *(cross-cutting)* |
| [`project-management`](skills/project-management/README.md) | 51 | Agile/lean delivery: story mapping, kanban, discovery, planning. *(cross-cutting)* |
| [`code-review`](skills/code-review/README.md) | 7 | Structured review checklists: architecture, structure, concurrency, security, integration-code quality/hygiene. *(cross-cutting)* |
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

## Creating a new skill

Scaffold a new skill with:

```bash
npx skills init my-skill
```

Every skill lives in its own directory containing a `SKILL.md` with valid YAML frontmatter (`name` matching the directory name, and `description`). Place it under the `skills/<package>/` directory that matches its stack, or start a new package folder if it doesn't fit an existing one — then add or update that package's `README.md`.

## License

MIT
