# Python — PostgreSQL (SQLAlchemy + Alembic)

Python-specific PostgreSQL usage: SQLAlchemy 2.0 async ORM and the Repository pattern, plus Alembic migrations (authoring, RLS enforcement, and rebasing local-only migration branches).

The Python ORM/migrations counterpart to the `database-postgresql` package's engine-level concepts. Use this instead of `python-mongodb` when the project's datastore is PostgreSQL (see the PostgreSQL-vs-MongoDB decision guide in the `python` skill in `python-core`).

## Install

```bash
npx skills add mihailtd/skills/skills/python-postgresql --all
```

Add `-g` to install globally instead of per-project. Use `--skill <name>` (repeatable) instead of `--all` to cherry-pick individual skills from this package, e.g.:

```bash
npx skills add mihailtd/skills/skills/python-postgresql --skill python-alembic-migration-rebase
```

## Skills (3)

- **[python-alembic-migration-rebase](python-alembic-migration-rebase/SKILL.md)** — Rebase and flatten Alembic database migrations that were created locally but never applied to production environments. Resolves branched migration history by merging heads and recreating migrations in a linear sequence. Use when you have multiple migration heads or migrations that need to be reorganized before production deployment.
- **[python-alembic-migrations](python-alembic-migrations/SKILL.md)** — Create and manage Alembic database migrations for PostgreSQL schema changes. Includes critical rules for immutability, idempotency, and Row Level Security (RLS) enforcement. Use when creating new migrations, modifying schema, or working with database changes.
- **[python-sqlalchemy-repository](python-sqlalchemy-repository/SKILL.md)** — Instructs the agent on building an asynchronous data layer using SQLAlchemy 2.0 syntax (Mapped, mapped_column), handling database relationships and query optimization, and strictly encapsulating all database access behind the Clean Architecture Repository pattern.
