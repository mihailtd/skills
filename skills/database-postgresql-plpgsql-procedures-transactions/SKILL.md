---
name: postgresql-plpgsql-procedures-transactions
description: Guides the creation of PL/pgSQL stored procedures and functions, distinguishing their capabilities for handling intricate transactional logic and batch processing in PostgreSQL.
---

# PostgreSQL PL/pgSQL Procedures and Transactions

This skill helps developers write PostgreSQL routines correctly by distinguishing
between functions and stored procedures, managing transaction control, and
structuring batch-processing logic in PL/pgSQL.

---

## 1. The Critical Divide: Functions vs. Procedures

- **Functions are atomic:** In PostgreSQL, functions are executed as part of a
  normal SQL statement and always run inside the surrounding transaction.
  They cannot issue `COMMIT` or `ROLLBACK` internally.
- **Procedures control transactions:** Stored procedures, invoked via `CALL`,
  are separate objects that can execute `COMMIT`, `ROLLBACK`, and manage
  transaction boundaries from inside their body.
- **Migration trap:** Warn users coming from Oracle or SQL Server that PostgreSQL
  treats functions and procedures as different objects with different behavior.

---

## 2. Writing Stored Procedures for Batch Processing

- **Best use case:** Use procedures for long-running batch jobs, bulk loading,
  and maintenance tasks that benefit from periodic transaction commits.
- **Transaction chaining:** Process large datasets in chunks and issue a
  `COMMIT` periodically to reduce memory pressure, limit lock duration, and
  preserve progress.
- **Syntax example:**

```sql
CREATE PROCEDURE load_with_transform() AS $$
DECLARE
    v_cnt int := 0;
    v_rec record;
BEGIN
    FOR v_rec IN SELECT * FROM data_source LOOP
        -- complex transformation logic
        INSERT INTO target_table VALUES (v_rec.id, v_rec.value);
        v_cnt := v_cnt + 1;

        IF v_cnt >= 50000 THEN
            COMMIT;
            v_cnt := 0;
        END IF;
    END LOOP;

    COMMIT; -- final commit for remaining records
END;
$$ LANGUAGE plpgsql;
```

---

## 3. Structuring PL/pgSQL Code

- **Dollar quoting:** Wrap procedure or function bodies with dollar quoting like
  `$$` or `$BODY$` to avoid escaping single quotes inside the code.
- **Block organization:** Use `DECLARE` to define variables, then place logic
  inside `BEGIN ... END;`.
- **Readable structure:** Keep declarations, exception handlers, and transaction
  control clearly separated.

---

## 4. Flow Control and Error Handling

- **Control structures:** Use `IF ... THEN ... ELSIF ... ELSE`, `CASE`, and
  loops (`FOR`, `WHILE`) to implement business logic cleanly.
- **Raising errors:** Use `RAISE EXCEPTION` to abort execution and roll back the
  transaction, or `RAISE WARNING` / `RAISE NOTICE` to log messages without
  stopping the procedure.

Example:

```sql
BEGIN
    IF v_amount < 0 THEN
        RAISE EXCEPTION 'Amount must be non-negative: %', v_amount;
    END IF;

    -- proceed with logic
EXCEPTION
    WHEN OTHERS THEN
        RAISE WARNING 'Batch failed: %', SQLERRM;
        ROLLBACK;
        RAISE;
END;
```

---

## 5. Practical advice

- Use functions for reusable calculations, queries, and operations that should
  remain inside the caller transaction.
- Use procedures for batch jobs, maintenance tasks, and any flow that requires
  explicit transaction control.
- Keep transaction commits explicit and infrequent in procedures to avoid split-
  brain behavior.
- Always test procedures in a safe environment before running production batch
  jobs.

---

## Example prompt

- "Write a PL/pgSQL stored procedure that processes 100k rows in chunks,
  committing every 50,000 records."
- "Explain why PostgreSQL functions cannot call `COMMIT` and when I should use a
  stored procedure instead."
