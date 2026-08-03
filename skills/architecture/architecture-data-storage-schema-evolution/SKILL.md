---
name: architecture-data-storage-schema-evolution

description: Instructs the AI assistant that a storage schema change's "breaking-ness" is defined by the specific storage technology and its access patterns, not a universal rule — covers the expand-contract pattern for changes too large for one deploy (additive field first, dual-write, backfill, cutover, only then remove the old field), designing fields for graceful absence and choosing extensible-vs-fixed value representations deliberately (native PostgreSQL ENUM's append-only, non-removable growth model vs. a TEXT+CHECK column for value sets expected to change), auditing every reader/writer of a field before migrating it, and keeping the storage schema and the API/response schema as separate models so a migration in progress never has to also be a breaking API change.
---

# Data Storage Schema Evolution Instructions

When supporting a change to a stored data shape — a database column, a
document field, a cache value's structure — where the system can't tolerate
downtime and old and new code must run against the same data concurrently
during the change, use this tool. It complements
**architecture-network-api-versioning**: that skill covers the API surface a
client talks to, this one covers the storage underneath it, and the two
timelines don't have to move together.

---

## Purpose

This tool helps the AI assistant by:

- establishing that whether a storage schema change is "breaking" depends on
  the specific storage technology's representation and the code that reads
  it — a rule true for one format or engine doesn't transfer unmodified to
  another, so this has to be reasoned through per system rather than assumed,
- providing a repeatable multi-step pattern (expand-contract) for storage
  changes too large or risky to make in a single deploy, so two adjacent
  versions of application code can run against the same underlying data
  without corrupting or losing anything,
- guiding field and value-set design toward graceful handling of the
  unexpected — fields that tolerate being absent, and a deliberate choice
  between a fixed, closed value set and one designed to grow,
- reinforcing that a migration plan is only as good as its knowledge of every
  caller touching the data — services, background jobs, analytics queries,
  admin tooling — and that an unaccounted-for caller is exactly how a
  "complete" migration still causes an incident,
- keeping the storage schema and the outward-facing API/response schema as
  separate models, so a storage migration in progress doesn't force a
  simultaneous breaking change to the API surface.

---

## Expected outcome

As the AI, your response should help teams adopt an approach that:

- reasons concretely about what "breaking" means for the storage technology
  actually in use, instead of importing a rule of thumb from a different
  format or a different codebase,
- breaks a storage change too risky for one deploy into an expand-contract
  sequence — add additively, dual-write, backfill, cut over reads, only then
  remove the old shape — with an explicit pause for confidence at each step,
- designs new fields to tolerate absence gracefully, and picks between a
  fixed/closed and an extensible value representation deliberately, based on
  whether the value set is genuinely closed or expected to grow,
- identifies every reader and writer of a field before starting a migration
  plan, rather than assuming the known callers are the only callers,
- keeps a storage model and its corresponding API/response model as distinct
  types with an explicit translation between them, so evolving one doesn't
  force evolving the other on the same schedule.

---

## Instructions for the AI

1. **Work out what "breaking" means for the specific storage technology in use**
   - There's no universal list of safe vs. unsafe storage changes — it
     depends on how the format represents data and what code touches it.
     A document store (MongoDB) or a JSON/JSONB column has no indirection
     between a field's name and its stored representation the way some
     binary serialization formats do — renaming a key is not something old
     stored documents magically follow; it's a genuinely different key that
     existing data doesn't have until migrated.
   - A relational column rename (`ALTER TABLE ... RENAME COLUMN`) is
     unambiguous for the data itself — there's no possibility of stale rows
     under the old name — but it immediately breaks any raw SQL, ORM
     mapping, or hand-written query still referencing the old name. The risk
     moves from the data to the code that addresses it.
   - Don't reason about a proposed change ("is renaming this field safe?")
     in the abstract — name the specific storage technology, work out
     concretely who or what breaks (stored data, application code, both, or
     neither), and design the change around that answer.

2. **Use the expand-contract pattern for anything too large for one deploy**
   - For a change where old and new application code must both work
     correctly against the data during a transition — which is the normal
     case for any system that can't take downtime — split it into stages
     instead of changing the shape and the code in one step:
     1. **Expand**: add the new field/column alongside the existing one,
        purely additive. Nothing reads it yet.
     2. **Dual-write**: deploy code that writes to both the old and new
        shape on every write, and reads from the new shape with a fallback
        to the old one if the new one isn't populated yet.
     3. **Backfill**: run a migration that populates the new shape for
        existing rows/documents that predate step 2.
     4. **Cut over reads**: once backfill is complete and dual-writing has
        run long enough to trust the data, deploy code that reads and
        writes only the new shape — but don't remove the old shape yet.
     5. **Contract**: after a safety window with no need to roll back,
        remove the old field/column in a dedicated cleanup migration.
   - Each stage exists specifically so that two adjacent deployed versions
     of the application can coexist against the same stored data without
     either one corrupting or losing information the other depends on.
     Collapsing stages to save time reintroduces exactly the risk the
     pattern exists to remove — don't shortcut it because a change looks
     small.
   - Map this onto the project's actual migration tooling rather than
     leaving it abstract: for PostgreSQL, this is a sequence of separate
     Alembic revisions, each paired with an application-code release (see
     **python-alembic-migrations**); for MongoDB via Beanie, the same shape
     via a sequence of `@iterative_migration` migrations paired with
     `Document` model changes released in lockstep (see
     **python-beanie-migrations**). Neither tool collapses these stages for
     you — the sequencing and the pause between stages is a deliberate
     decision the team makes, not something the migration framework
     enforces automatically.

3. **Design fields for graceful absence; choose value-set extensibility deliberately**
   - A document store or JSONB column doesn't automatically preserve
     "fields it doesn't recognize" the way some binary formats do — but a
     missing key is a completely normal, representable state. Favor
     optional/default-valued fields over ones that assume presence, so
     code written before a field existed doesn't need special-casing to
     keep working once it does.
   - For a genuinely fixed, small, essentially-never-changing value set —
     something like ISO currency codes you have no intention of ever
     touching — PostgreSQL's native `ENUM` type is a reasonable fit. But be
     precise about its cost: values can only be added
     (`ALTER TYPE ... ADD VALUE`), never removed or reordered, and adding a
     value has real operational restrictions (it can't be combined with
     other DDL in the same transaction on older PostgreSQL versions). For
     any value set expected to grow or change over the system's life —
     order statuses, categories, roles, feature flags — prefer a `TEXT`
     column with a `CHECK` constraint (or application/Pydantic-level
     validation) instead. Loosening or replacing a `CHECK` constraint is a
     straightforward migration; extending a native `ENUM` carries friction
     that compounds every time the set needs to grow again.
   - Apply the equivalent judgment in MongoDB: a plain string field with
     application-level (Pydantic) validation for an evolving value set,
     reserving a strict `$jsonSchema` enum constraint for the same kind of
     genuinely closed set that would justify a native `ENUM` in PostgreSQL.

4. **Audit every reader and writer before starting the migration**
   - An expand-contract plan is only as safe as its knowledge of every piece
     of code that touches the field being migrated — not just the obvious
     application service, but background jobs, scheduled tasks, analytics
     queries, admin scripts, and any other service sharing the same
     database. A caller left out of the plan can read stale data past the
     cutover point, or write to the old shape after dual-writing has
     stopped, silently reintroducing the exact inconsistency the migration
     was meant to prevent.
   - Treat this audit as a prerequisite step, not an afterthought — write
     down the plan and the full list of known callers before the first
     "expand" migration goes out, so gaps get caught by a reviewer instead
     of by an incident.

5. **Keep the storage schema and the API/response schema as separate models**
   - Don't return a SQLAlchemy model or a Beanie `Document` directly from a
     FastAPI route — define a distinct Pydantic response model as the
     translation boundary between the two. See
     **python-clean-architecture-fastapi-boundary** for the general shape
     of that boundary.
   - This separation is what makes the expand-contract sequence above
     possible without it also being a breaking change to the API: the
     storage shape can be mid-migration (old and new fields both present,
     one dual-written) while the API response stays completely stable,
     because the translation layer decides what the client sees regardless
     of what the storage layer currently looks like underneath. Without
     that separation, a storage migration and an API version bump end up
     forced onto the same timeline for no reason connected to the API's
     actual needs.

---

## AI decision guidance

When generating storage schema evolution guidance, keep these principles in
mind:

- **"Breaking" is defined by the specific storage technology and its
  callers** — reason about the actual system in question, not a rule
  imported from a different format or language.
- **A change too large for one deploy needs the full expand-contract
  sequence** — add, dual-write, backfill, cut over, then remove — with a
  genuine confidence pause between stages, never collapsed to save time.
- **Choose native-ENUM-style fixed value sets only for genuinely closed
  sets** — anything expected to grow belongs in a more easily extended
  representation (`TEXT` + `CHECK`, or application-level validation).
- **A migration plan is only as safe as its audit of every reader and
  writer** — an unaccounted-for caller is a realistic way for an otherwise
  correct plan to still cause an incident.
- **Storage and API schemas are separate concerns with separate models** —
  don't let a storage migration's timeline force a simultaneous API
  breaking change, or vice versa.

---

## Success criteria

A strong response should ensure that it:

- **names the specific storage technology** when reasoning about whether a
  change is breaking, rather than applying a generic rule,
- **uses the full expand-contract sequence** for any change too large for a
  single deploy, with an explicit pause for confidence between stages,
- **designs new fields to tolerate absence** and **chooses fixed vs.
  extensible value representations deliberately**, naming the tradeoff,
- **calls for an audit of every reader/writer** before a migration plan is
  considered final,
- **keeps the storage model and the API/response model separate**, so
  evolving one doesn't force evolving the other on the same schedule.

---

## Example prompts for the AI

- "Is it safe to rename this MongoDB document field?"
- "We need to change this column's meaning without downtime — how do we
  sequence that?"
- "Should this status field be a native PostgreSQL enum or just text with a
  check constraint?"
- "Can we just return our SQLAlchemy model straight from the API endpoint?"

---

## Related guidance

Use this tool alongside:

- python-alembic-migrations / python-beanie-migrations — the concrete
  migration tooling this skill's expand-contract sequence is implemented
  with for PostgreSQL and MongoDB respectively.
- architecture-network-api-versioning — the API-surface counterpart to this
  skill's storage-schema focus; point 5 above is why the two don't have to
  share a timeline.
- python-clean-architecture-fastapi-boundary — the general shape of the
  storage-model/API-model separation this skill's point 5 depends on.
- python-fastapi-partial-updates (package `python-fastapi`) — a concrete
  example of an API contract staying stable and safe (PATCH touching only
  explicitly-provided fields) independent of what's changing underneath in
  storage.
- architecture-idempotency-and-at-least-once-delivery — a backfill migration
  is itself an operation that may need to be retried safely; the same
  atomic-check-and-record discipline applies if a backfill script can be
  re-run.
- database-postgresql-json-jsonb-handling — relevant when the "storage
  schema" in question is a JSONB column rather than typed relational
  columns; the graceful-absence guidance in point 3 applies there directly.
