# Python — MongoDB (Beanie)

Python-specific MongoDB usage: Beanie (the async ODM built on Pydantic + PyMongo's async driver) — document modeling, `init_beanie`, CRUD, querying, Link/BackLink relations, and Beanie's built-in schema migration framework.

The Python driver/ODM counterpart to the `database-mongodb` package's engine-level concepts. Use this instead of `python-postgresql` when the project's datastore is MongoDB (see the PostgreSQL-vs-MongoDB decision guide in the `python` skill in `python-core`).

## Install

```bash
npx skills add mihailtd/skills/skills/python-mongodb --all
```

Add `-g` to install globally instead of per-project. Use `--skill <name>` (repeatable) instead of `--all` to cherry-pick individual skills from this package, e.g.:

```bash
npx skills add mihailtd/skills/skills/python-mongodb --skill python-beanie-documents
```

## Skills (2)

- **[python-beanie-documents](python-beanie-documents/SKILL.md)** — Instructs the agent on modeling MongoDB collections as Beanie Documents (async ODM built on Pydantic and PyMongo's async driver), initialization with init_beanie, indexes, CRUD operations, querying, and Link/BackLink relations between documents.
- **[python-beanie-migrations](python-beanie-migrations/SKILL.md)** — Instructs the agent on writing and running Beanie's built-in schema migrations for MongoDB — iterative_migration and free_fall_migration decorators, the beanie CLI, and forward/backward migration file structure. Use when a Document's shape changes and existing data needs to be backfilled or transformed.
