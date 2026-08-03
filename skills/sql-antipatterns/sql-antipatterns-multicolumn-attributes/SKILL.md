---
name: sql-antipatterns-multicolumn-attributes

description: Detects the Multicolumn Attributes antipattern — storing a multivalue attribute (like tags) as several similarly-named columns (tag1, tag2, tag3) on one table — which makes search, add/remove, uniqueness, and growth all awkward, and guides toward a dependent table with one row per value instead.
---

# SQL Antipatterns — Multicolumn Attributes

This skill helps developers recognize when a multivalue attribute has been
spread across several numbered columns on one table (`tag1`, `tag2`,
`tag3`, ...) instead of a dependent table, and guides them back to one row
per value.

---

## 1. Recognize the Antipattern

- **The tell:** a table has several similarly-named columns holding the
  same *kind* of value, left `NULL` when unused:

  ```sql
  CREATE TABLE Bugs (
      bug_id      SERIAL PRIMARY KEY,
      description VARCHAR(1000),
      tag1        VARCHAR(20),
      tag2        VARCHAR(20),
      tag3        VARCHAR(20)
  );
  ```

- This is a sibling of [[sql-antipatterns-jaywalking]] — both exist to
  solve the same underlying problem, storing an attribute that can have
  multiple values. Jaywalking usually shows up for many-to-many
  relationships (packed into one delimited column); Multicolumn Attributes
  usually shows up for one-to-many relationships (spread across several
  columns) — but either shape can appear for either kind of relationship,
  so don't assume the mapping is strict.
- **Listen for these phrases**:
  - "How many is the greatest number of tags we need to support?" —
    the same "unknowable ceiling" question Jaywalking provokes, just
    applied to a column count instead of a string length.
  - "How can I search multiple columns at the same time in SQL?" — a
    direct sign that several columns are standing in for one logical,
    repeatable attribute.

---

## 2. Why It Breaks Down

- **Searching means checking every column, every time.** Finding rows
  with one tag value requires an `OR` (or an `IN (tag1, tag2, tag3)`) across
  every column, and it only gets more tedious per additional search term:

  ```sql
  SELECT * FROM Bugs
  WHERE 'performance' IN (tag1, tag2, tag3)
  AND   'printing'    IN (tag1, tag2, tag3);
  ```

- **Adding a value safely needs a read before the write.** You must find
  which column is free before you can `UPDATE` it, which is a race
  condition under concurrent writers — two clients can both read "tag2 is
  free" and step on each other. Avoiding the read means resorting to
  verbose, repetitive `CASE`/`COALESCE` expressions that repeat the new
  value once per column, and even then silently do nothing if every column
  is already full.
- **Nothing enforces uniqueness across the columns.** The database can't
  stop the same value from being duplicated in two columns on one row —
  there's no constraint that spans `tag1`, `tag2`, and `tag3` as a set.
- **The column count is a silent ceiling.** Three columns means three tags
  per row, forever, until someone alters the table. Adding a column later
  costs real table-lock/rewrite time on a populated table, and — worse —
  requires revisiting and editing *every* query in every application that
  touches the attribute, since each one hardcodes the current column
  count. Miss one, and it silently keeps ignoring the new column.

---

## 3. Legitimate Uses

- When the "multiple values" are actually different attributes that happen
  to share a data type and order/position matters — e.g. `reported_by`,
  `assigned_to`, `verified_by` on a `Bugs` table are all account
  references, but each means something distinct. Plain, separately-named
  columns are appropriate here; the drawbacks in this skill matter far
  less because you'll rarely need to search across all of them
  interchangeably, and when you occasionally do, that complexity is a fair
  trade for simplicity everywhere else.
- Don't reflexively push this case into a dependent table with a "role"
  column either — that trade brings its own risk of drifting into
  [[sql-antipatterns-entity-attribute-value]]. Decide based on whether the
  values are genuinely interchangeable instances of the same fact (favor a
  dependent table) or distinct facts that happen to share a type (favor
  separate named columns).

---

## 4. The Fix: Create a Dependent Table

Store the multiple values as multiple *rows* in a table of their own, one
value per row, with a foreign key back to the parent:

```sql
CREATE TABLE Tags (
    bug_id BIGINT UNSIGNED NOT NULL,
    tag    VARCHAR(20),
    PRIMARY KEY (bug_id, tag),
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
);

INSERT INTO Tags (bug_id, tag)
VALUES (1234, 'crash'), (3456, 'printing'), (3456, 'performance');
```

This resolves every problem above:

- **Searching is a plain join**, and searching for a combination of values
  is a self-join, not a wall of `OR`s:

  ```sql
  SELECT * FROM Bugs JOIN Tags USING (bug_id) WHERE tag = 'performance';

  SELECT * FROM Bugs
  JOIN Tags AS t1 USING (bug_id)
  JOIN Tags AS t2 USING (bug_id)
  WHERE t1.tag = 'printing' AND t2.tag = 'performance';
  ```

- **Adding/removing is a single `INSERT`/`DELETE`**, with no read-first
  step and no race condition:

  ```sql
  INSERT INTO Tags (bug_id, tag) VALUES (1234, 'save');
  DELETE FROM Tags WHERE bug_id = 1234 AND tag = 'crash';
  ```

- **The `PRIMARY KEY (bug_id, tag)` enforces uniqueness for free** — the
  same tag can't be applied to the same bug twice; the database rejects it
  outright instead of silently allowing it.
- **No column-count ceiling** — a bug can carry as many tags as needed,
  and adding capacity never requires a schema change or touching existing
  queries.

---

## 5. Review Checklist

- Are there numbered/suffixed columns (`tag1`/`tag2`/`tag3`,
  `phone1`/`phone2`, ...) holding the same kind of value on one row?
- Do searches need to check the same condition across several columns
  with `OR`/`IN`?
- Is there a hardcoded, seemingly arbitrary cap on how many values an
  attribute can have, that traces back to a fixed column count?
- Does adding one more value ever require an `ALTER TABLE`, and if so, how
  many existing queries would also need editing?
- If the "multiple columns" are actually distinct attributes (different
  roles, different meanings) rather than repeated instances of the same
  fact, plain separate columns may already be the right call — don't force
  a dependent table where it isn't needed.

---

## Related guidance

PostgreSQL-specific remedy:

- database-postgresql-json-jsonb-handling
