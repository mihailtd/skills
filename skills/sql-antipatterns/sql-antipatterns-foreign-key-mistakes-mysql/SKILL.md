---
name: sql-antipatterns-foreign-key-mistakes-mysql

description: Checklist of MySQL-specific foreign key mistakes beyond the standard-SQL ones — mismatched storage engines (foreign keys silently ignored on non-InnoDB tables), keys on BLOB/TEXT/JSON columns, non-unique/left-most-subset index references, unsupported inline REFERENCES syntax, unsupported default-column REFERENCES syntax, and TEMPORARY/PARTITIONED table restrictions. Use alongside sql-antipatterns-foreign-key-mistakes-standard-sql when working with MySQL specifically.
---

# SQL Antipatterns — Foreign Key Mistakes in MySQL

MySQL-specific foreign key pitfalls beyond
[[sql-antipatterns-foreign-key-mistakes-standard-sql]] — check both when
defining or debugging a `FOREIGN KEY` in MySQL. Examples show MySQL 8.0
error text; exact wording may vary by version.

---

## 1. Incompatible Storage Engines

Both tables in a foreign key relationship must use the same storage
engine, and that engine must support foreign keys at all. MySQL's default,
InnoDB, does; most others (notably MyISAM) don't.

**If the parent table uses a non-supporting engine**, creating the child
fails outright:

```sql
CREATE TABLE Parent (parent_id INT PRIMARY KEY) ENGINE=MyISAM;
CREATE TABLE Child (
    child_id INT PRIMARY KEY, parent_id INT NOT NULL,
    FOREIGN KEY (parent_id) REFERENCES Parent(parent_id)
) ENGINE=InnoDB;
-- ERROR 1824 (HY000): Failed to open the referenced table 'Parent'
```

**If the *child* table uses a non-supporting engine, this is worse — it
fails silently.** No error, no warning — the table is simply created
*without* the foreign key constraint you wrote, because MySQL historically
allowed importing SQL from other database brands even for features it
hadn't implemented yet, and silently dropped what it couldn't support:

```sql
CREATE TABLE Parent (parent_id INT PRIMARY KEY) ENGINE=InnoDB;
CREATE TABLE Child (
    child_id INT PRIMARY KEY, parent_id INT NOT NULL,
    FOREIGN KEY (parent_id) REFERENCES Parent(parent_id)
) ENGINE=MyISAM;
-- no error — but Child has no foreign key at all
```

**Verify the storage engine explicitly** — don't assume; a table created
without an explicit `ENGINE` should default to InnoDB, but confirm rather
than assume, especially on older configurations or after a migration:

```sql
SHOW CREATE TABLE TableName;
-- or:
SELECT ENGINE FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = ? AND TABLE_NAME = ?;
```

The same same-engine restriction applies to the NDB engine (MySQL
Cluster): if one side uses NDB, the other must too.

---

## 2. Keys on Large Data Types (BLOB/TEXT/JSON)

Standard SQL disallows a `PRIMARY KEY`/`UNIQUE KEY`/foreign key directly on
`BLOB`/`CLOB`/`TEXT`/`JSON`/`ARRAY` columns. MySQL additionally caps
indexed data at 3072 bytes (767 in older versions), and while it allows a
**prefix index** (indexing only the leading N bytes of a long column) for
`PRIMARY KEY`/`UNIQUE KEY` purposes, a foreign key **cannot** reference a
prefix index:

```sql
CREATE TABLE Parent (parent_id TEXT NOT NULL, UNIQUE KEY (parent_id(40)));
CREATE TABLE Child (
    child_id INT PRIMARY KEY, parent_id TEXT NOT NULL,
    KEY (parent_id(40)),
    FOREIGN KEY (parent_id) REFERENCES Parent(parent_id)
);
-- ERROR 1170 (42000): BLOB/TEXT column 'parent_id' used in key specification without a key length
```

**Workaround (MySQL 5.7+):** add a `STORED` generated column that hashes
the long value, and key/reference *that* instead:

```sql
CREATE TABLE Parent (
    parent_id TEXT NOT NULL,
    parent_id_crc INT UNSIGNED AS (CRC32(parent_id)) STORED,
    UNIQUE KEY (parent_id_crc)
);
CREATE TABLE Child (
    child_id INT PRIMARY KEY,
    parent_id_crc INT UNSIGNED,
    FOREIGN KEY (parent_id_crc) REFERENCES Parent(parent_id_crc)
);
```

This carries a real (if small) hash-collision risk — two different texts
producing the same `CRC32` would violate the intended uniqueness. A wider
hash (`MD5()`, `SHA1()`) shrinks but never eliminates that risk. **Better:**
give the parent table a proper surrogate pseudokey (see
[[sql-antipatterns-id-required]]) instead of trying to key on long text
content at all.

---

## 3. Foreign Keys to Non-Unique Indexes (Non-Standard MySQL Extension)

Standard SQL requires a foreign key to reference the *entire* `PRIMARY
KEY`/`UNIQUE KEY` of the parent. InnoDB relaxes this: a foreign key may
reference a **non-unique** index, or a **left-most subset** of a compound
key's columns — but only if the referenced columns are exactly the
left-most columns of some key/index on the parent:

```sql
-- WRONG — parent_id2, parent_id3 aren't the left-most columns of the PK
CREATE TABLE Parent (
    parent_id1 INT, parent_id2 INT, parent_id3 INT,
    PRIMARY KEY (parent_id1, parent_id2, parent_id3)
);
CREATE TABLE Child (
    child_id INT PRIMARY KEY, parent_id2 INT NOT NULL, parent_id3 INT NOT NULL,
    FOREIGN KEY (parent_id2, parent_id3) REFERENCES Parent(parent_id2, parent_id3)
);
-- ERROR 1822 (HY000): Missing index for constraint ...
```

```sql
-- OK — parent_id1, parent_id2 ARE the left-most columns of the PK
FOREIGN KEY (parent_id1, parent_id2) REFERENCES Parent(parent_id1, parent_id2)
```

**Even though this is allowed, avoid it.** Referencing a non-unique index
means a single child row can match *multiple* parent rows, which makes
referential actions logically ambiguous — is a child orphaned if *one* of
several matching parents is deleted? Does `ON DELETE RESTRICT` block
deleting any matching parent, or only the last one? Does `ON UPDATE
CASCADE` propagate from every matching parent, or is that even
well-defined? These questions don't have objectively correct answers —
they're app-specific judgment calls, which is exactly the kind of
ambiguity a foreign key constraint should be eliminating, not
introducing. **Prefer referencing the full `PRIMARY KEY`/`UNIQUE KEY`** so
each child row maps to exactly one parent row, unambiguously.

---

## 4. Inline `REFERENCES` Syntax Is Silently Ignored

Standard SQL lets you attach a single-column foreign key inline, on the
same line as the column definition. **MySQL parses this without error but
silently drops the constraint** — this is a long-standing, well-known
MySQL quirk, not a typo-shaped bug:

```sql
CREATE TABLE Child (
    child_id INT PRIMARY KEY,
    parent_id VARCHAR(10) NOT NULL REFERENCES Parent(parent_id)  -- silently ignored!
);
```

`SHOW CREATE TABLE Child` afterward confirms the constraint simply isn't
there — no error was ever raised to explain why. **Always use table-level
constraint syntax in MySQL:**

```sql
CREATE TABLE Child (
    child_id INT PRIMARY KEY,
    parent_id VARCHAR(10) NOT NULL,
    FOREIGN KEY (parent_id) REFERENCES Parent(parent_id)
);
```

---

## 5. Default (Implicit) Referenced Columns Aren't Supported

Standard SQL allows omitting the referenced column list, defaulting to the
parent's primary key. **MySQL requires the referenced columns to be named
explicitly** — omitting them is an error, not a shorthand:

```sql
FOREIGN KEY (parent_id) REFERENCES Parent  -- no column list
-- ERROR 1239 (42000): Incorrect foreign key definition ...: Key reference and table reference don't match
```

```sql
FOREIGN KEY (parent_id) REFERENCES Parent(parent_id)  -- always name the column(s)
```

---

## 6. Incompatible Table Types in MySQL

Beyond the standard-SQL table-type rule
([[sql-antipatterns-foreign-key-mistakes-standard-sql]] §11), **MySQL
additionally disallows foreign keys where either side is a `TEMPORARY`
table or a `PARTITIONED` table**:

```sql
CREATE TABLE Child (
    ...
    FOREIGN KEY (parent_id) REFERENCES Parent(parent_id)
) PARTITION BY HASH(child_id) PARTITIONS 11;
-- ERROR 1506 (HY000): Foreign keys are not yet supported in conjunction with partitioning
```

```sql
CREATE TEMPORARY TABLE Child (
    ...
    FOREIGN KEY (parent_id) REFERENCES Parent(parent_id)
);
-- ERROR 1215 (HY000): Cannot add foreign key constraint
```

Both parent and child must be ordinary, non-temporary, non-partitioned
tables.

---

## Review Checklist

- Do both tables in the relationship use the same storage engine, and does
  that engine actually support foreign keys (InnoDB, or NDB on MySQL
  Cluster)? Confirm with `SHOW CREATE TABLE`/`INFORMATION_SCHEMA.TABLES`
  rather than assuming.
- Is a foreign key ever attempted against a `BLOB`/`TEXT`/`JSON` column or
  a prefix index on one? If long text needs to be referenced, is a stored
  generated-column hash (or better, a real surrogate key) used instead?
- Does any foreign key reference a non-unique index or a left-most subset
  of a compound key? If so, is the resulting one-to-many-parent ambiguity
  (orphaning, `RESTRICT`, `CASCADE` behavior) actually acceptable — or
  should the FK reference the full key instead?
- Is every foreign key defined with table-level `FOREIGN KEY (...)
  REFERENCES ...(...)` syntax, never inline `REFERENCES` on a column
  definition (silently ignored in MySQL)?
- Does every `REFERENCES` clause explicitly name the referenced column(s)?
- Are both tables ordinary (non-`TEMPORARY`, non-`PARTITIONED`)?
