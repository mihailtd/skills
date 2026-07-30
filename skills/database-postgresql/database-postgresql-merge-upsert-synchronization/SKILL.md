---
name: database-postgresql-merge-upsert-synchronization

description: Guides the flawless execution of MERGE and UPSERT (ON CONFLICT) commands to synchronize large data inserts and updates, including PostgreSQL 17 enhancements and NULL handling.
---

# PostgreSQL MERGE and UPSERT Synchronization

This skill helps developers synchronize datasets in PostgreSQL using
`INSERT ... ON CONFLICT` and `MERGE`, including modern features for NULL handling
and stale-data cleanup.

---

## 1. The PostgreSQL Way of UPSERT (`ON CONFLICT`)

- **The concept:** PostgreSQL does not have a native `UPSERT` keyword.
  Instead it uses `INSERT ... ON CONFLICT` to handle duplicate-key conditions.
- **Handling conflicts:** Use `ON CONFLICT DO NOTHING` to skip duplicate rows
  silently, or `ON CONFLICT (...) DO UPDATE SET ...` to update existing rows.
- **EXCLUDED keyword:** In the update branch, use the `EXCLUDED` table to
  reference incoming values.
- **Constraint requirement:** The `ON CONFLICT` clause must target a unique or
  exclusion constraint.

Example:

```sql
INSERT INTO users (id, email, name)
VALUES (1, 'alice@example.com', 'Alice')
ON CONFLICT (id) DO UPDATE
SET name = EXCLUDED.name;
```

---

## 2. Handling NULLs in UPSERTs (PostgreSQL 15+)

- **The NULL trap:** A composite unique constraint containing `NULL` treats each
  `NULL` as distinct, so duplicates with `NULL` values may still insert.
- **The solution:** In PostgreSQL 15 and later, use `NULLS NOT DISTINCT` on the
  index to make `NULL` values compare as duplicates.

Example:

```sql
CREATE UNIQUE INDEX idx_order_key
ON orders (customer_id, order_number)
WHERE status = 'active'
NULLS NOT DISTINCT;
```

This enables the UPSERT logic to handle rows with `NULL` values in the key.

---

## 3. Mastering the `MERGE` Statement

- **ANSI SQL standard:** Use `MERGE` for complex synchronization scenarios.
  It was introduced in PostgreSQL 15 and follows the SQL standard.
- **The syntax:** `MERGE` compares a source dataset to a target table using a
  join condition, then executes conditional actions.
- **The identity column trap:** If the target has a `GENERATED ALWAYS`
  identity column and you insert explicit primary key values, the `MERGE`
  will fail unless you use `OVERRIDING SYSTEM VALUE`.

Example:

```sql
MERGE INTO target_table t
USING source_table s
ON t.id = s.id
WHEN MATCHED THEN
  UPDATE SET value = s.value
WHEN NOT MATCHED THEN
  INSERT (id, value)
  VALUES (s.id, s.value)
  OVERRIDING SYSTEM VALUE;
```

---

## 4. Advanced `MERGE` Enhancements (PostgreSQL 17+)

- **Deleting stale data:** PostgreSQL 17 adds `WHEN NOT MATCHED BY SOURCE THEN`
  so you can delete or update target rows that no longer exist in the source.
- **Debugging the merge:** Use `RETURNING merge_action()` to see whether each row
  was inserted, updated, or deleted.

Example:

```sql
MERGE INTO target_table t
USING source_table s
ON t.id = s.id
WHEN MATCHED THEN
  UPDATE SET value = s.value
WHEN NOT MATCHED THEN
  INSERT (id, value) VALUES (s.id, s.value)
WHEN NOT MATCHED BY SOURCE THEN
  DELETE
RETURNING merge_action();
```

This returns a column indicating `INSERT`, `UPDATE`, or `DELETE` for each row,
which is very useful for debugging synchronization logic.

---

## Example prompt

- "Synchronize this staging dataset into the production table using PostgreSQL
  MERGE, and delete rows that no longer exist in the source."
- "Write an UPSERT that updates existing rows on conflict and handles NULL values
  in the composite key using PostgreSQL 15 index semantics."
