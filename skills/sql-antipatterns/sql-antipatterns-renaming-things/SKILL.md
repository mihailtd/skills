---
name: sql-antipatterns-renaming-things

description: Explains why renaming a live table or column is a chicken-and-egg deployment problem — the database change and the code change can't land atomically across a rolling deployment — and gives two zero-downtime techniques (dual-write-then-migrate, or a compatibility view aliasing the old name) for renaming without an outage.
---

# SQL Antipatterns — Renaming Things Safely

This skill helps plan a table or column rename against a live database
without downtime, by recognizing why a naive "just rename it" deploy
breaks, and applying one of two safe sequencing techniques.

---

## 1. Recognize the Problem

- **The tell:** a rename looks trivial — `ALTER TABLE ... RENAME` or `ALTER
  TABLE ... RENAME COLUMN` — but the application code and the database
  schema can't change atomically together in a live, rolling deployment.
- Whichever order you do it in, there's a window of breakage:
  - Rename the table first, and the still-running old application code
    errors immediately — it's still querying the old name.
  - Deploy new code first, and it immediately errors — the table hasn't
    been renamed yet.
  - Deployments to multiple servers roll out independently, so at any
    given moment some servers may be running old code and others new code
    against the *same* database — there's no instant where "old code +
    old name" and "new code + new name" aren't both live somewhere at
    once.
- The only case with no problem is being able to take the application
  fully offline for the rename — rare for anything expected to run with
  minimal downtime.

---

## 2. Why It's Worth the Trouble (When It Is)

Renaming is meaningfully harder and riskier than purely additive schema
changes (new table, new column, new index) — some teams conclude it's
not worth doing except for a real, non-cosmetic reason:

- The existing name is offensive or creates legal exposure.
- The name refers to a discontinued technology, retired partner, or
  corporate brand no longer in use (e.g. after an acquisition).
- The name now conflicts with a trademark the business isn't authorized to
  use.
- The name is confusingly close to a different name in active use
  elsewhere in the same company/project.

Personal preference or style isn't a strong enough reason to take on this
risk — reserve renames for cases with real, durable justification.

---

## 3. The Fix: Two Zero-Downtime Techniques

### a. Dual-write, then migrate

Move data to a new table/column gradually, keeping both old and new
consistent throughout, so each deployment step is independently safe:

1. Create the new table/column under the new name.
2. Deploy code that writes to *both* old and new on every write, but still
   *reads* from the old one (the old one is the only one guaranteed fully
   up to date at this point).
3. Backfill: copy existing data from old to new.
4. Deploy code that reads from the new table/column, while still writing
   to both.
5. Deploy code that stops writing to the old one.
6. Drop the old table/column.

Every step is independently deployable and reversible without an instant
where any running version of the code is querying something that doesn't
exist yet. This costs more engineering time than a single rename
statement, but it's what makes each step safe to roll out gradually across
a fleet of servers.

### b. Compatibility view aliasing the old name

Faster to execute, when the database supports updatable views for your
query patterns:

1. Rename the table/column to the new name directly.
2. Immediately (as close to atomically as possible) create a view under
   the *old* name that selects from the new one, standing in as an alias.
3. The application keeps working against the old name via the view —
   including `INSERT`/`UPDATE` in databases where the view supports
   that — while you migrate application code to the new name at a
   comfortable pace, not under deployment-timing pressure.
4. Once every caller has moved to the new name, drop the compatibility
   view.

Test carefully before relying on this — not every query pattern that works
against a base table behaves identically against a view (especially for
writes), so verify the specific queries your application issues actually
work through the view before cutting over.

---

## 4. Review Checklist

- Is a table/column rename being planned as a single atomic
  statement, without accounting for a rolling deployment where old and
  new application code run concurrently against the same database?
- Is the rename justified by a real, durable reason (legal, branding,
  naming conflict) rather than stylistic preference — given the extra
  engineering cost either safe technique requires?
- If using dual-write-then-migrate: is there a concrete plan (and
  monitoring) for each step, especially the backfill and the final
  cutover/cleanup steps, so the migration doesn't stall half-finished?
- If using a compatibility view: have the actual queries the application
  issues against the old name been tested against the view, including any
  writes?
