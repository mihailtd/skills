---
name: python-beanie-migrations
description: Instructs the agent on writing and running Beanie's built-in schema migrations for MongoDB — iterative_migration and free_fall_migration decorators, the beanie CLI, and forward/backward migration file structure. Use when a Document's shape changes and existing data needs to be backfilled or transformed.
---

# Python Beanie Migrations Guidelines

You are an expert Python developer specializing in evolving MongoDB document shapes safely with Beanie's built-in migration framework. When a `Document` model's fields change and existing data must be updated to match, use this skill instead of ad-hoc one-off scripts.

## 1. MongoDB migrations are not Alembic — but Beanie ships a real equivalent

MongoDB itself is schemaless and has no DDL migration system. But Beanie (specifically, not raw PyMongo) provides its own migration framework — do not assume "MongoDB has no migrations" and fall back to writing untracked one-off scripts if the project uses Beanie. Use it the same way Alembic is mandatory for the SQLAlchemy path (see the **python-alembic-migrations** skill for the relational-side equivalent).

## 2. Creating a migration

Generate a new migration file with the `beanie` CLI. Files are timestamp-prefixed so they always execute in creation order.

```bash
beanie new-migration -n add_active_flag -p migrations/
```

This creates `migrations/YYYYMMDDHHMMSS_add_active_flag.py`. Files beginning with an underscore are skipped by the runner — use that prefix for shared helpers, never for a migration you want applied.

Every migration file defines two classes: `Forward` for applying the change, `Backward` for reverting it. Never ship a migration with an empty or incorrect `Backward` — treat rollback as seriously as forward migration.

## 3. `@iterative_migration()` — the default choice

Use this for the overwhelming majority of migrations: field renames, type changes, default backfills, dropping a field. It streams every document in the collection through your function one at a time — memory-safe for large collections, no need to hand-write pagination.

- Define an `input_document` model (the old shape) and `output_document` model (the new shape) as typed parameters; Beanie handles fetching and writing each document.
- Only fields you assign on `output_document` are changed — everything else round-trips unmodified.

```python
from beanie import Document, iterative_migration

class ProductOld(Document):
    class Settings:
        name = "products"
    name: str
    price: float

class ProductNew(Document):
    class Settings:
        name = "products"
    name: str
    price: float
    active: bool = True

class Forward:
    @iterative_migration()
    async def backfill_active(self, input_document: ProductOld, output_document: ProductNew):
        output_document.active = True

class Backward:
    @iterative_migration()
    async def remove_active(self, input_document: ProductNew, output_document: ProductOld):
        pass  # dropping a field needs no explicit assignment on the old shape
```

## 4. `@free_fall_migration()` — for anything iterative can't express

Reach for this only when the change doesn't fit a per-document transform — restructuring across collections, conditional logic that needs custom queries, or bulk operations. It hands you a raw `session` instead of an automatic per-document loop; pass that `session` into any `Document` write calls so the migration stays transactional and rollback-safe.

```python
from beanie import free_fall_migration

class Forward:
    @free_fall_migration(document_models=[ProductOld, ProductNew])
    async def restructure(self, session):
        async for doc in ProductOld.find_all(session=session):
            new_doc = ProductNew(name=doc.name, price=doc.price, active=True)
            await new_doc.replace(session=session)
```

Prefer `@iterative_migration()` whenever the change can be expressed as "map old document to new document" — only drop to `free_fall_migration` when it genuinely can't.

## 5. Running migrations

```bash
# Apply all pending migrations, forward, in a transaction
beanie migrate -uri "mongodb+srv://user:pass@host" -db app_db -p migrations/

# Apply exactly one migration
beanie migrate -uri "..." -db app_db -p migrations/ --distance 1

# Roll back
beanie migrate -uri "..." -db app_db -p migrations/ --backward
```

Migrations run inside a transaction by default, which requires a replica set (the standard MongoDB Atlas/production topology). Only pass `--no-use-transaction` against a standalone, non-replica-set instance — never in an environment where transactional safety matters, since a failed migration can then leave data partially transformed.

## 6. Rules

- Never mutate a `Document` model's fields for an existing collection without a corresponding migration — the model and the data must move together.
- Run migrations in CI/CD the same way Alembic migrations are run for the relational path: as an explicit deploy step, never implicitly on app startup.
- Keep migrations idempotent where possible — running `beanie migrate` twice against an already-migrated database should be a no-op, not an error.
- Treat applied migrations as immutable, exactly like Alembic revisions: to fix a mistake, write a new migration, don't edit a merged one.

## 7. Related guidance

- **architecture-data-storage-schema-evolution** (package `architecture`) — for a field rename or shape change too risky to make in one migration/deploy without downtime, sequence it as expand (add the new field) → dual-write → backfill (this skill's `@iterative_migration`) → cutover → contract (drop the old field in a later migration), each paired with its own app-code release, rather than collapsing it into one `Forward`/`Backward` pair.
