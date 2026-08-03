---
name: sql-antipatterns-jaywalking

description: Detects and fixes the Jaywalking antipattern — storing a multivalue or many-to-many relationship as a comma-separated list of IDs in a single column — by replacing it with a proper intersection table.
---

# SQL Antipatterns — Jaywalking

This skill helps developers recognize when a comma-separated list of foreign
keys has been used to dodge a many-to-many relationship, and guides them back
to a normalized intersection table. Named because avoiding the intersection
(table) is like jaywalking.

---

## 1. Recognize the Antipattern

- **The tell:** a column that should reference another table (`account_id`,
  `tag_id`, `user_id`, ...) is instead typed `VARCHAR`/`TEXT` and holds values
  like `'12,34,56'`.
- **The origin story:** it usually starts as a genuine one-to-many column
  (`account_id INT`), and when a one-to-many relationship needs to become
  many-to-many, someone widens the column to a string instead of adding a
  table.
- **Listen for these phrases** in code review or design discussion — they're
  a strong signal this antipattern is present or about to be introduced:
  - "What's the greatest number of entries this list needs to support?"
  - "Do you know how to match a word boundary in SQL?" (i.e. reaching for
    `REGEXP '\\bID\\b'` to query the list)
  - "What character will never appear in an entry?" (choosing a separator)

```sql
-- antipattern
CREATE TABLE Products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(1000),
    account_id   VARCHAR(100)  -- comma-separated list, e.g. '12,34'
);
```

---

## 2. Why It Breaks Down

- **Querying is a pattern match, not a lookup:** finding rows for one account
  requires `REGEXP`/`LIKE` tricks instead of `=`, can't use an index, and
  isn't portable across database brands.

  ```sql
  SELECT * FROM Products WHERE account_id REGEXP '\\b12\\b';
  ```

- **Joins degrade into cross products:** joining the list column back to the
  referenced table forces the optimizer to evaluate a regex per combination
  of rows — no index can help.
- **Aggregates need string arithmetic:** counting members of the list means
  hacks like `LENGTH(col) - LENGTH(REPLACE(col, ',', '')) + 1` instead of
  `COUNT(*) ... GROUP BY`. Some aggregates (e.g. per-member `AVG`) simply
  can't be expressed this way at all.
- **Updates require read-modify-write:** removing one member means fetching
  the whole list, splitting it in application code, rejoining it, and writing
  it back — never a single atomic statement.
- **No referential integrity:** a `FOREIGN KEY` can validate a whole column,
  not one element of a delimited list, so nothing stops `'12,34,banana'`
  from being inserted.
- **The separator is never actually safe:** any character you pick as a
  delimiter can eventually show up inside a value.
- **The list has a silent length ceiling:** how many IDs fit depends on how
  many digits each ID has, so the same `VARCHAR(n)` column can silently start
  rejecting inserts as IDs grow — a bug that's miserable to explain and
  impossible to size correctly up front.

---

## 3. Legitimate Exceptions

- The data arrives as a delimited string from an external system, is stored
  and returned as-is, and nothing ever needs to query, join, count, or
  validate individual members. Denormalization for a genuine write-once/
  read-as-blob field is fine.
- Start normalized by default. Only denormalize into a delimited string after
  proving a specific read-performance need — don't reach for it as the
  default way to model a one-to-many or many-to-many relationship.
- Native array types (PostgreSQL `ARRAY`, and similar support in Oracle,
  DB2, Informix) reduce some of the pain — typed elements, no separator
  ambiguity — but still can't enforce a foreign key on individual elements
  and are non-portable. Prefer them over ad hoc CSV strings if you must
  denormalize, but don't use them as a substitute for an intersection table
  when the relationship needs real referential integrity or per-row queries.

**PostgreSQL note:** among mainstream databases, Postgres's `ARRAY`
support is unusually complete, which makes the "prefer arrays over CSV
strings if you must denormalize" advice above considerably more viable
there than on most engines:

```sql
CREATE TABLE Products (
    product_id   SERIAL PRIMARY KEY,
    product_name VARCHAR(1000),
    account_ids  INTEGER[]   -- typed array of account IDs
);

-- indexed containment/overlap queries via GIN
CREATE INDEX products_account_ids_gin ON Products USING GIN (account_ids);
SELECT * FROM Products WHERE account_ids @> ARRAY[12];       -- contains 12
SELECT * FROM Products WHERE account_ids && ARRAY[12, 34];   -- overlaps either

-- unnest() expands the array back to one row per value when you need it
SELECT product_id, unnest(account_ids) AS account_id FROM Products;
```

This closes off the two worst problems from §2: values are actually typed
(`INTEGER[]` rejects `'banana'` outright, unlike a CSV string), and with a
`GIN` index, containment/overlap queries use the index instead of forcing
a full scan — genuinely fast, not just "technically indexable." What it
still doesn't give you, so the intersection-table recommendation in this
skill still holds for anything beyond the narrow denormalization case in
§3: **no `FOREIGN KEY` on individual array elements** (an ID can go stale
— point at a deleted account — with nothing enforcing otherwise), **no
per-element extra attributes** (an intersection table can carry
`added_at`/`is_primary` per pairing; an array element can't carry
anything), and it's still a **PostgreSQL-specific feature**, not portable
SQL. Reach for `ARRAY` when §3's denormalization case genuinely applies
and you're committed to Postgres — not as a default replacement for a
many-to-many relationship that needs real referential integrity.

---

## 4. The Solution: an Intersection Table

Move each list entry to its own row in a table with a foreign key to each
side of the relationship. The two foreign keys together (or a surrogate key)
form the primary key.

```sql
CREATE TABLE Contacts (
    product_id BIGINT UNSIGNED NOT NULL,
    account_id BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (product_id, account_id),
    FOREIGN KEY (product_id) REFERENCES Products(product_id),
    FOREIGN KEY (account_id) REFERENCES Accounts(account_id)
);
```

This fixes every problem above:

```sql
-- lookup uses an index, not a regex
SELECT p.*
FROM Products AS p
JOIN Contacts AS c ON p.product_id = c.product_id
WHERE c.account_id = 34;

-- aggregates are plain GROUP BY
SELECT product_id, COUNT(*) AS accounts_per_product
FROM Contacts
GROUP BY product_id;

-- add/remove one member = one row, one statement
INSERT INTO Contacts (product_id, account_id) VALUES (456, 34);
DELETE FROM Contacts WHERE product_id = 456 AND account_id = 34;
```

- **No length ceiling** other than how many rows a table can hold — if you
  need a policy limit, enforce it with a `COUNT(*)` check in application
  logic, not a column width.
- **No separator ambiguity** — nothing is ever concatenated.
- **Full referential integrity** — the `FOREIGN KEY`s guarantee every member
  actually exists in the referenced table.
- **Room to grow** — extra columns on the intersection table (e.g. `added_at`,
  `is_primary_contact`) can attach metadata to each association, which a CSV
  column can never do.

---

## 5. If You're Stuck Reading Legacy CSV Columns

When a comma-separated column can't be migrated yet but a query needs one row
per value, expand it rather than parsing it in application code:

```sql
-- PostgreSQL
SELECT a FROM Products
CROSS JOIN unnest(string_to_array(account_id, ',')) AS a;

-- SQL Server 2016+
SELECT product_id, product_name, value
FROM Products CROSS APPLY STRING_SPLIT(account_id, ',');
```

Treat this as a stopgap for reporting, not a reason to leave the underlying
design unfixed — plan the migration to an intersection table.

---

## 6. Review Checklist

- Is any FK-like column typed as a string instead of an integer/UUID
  referencing another table?
- Does any query use `LIKE`/`REGEXP` against that column to find "does X
  belong to this list" matches?
- Is there a hardcoded max-length assumption on that column that maps to "max
  number of related entries"?
- Would adding metadata to one association (who added it, when, is it
  primary) currently require a bigger hack than adding a column?

If yes to any of these, propose an intersection table.
