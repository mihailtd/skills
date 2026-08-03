---
name: sql-antipatterns-entity-attribute-value

description: Detects the Entity-Attribute-Value (EAV) antipattern — a generic entity/attribute-name/attribute-value table used to fake a flexible schema — which loses data types, mandatory columns, referential integrity, and simple queries, and guides toward Single/Concrete/Class Table Inheritance or a JSON column instead.
---

# SQL Antipatterns — Entity-Attribute-Value

This skill helps developers recognize when a generic `(entity, attr_name,
attr_value)` table is being used to simulate a flexible schema for related
subtypes, and guides them to a design that keeps SQL's actual guarantees —
data types, `NOT NULL`, foreign keys, simple queries — while still supporting
subtype-specific attributes.

---

## 1. Recognize the Antipattern

- **The tell:** instead of columns, attributes are stored as *rows* in a
  generic table keyed by entity ID and attribute name:

  ```sql
  CREATE TABLE IssueAttributes (
      issue_id   BIGINT UNSIGNED NOT NULL,
      attr_name  VARCHAR(100) NOT NULL,
      attr_value VARCHAR(100),
      PRIMARY KEY (issue_id, attr_name),
      FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
  );
  ```

  Also called "open schema," "schemaless," or "name-value pairs" when
  reimplemented on top of a relational database (as opposed to genuinely
  using a document/key-value store for it).
- **Listen for these phrases**:
  - "This database is totally extensible without metadata changes — you
    can define new attributes at runtime." Relational databases don't
    actually offer that; claiming it usually means EAV is underneath.
  - "What's the maximum number of joins I can do in a query?" — a sign a
    query is joining the attribute table once per attribute to reconstruct
    a single logical row.
  - "I can't figure out how to write a report for this platform, we need a
    consultant." Common for turnkey systems built on EAV — routine
    reporting becomes genuinely hard.

---

## 2. Why It Breaks Down

- **Simple queries turn into string matching.** Fetching one attribute for
  all rows means filtering the generic table by `attr_name`:

  ```sql
  SELECT issue_id, attr_value AS date_reported
  FROM IssueAttributes
  WHERE attr_name = 'date_reported';
  ```

  instead of `SELECT issue_id, date_reported FROM Issues`.
- **No mandatory attributes.** `NOT NULL` applies to a column; EAV has no
  columns per attribute, so nothing stops an entity from simply never
  getting a row for a required attribute. Enforcing "every issue has a
  `date_reported`" moves entirely into application code, checked on every
  read, with no principled way to backfill what's missing.
- **No real data types.** `attr_value` is typically one wide string column
  shared by every attribute, so `'banana'` is just as valid a
  `date_reported` as an actual date. Splitting `attr_value` into
  per-type columns (`attr_value_date`, `attr_value_integer`, ...) recovers
  typing but makes every read a `COALESCE` across all of them.
- **No referential integrity on individual attributes.** A `FOREIGN KEY`
  on `attr_value` would apply to *every* attribute stored in the table,
  not just the one (e.g. `status`) that should reference a lookup table —
  there's no way to constrain only one row-shape's meaning of that column.
- **No control over attribute names.** Nothing stops `date_reported` and
  `report_date` from coexisting as two names for the same fact, or an
  entity from having neither, or both. Queries then have to account for
  every naming variant that ever slipped in:

  ```sql
  SELECT date_reported, COUNT(*) AS bugs_per_date
  FROM (SELECT DISTINCT issue_id, attr_value AS date_reported
        FROM IssueAttributes
        WHERE attr_name IN ('date_reported', 'report_date'))
  GROUP BY date_reported;
  ```

- **Reconstructing one logical row costs one join or `CASE` per
  attribute**, and that cost grows every time a new attribute is added —
  the query has to be rewritten to know about it in advance:

  ```sql
  SELECT issue_id,
      MAX(CASE attr_name WHEN 'date_reported' THEN attr_value END) AS date_reported,
      MAX(CASE attr_name WHEN 'status'         THEN attr_value END) AS status,
      MAX(CASE attr_name WHEN 'priority'       THEN attr_value END) AS priority
  FROM IssueAttributes
  WHERE issue_id = 1234
  GROUP BY issue_id;
  ```

- **This is a specific case of the "inner-platform effect":** building a
  system flexible enough to redefine its own schema at runtime is, in
  effect, reimplementing the database inside the database — with a
  fraction of the engineering years and none of the query optimizer,
  type system, or constraint enforcement the real one has.

---

## 3. Legitimate Uses

- If exactly one (or a couple) of tables in an otherwise conventional
  schema genuinely need schemaless attributes, and the rest of the model
  doesn't, isolating the EAV pattern to just that table can be the lesser
  evil versus distorting the entire schema. Budget for the extra
  application-side work this costs — experienced teams report EAV designs
  become unwieldy within about a year, so revisit the decision, don't
  treat it as permanent.
- If the actual need is genuinely non-relational (fully dynamic, per-row
  schema with no bounded set of subtypes), consider a real non-relational
  store built for it — a document database (MongoDB), key-value store
  (Redis, DynamoDB, Berkeley DB), or search engine (Elasticsearch) —
  rather than reimplementing one on top of SQL. Note this doesn't remove
  the underlying tension: fluid metadata always costs simple queries and
  discovery overhead, whichever storage engine hosts it.
- If you're stuck maintaining an existing EAV-based system (inherited
  project, acquired third-party platform), see the post-processing
  approach below rather than fighting the model.

---

## 4. The Fix: Model the Subtypes

If there's a finite, known set of subtypes with knowable attributes —
almost always true even when a design feels like it needs EAV — model
them directly. Pick per relationship based on query pattern, not
universally:

### a. Single Table Inheritance

One table, one column per attribute across *all* subtypes, plus a
discriminator column. Subtype-specific columns are `NULL` on rows where
they don't apply.

```sql
CREATE TABLE Issues (
    issue_id   SERIAL PRIMARY KEY,
    issue_type VARCHAR(10),      -- 'BUG' or 'FEATURE'
    priority   VARCHAR(20),
    severity   VARCHAR(20),      -- only for bugs
    sponsor    VARCHAR(50)       -- only for feature requests
);
```

Simplest option, works well with single-table ORM access patterns
(Active Record). Weakest when subtypes or subtype-specific attributes are
numerous — the table gets sparse and wide, and nothing in the schema
documents which columns belong to which subtype; that tracking is manual.

### b. Concrete Table Inheritance

One full table per subtype, each repeating the shared base attributes.

```sql
CREATE TABLE Bugs (
    issue_id SERIAL PRIMARY KEY,
    priority VARCHAR(20),
    severity VARCHAR(20)
);
CREATE TABLE FeatureRequests (
    issue_id SERIAL PRIMARY KEY,
    priority VARCHAR(20),
    sponsor  VARCHAR(50)
);
```

The database rejects attributes that don't belong to a subtype for free
(there's no column for them). Costs: shared attributes must be kept in
sync by hand across every subtype table if they change, nothing in the
metadata documents the tables are related, and querying across all
subtypes needs a `UNION ALL` view. Best when you rarely need to query
across subtypes at once.

### c. Class Table Inheritance

A base table for shared attributes, plus one table per subtype whose
primary key is also a foreign key back to the base table.

```sql
CREATE TABLE Issues (
    issue_id SERIAL PRIMARY KEY,
    priority VARCHAR(20)
);
CREATE TABLE Bugs (
    issue_id BIGINT UNSIGNED PRIMARY KEY,
    severity VARCHAR(20),
    FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
);
CREATE TABLE FeatureRequests (
    issue_id BIGINT UNSIGNED PRIMARY KEY,
    sponsor  VARCHAR(50),
    FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
);
```

Efficient for querying/searching against shared base attributes across all
subtypes, then joining out to the relevant subtype table only for matches.
Downsides: no constraint can enforce that an entity appears in exactly one
subtype table (it could appear in none or several), and it's the most
tables to maintain of the three relational options. Best when queries
across all subtypes on shared attributes are common.

### d. Semistructured Data (JSON/XML column)

When the number of subtypes or attributes is genuinely unbounded, keep
the shared attributes as real columns and push only the dynamic part into
a `JSON` column:

```sql
CREATE TABLE Issues (
    issue_id   SERIAL PRIMARY KEY,
    issue_type VARCHAR(10),
    priority   VARCHAR(20),
    attributes JSON NOT NULL   -- dynamic, subtype-specific attributes
);
```

This is the most flexible option — every row can carry its own attribute
set — but querying into the JSON payload is more awkward and less
efficient than plain columns, even with a database's built-in JSON
functions. Use it when true per-row extensibility is required, not as a
default.

**PostgreSQL note:** this is the one legitimate-use case where PostgreSQL
specifically changes the calculus, because `JSONB` (as opposed to plain
`JSON`, or the `JSON`/`TEXT` blobs most other engines offer) is a genuinely
strong semistructured-data column, not just an escape-hatch blob:

```sql
CREATE TABLE Issues (
    issue_id   SERIAL PRIMARY KEY,
    issue_type VARCHAR(10),
    priority   VARCHAR(20),
    attributes JSONB NOT NULL
);

-- indexed, efficient containment/existence queries
CREATE INDEX issues_attributes_gin ON Issues USING GIN (attributes);
SELECT * FROM Issues WHERE attributes @> '{"severity": "loss of functionality"}';
SELECT * FROM Issues WHERE attributes ? 'sponsor';

-- jsonpath queries (PostgreSQL 12+) for deeper/conditional lookups
SELECT * FROM Issues WHERE attributes @@ '$.severity == "critical"';
```

`JSONB` stores a decomposed binary form (not just text), supports a `GIN`
index over the whole document so containment/key-existence queries
(`@>`, `?`, `jsonpath` via `@@`/`jsonb_path_query`) run efficiently instead
of scanning and re-parsing JSON text per row, and lets you mix real
columns for the stable, shared attributes with `attributes JSONB` for the
genuinely dynamic part — which is exactly the "isolate EAV to the one
place that truly needs it" legitimate use from §3, just implemented as one
well-indexed column instead of a second sprawling table. This still
doesn't grant a `FOREIGN KEY`, `NOT NULL`, or a real data type on anything
inside the JSONB payload — those constraints remain the reason to prefer
Single/Concrete/Class Table Inheritance whenever the set of attributes is
actually knowable in advance. Reach for `JSONB` when it genuinely isn't,
not as a default because it's convenient.

---

## 5. If You're Stuck with EAV

When migrating away isn't an option (inherited system, third-party
platform), don't fight the model by trying to reconstruct a single
conventional row per entity in SQL. Query it the way it's actually shaped
— one row per attribute — and assemble the object in application code:

```sql
SELECT issue_id, attr_name, attr_value
FROM IssueAttributes
WHERE issue_id = 1234;
```

```python
issue = {}
for (field, value) in cursor:
    issue[field] = value
```

This is more application code than a conventional schema would need, but
it's the direct consequence of having given up SQL's own way of managing
metadata — expect to write in code what the schema would otherwise have
enforced.

---

## 6. Review Checklist

- Is there a generic `(entity_id, attr_name, attr_value)` table standing
  in for what should be typed columns on a conventional table?
- Are "required" attributes only enforced by application code checking
  for the presence of a row, instead of `NOT NULL`?
- Does any column store values for multiple logical attributes with
  different real data types, relying on string comparison or `COALESCE`
  across type-specific shadow columns?
- Can the same fact be stored under more than one attribute name because
  nothing constrains `attr_name` to a fixed vocabulary?
- Does reconstructing one logical entity require a join or `CASE` branch
  per attribute, with the query needing to be edited every time an
  attribute is added?
- If the answer to "how many subtypes/attributes are there" is actually
  bounded and knowable, prefer Single/Concrete/Class Table Inheritance
  over EAV or a JSON blob.

---

## Related guidance

PostgreSQL-specific remedy:

- database-postgresql-json-jsonb-handling
