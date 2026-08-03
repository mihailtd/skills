---
name: sql-antipatterns-foreign-key-mistakes-standard-sql

description: Checklist of standard-SQL foreign key mistakes — wrong reference direction, table creation order, referencing a non-key column, splitting a compound key into separate constraints, wrong column order, mismatched data types/collations, orphaned data, SET NULL on a NOT NULL column, duplicate constraint names, and mismatched table types. Use when defining, debugging, or reviewing FOREIGN KEY constraints in any SQL database brand.
---

# SQL Antipatterns — Foreign Key Mistakes in Standard SQL

A checklist of foreign key mistakes that apply across SQL database brands
(examples show MySQL 8.0 error text; exact wording varies by brand, but
the underlying rule doesn't). Use this when defining a new foreign key,
debugging one that won't create, or reviewing schema changes involving
`REFERENCES`.

Convention used throughout: `Parent` is the "one" side, `Child` is the
"many" side of a one-to-many relationship — a `Parent` row may have zero,
one, or many related `Child` rows; a `Child` row must relate to exactly one
`Parent` row.

---

## 1. Reversing the Direction of Reference

Thinking "Parent can have many Children" can lead to putting the foreign
key on the wrong table:

```sql
-- WRONG — a Parent row can only reference one Child, the opposite of intent
CREATE TABLE Parent (
    parent_id INT PRIMARY KEY,
    child_id  INT NOT NULL,
    FOREIGN KEY (child_id) REFERENCES Child(child_id)
);
```

This doesn't error — it just silently models the wrong relationship (each
`Parent` can only point at one `Child`, while any number of `Parent` rows
could point at the *same* `Child`). Think of it instead as "Child belongs
to Parent," and put the foreign key on the many side:

```sql
CREATE TABLE Child (
    child_id  INT PRIMARY KEY,
    parent_id INT NOT NULL,
    FOREIGN KEY (parent_id) REFERENCES Parent(parent_id)
);
```

**Rule: the foreign key constraint belongs on the "many" side of a
one-to-many relationship.**

---

## 2. Referencing a Table Before It Exists

```sql
-- WRONG — Parent doesn't exist yet when Child is created
CREATE TABLE Child (
    child_id  INT PRIMARY KEY,
    parent_id INT NOT NULL,
    FOREIGN KEY (parent_id) REFERENCES Parent(parent_id)
);
CREATE TABLE Parent (parent_id INT PRIMARY KEY);
-- ERROR 1824 (HY000): Failed to open the referenced table 'Parent'
```

Create the referenced table first. For **mutual references** (each table
references the other), you can't satisfy both at creation time — create
one table without its FK, create the second table referencing the first,
then `ALTER TABLE` the first to add its FK to the second:

```sql
CREATE TABLE Parent (parent_id INT PRIMARY KEY, favorite_child_id INT);
CREATE TABLE Child (
    child_id  INT PRIMARY KEY,
    parent_id INT NOT NULL,
    FOREIGN KEY (parent_id) REFERENCES Parent(parent_id)
);
ALTER TABLE Parent ADD FOREIGN KEY (favorite_child_id) REFERENCES Child(child_id);
```

A self-referential foreign key can be defined directly in its own `CREATE
TABLE`. For a large schema, either order table creation starting from
tables with no FKs ("root" tables) outward, or — simpler when there are
cycles — create every table first with no FK constraints, then add all
constraints afterward via `ALTER TABLE`.

---

## 3. Referencing a Column That Isn't a Key

```sql
-- WRONG — parent_id has no PRIMARY KEY or UNIQUE constraint
CREATE TABLE Parent (parent_id INT NOT NULL);
CREATE TABLE Child (
    child_id  INT PRIMARY KEY,
    parent_id INT NOT NULL,
    FOREIGN KEY (parent_id) REFERENCES Parent(parent_id)
);
-- ERROR 1822 (HY000): Missing index for constraint ... in the referenced table 'Parent'
```

**Rule: a foreign key must reference a `PRIMARY KEY` or `UNIQUE` key of the
parent table** — not just any column, however unique its values happen to
be in practice.

---

## 4. Splitting a Compound Key Into Separate Constraints

If the parent's primary key spans multiple columns, don't define one FK
per column — that doesn't express "these columns together identify one
parent row":

```sql
-- WRONG
CREATE TABLE Parent (parent_id1 INT, parent_id2 INT, PRIMARY KEY (parent_id1, parent_id2));
CREATE TABLE Child (
    child_id INT PRIMARY KEY,
    parent_id1 INT NOT NULL, parent_id2 INT NOT NULL,
    FOREIGN KEY (parent_id1) REFERENCES Parent(parent_id1),
    FOREIGN KEY (parent_id2) REFERENCES Parent(parent_id2)
);
-- ERROR 1822 (HY000): Missing index for constraint ...
```

Define one compound foreign key referencing both columns together:

```sql
FOREIGN KEY (parent_id1, parent_id2) REFERENCES Parent(parent_id1, parent_id2)
```

---

## 5. Wrong Column Order in a Compound Key

Column order in the FK must match the order in the parent's key —
otherwise you may get no error at creation time, but inserts fail (or
silently reference the wrong logical columns if types happen to be
compatible):

```sql
-- WRONG — parent_id2 and parent_id1 are swapped relative to Parent's PK order
FOREIGN KEY (parent_id2, parent_id1) REFERENCES Parent(parent_id1, parent_id2)
-- ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails
```

**Rule: list foreign key columns in the same order as they appear in the
parent's `PRIMARY KEY`/`UNIQUE` definition.**

---

## 6. Mismatched Data Types

```sql
-- WRONG — INT vs VARCHAR
CREATE TABLE Parent (parent_id INT PRIMARY KEY);
CREATE TABLE Child (
    child_id INT PRIMARY KEY,
    parent_id VARCHAR(10) NOT NULL,
    FOREIGN KEY (parent_id) REFERENCES Parent(parent_id)
);
-- ERROR 3780 (HY000): ... are incompatible.
```

Even a **signed vs. unsigned** integer mismatch is enough to fail:

```sql
-- WRONG — INT vs INT UNSIGNED
parent_id INT UNSIGNED NOT NULL  -- referencing a signed INT PRIMARY KEY
-- ERROR 3780 (HY000): ... are incompatible.
```

**Rule: match data types exactly** (including signedness). One narrow
exception: `VARCHAR` columns of *different maximum lengths* are still
compatible with each other. That's not license to ignore it, though —
a shorter child column can only ever reference parent values that fit
within its own length, and a shorter parent column limits what the child
can be inserted as; mismatched lengths just don't error, they change what
data is actually reachable.

---

## 7. Mismatched Character Collations

Even when the character set matches, a different **collation** (the rule
for how characters compare) makes columns incompatible for FK purposes:

```sql
-- WRONG — utf8mb4_unicode_ci vs utf8mb4_general_ci
CREATE TABLE Parent (parent_id VARCHAR(10) PRIMARY KEY)
    CHARSET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE TABLE Child (
    child_id INT PRIMARY KEY,
    parent_id VARCHAR(10) NOT NULL,
    FOREIGN KEY (parent_id) REFERENCES Parent(parent_id)
) CHARSET utf8mb4 COLLATE utf8mb4_general_ci;
-- ERROR 3780 (HY000): ... are incompatible.
```

**Rule: string columns in a foreign key relationship need matching
character set and collation** — in practice, make the collations
identical.

---

## 8. Creating Orphan Data

Adding a foreign key to a table that already has rows requires every
existing value to already match a parent row:

```sql
-- Child has a row referencing parent_id 5678, which doesn't exist in Parent
ALTER TABLE Child ADD FOREIGN KEY (parent_id) REFERENCES Parent(parent_id);
-- ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails
```

Check for orphans before adding the constraint:

```sql
SELECT CASE COUNT(*)
    WHEN 0 THEN 'Ready to add foreign key'
    ELSE 'Do not add foreign key, because orphan rows exist'
END AS `check`
FROM Child
LEFT OUTER JOIN Parent ON Child.parent_id = Parent.parent_id
WHERE Parent.parent_id IS NULL;
```

Fix orphans before adding the FK, by one of: inserting the missing parent
rows, updating the orphaned child values (to `NULL` or a valid reference),
or deleting the orphaned child rows.

---

## 9. `SET NULL` on a Non-Nullable Column

```sql
-- WRONG — parent_id is NOT NULL but the action tries to null it out
CREATE TABLE Child (
    child_id  INT PRIMARY KEY,
    parent_id INT NOT NULL,
    FOREIGN KEY (parent_id) REFERENCES Parent(parent_id) ON DELETE SET NULL
);
-- ERROR 1830 (HY000): Column 'parent_id' cannot be NOT NULL: needed in a foreign key constraint ... SET NULL
```

**Rule: a column used with `ON UPDATE SET NULL`/`ON DELETE SET NULL` must
itself be nullable.** (See [[sql-antipatterns-keyless-entry]] for how to
choose the right `ON DELETE`/`ON UPDATE` action in general.)

---

## 10. Duplicate Constraint Names

Named constraints must be unique **within the whole schema**, not just
within one table:

```sql
-- WRONG — both use constraint name c1
CREATE TABLE Child1 (..., CONSTRAINT c1 FOREIGN KEY (parent_id) REFERENCES Parent(parent_id));
CREATE TABLE Child2 (..., CONSTRAINT c1 FOREIGN KEY (parent_id) REFERENCES Parent(parent_id));
-- ERROR 1826 (HY000): Duplicate foreign key constraint name 'c1'
```

If you name constraints explicitly, use a naming convention that
guarantees uniqueness across the schema (e.g. including the table name).
If you don't name them, the database generates unique names for you.

---

## 11. Incompatible Table Types

In standard SQL, the parent and child tables must be the same *kind* of
table — both ordinary persistent base tables, or both global temporary
tables, or both local temporary tables. Mixing kinds isn't valid. (MySQL
has additional table-type restrictions of its own — see
[[sql-antipatterns-foreign-key-mistakes-mysql]].)

---

## Review Checklist

- Is the foreign key defined on the "many" side of the relationship, not
  the "one" side?
- Are referenced tables created (or constraints added via `ALTER TABLE`
  afterward) in an order that resolves creation dependencies, including
  mutual/self references?
- Does the FK reference an actual `PRIMARY KEY`/`UNIQUE` key of the parent,
  not just some column?
- For a compound parent key: is there one compound FK (not one FK per
  column), with columns listed in the same order as the parent's key?
- Do the referencing and referenced columns have identical data types
  (including signedness) and, for strings, matching collation?
- Before adding a FK to a populated table, has the data been checked for
  orphaned rows and fixed?
- Does any `SET NULL` action apply only to nullable columns?
- Are named constraints unique across the whole schema, not just the
  table?
- Do parent and child belong to compatible table types (persistent vs.
  temporary)?
