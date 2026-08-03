---
name: sql-antipatterns-31-flavors

description: Detects the 31 Flavors antipattern — baking a fixed set of allowed values into a column's metadata via ENUM or a CHECK constraint — which makes querying, adding, renaming, and retiring values require schema changes, and guides toward a lookup table with a foreign key instead.
---

# SQL Antipatterns — 31 Flavors

This skill helps developers recognize when a column's set of valid values
has been baked into the table's metadata (`ENUM`, `CHECK IN (...)`) instead
of stored as data, and guides them to a lookup table when that set isn't
truly fixed forever.

---

## 1. Recognize the Antipattern

- **The tell:** a column restricts its values via a `CHECK` constraint or a
  proprietary enumerated type, listing every allowed value directly in the
  column definition:

  ```sql
  CREATE TABLE Bugs (
      status VARCHAR(20) CHECK (status IN ('NEW', 'IN PROGRESS', 'FIXED'))
  );
  -- or, MySQL's ENUM:
  CREATE TABLE Bugs (
      status ENUM('NEW', 'IN PROGRESS', 'FIXED')
  );
  ```

  Same underlying shape with domains, user-defined types, or a trigger
  that hardcodes the permitted set.
- **Listen for these phrases**:
  - "We have to take the database offline to add a choice to a menu, it
    should take under thirty minutes." — a strong sign the set of values
    is metadata, not data.
  - "The status column can have one of the following values — we
    shouldn't need to revise this list." *Shouldn't need to* isn't *can't*
    — treat it as a near-certain future change request.
  - "The list of values in application code got out of sync with the
    database — again." — the recurring cost of maintaining the same set
    of values in two places (schema and app code) because the schema
    itself is hard to query for its own permitted values.
- Named after Baskin-Robbins' "31 Flavors" — a supposedly fixed lineup
  that grew and changed continuously over decades regardless of the
  original number baked into the brand.

---

## 2. Why It Breaks Down

- **You can't cheaply discover the permitted set from data.** `SELECT
  DISTINCT status FROM Bugs` only returns values currently *in use*, not
  every value that's *allowed* — if no bug is `FIXED` yet, it won't show
  up, creating a chicken-and-egg problem for anything (like a UI
  dropdown) that needs the full permitted set. Getting the real list means
  querying the database's own metadata catalog (e.g.
  `information_schema.columns`), which typically returns the whole `CHECK`
  or `ENUM` definition as one opaque string that application code then has
  to parse.
- **Adding or removing a value means altering the column**, not inserting
  a row:

  ```sql
  ALTER TABLE Bugs MODIFY COLUMN status
      ENUM('NEW', 'IN PROGRESS', 'FIXED', 'DUPLICATE');
  ```

  There's no "add one value" syntax — you redefine the entire set, which
  means you first have to know the current complete set (see above). Some
  databases can't restructure a populated column type in place at all,
  forcing an extract-transform-load cycle that takes the table offline.
- **This makes metadata changes routine instead of rare.** Schema changes
  should be infrequent, tested, deliberate events. When the business's
  changing list of options (like a company expanding into new countries,
  each with its own salutations) is stored as metadata, ordinary business
  changes turn into schema migrations under time pressure.
- **Retiring a value is destructive and ambiguous.** Removing a value from
  the enumeration doesn't tell the database what to do with existing rows
  that used it — you have to decide by hand whether to migrate them to a
  new value, null them out, or leave the old value in the definition
  purely so old rows remain valid (at which point your "clean" enum
  already has cruft in it).
- **Not portable.** `ENUM` is MySQL-specific; `CHECK`, domains, and UDTs
  vary in support and syntax across database brands. Any solution built on
  these ties the schema to a specific engine's feature set.

---

## 3. Legitimate Uses

- A fixed set of values genuinely works well with `ENUM`/`CHECK` when
  changing it would make no logical sense — a true mutually-exclusive
  either/or with no realistic third option: `LEFT`/`RIGHT`,
  `ACTIVE`/`INACTIVE`, `ON`/`OFF`, `INTERNAL`/`EXTERNAL`. Querying the
  metadata for the set of values is still awkward in this case, but at
  least you won't be fighting to keep an application-side copy in sync
  with a set that keeps changing.
- `CHECK` constraints are broadly useful for validation that has nothing
  to do with enumerating a fixed value set — e.g. asserting a time
  interval's start precedes its end. That's a different, unrelated use of
  the same SQL feature and isn't what this skill is about.
- Before choosing `ENUM`/`CHECK`, ask explicitly: is this set of values
  ever going to change, or even *might* it change? If there's real doubt,
  treat that doubt as "yes."

**PostgreSQL note:** Postgres's `ENUM` isn't a column modifier like
MySQL's — it's a real, separately-named type you create once and reuse
across columns/tables:

```sql
CREATE TYPE bug_status AS ENUM ('NEW', 'IN PROGRESS', 'FIXED');
CREATE TABLE Bugs (status bug_status);
```

Since PostgreSQL 9.1, you can add a value without recreating the type —
`ALTER TYPE bug_status ADD VALUE 'DUPLICATE';` — and as of PostgreSQL 12
this can even run inside a transaction alongside other DDL/DML, instead of
requiring its own separate, committed statement first. That closes off
the "adding a value forces an ETL cycle / takes the table offline"
problem described above — on modern Postgres, adding a value really is
close to a one-line, low-risk change.

What Postgres's `ENUM` still doesn't give you, so the core recommendation
in this skill still holds once the value set is genuinely fluid:

- **No removing or reordering values** — `ALTER TYPE ... DROP VALUE`
  doesn't exist; retiring a value has exactly the same "what happens to
  existing rows" ambiguity as MySQL's `ENUM`, with no built-in way to mark
  a value inactive versus fully gone.
- **No per-value metadata** — no way to attach an "active" flag, a display
  label, or an audit trail to individual values the way a lookup table's
  extra columns can (§4).
- **Still not portable** — it's a PostgreSQL-specific type, same portability
  cost as MySQL's `ENUM`, just with better ergonomics once you're
  committed to that engine.
- **Sort order follows declaration order, not alphabetical** — like
  MySQL's `ENUM`, comparisons and default sorting use the order values
  were declared in, which is often desired (e.g. a natural status
  progression) but can surprise anyone expecting lexical sort.

So on Postgres specifically, `ENUM` is a meaningfully more comfortable
choice for the "truly fixed, small set" case in this section — but a
value set that's genuinely expected to grow, shrink, or carry metadata
should still be a lookup table, not an `ENUM` type leaned on because
`ADD VALUE` made it convenient.

---

## 4. The Fix: Specify Values in Data

Create a lookup table holding one row per permitted value, and reference
it with a foreign key:

```sql
CREATE TABLE BugStatus (
    status VARCHAR(20) PRIMARY KEY
);
INSERT INTO BugStatus (status) VALUES ('NEW'), ('IN PROGRESS'), ('FIXED');

CREATE TABLE Bugs (
    status VARCHAR(20),
    FOREIGN KEY (status) REFERENCES BugStatus(status)
        ON UPDATE CASCADE
);
```

This is the same [[sql-antipatterns-keyless-entry]]-style constraint doing
the enforcement work `CHECK`/`ENUM` did, but the permitted set now lives in
data, not metadata:

- **Querying the full set is a plain `SELECT`**, sortable, filterable, no
  metadata parsing required:

  ```sql
  SELECT status FROM BugStatus ORDER BY status;
  ```

- **Adding a value is an `INSERT`**, with no downtime and no need to know
  the current set beforehand:

  ```sql
  INSERT INTO BugStatus (status) VALUES ('DUPLICATE');
  ```

- **Renaming a value is an `UPDATE`**, and with `ON UPDATE CASCADE` on the
  foreign key, every referencing row updates automatically:

  ```sql
  UPDATE BugStatus SET status = 'INVALID' WHERE status = 'BOGUS';
  ```

- **Retiring a value doesn't require deleting it** (the foreign key would
  block that anyway while rows still reference it). Add an `active` flag
  instead, so historical rows keep a valid reference while the UI only
  offers the current set:

  ```sql
  ALTER TABLE BugStatus ADD COLUMN active
      ENUM('INACTIVE', 'ACTIVE') NOT NULL DEFAULT 'ACTIVE';

  UPDATE BugStatus SET active = 'INACTIVE' WHERE status = 'DUPLICATE';

  SELECT status FROM BugStatus WHERE active = 'ACTIVE';
  ```

- **Portable** — this relies only on standard foreign key constraints, not
  a brand-specific type like `ENUM`, and scales to as many values as you
  need since each is just a row.

---

## 5. Review Checklist

- Is a column's permitted values list baked into `ENUM`/`CHECK`/a
  domain/UDT, when the business rule behind it could plausibly change
  (new categories, new regions, new statuses)?
- Does adding one new allowed value require an `ALTER TABLE`, possibly
  with downtime, rather than an `INSERT`?
- Is there a parallel list of the same values duplicated in application
  code, at risk of drifting out of sync with the schema?
- Is there any way to distinguish "obsolete but still referenced by
  historical rows" from "currently offerable" — or does removing a value
  destroy that distinction?
- If the set is genuinely a fixed, logically closed binary/small set
  (`ON`/`OFF`-shaped), `ENUM`/`CHECK` may still be the right, simpler
  choice — don't reach for a lookup table reflexively where nothing will
  ever be added.
