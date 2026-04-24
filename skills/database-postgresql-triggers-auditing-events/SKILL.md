---
name: postgresql-triggers-auditing-events
description: Guides the accurate implementation of PostgreSQL triggers and trigger functions for automated auditing, event checking, and cascading data adjustments.
---

# PostgreSQL Triggers, Auditing, and Event-Driven Adjustments

This skill helps developers design reliable trigger-based auditing, validation, and
cascading update logic in PostgreSQL using native trigger functions.

---

## 1. Understand Trigger Mechanics and Row vs. Statement Execution

- **The concept:** PostgreSQL triggers act as event handlers for
  `INSERT`, `UPDATE`, `DELETE`, and `TRUNCATE` operations.
- **Timing:** Triggers can fire `BEFORE`, `AFTER`, or `INSTEAD OF` events. The
  latter is only valid for views.
- **Row vs statement scope:** A trigger defined with `FOR EACH ROW` executes once
  per affected row, while `FOR EACH STATEMENT` executes once per SQL command.
- **Execution order:** When multiple triggers exist for the same event on the same
  table, PostgreSQL runs them in alphabetical order. If both rules and triggers
  exist, rules fire first and triggers run afterward.

---

## 2. The Two-Part Architecture (The Function and the Trigger)

- **Trigger function:** Every trigger must call a dedicated function. This
  function returns the special `trigger` type and accepts no normal SQL
  arguments.
- **`OLD` and `NEW`:** The function receives row context automatically.
  `NEW` is available for `INSERT` and `UPDATE`, while `OLD` is available for
  `UPDATE` and `DELETE`.
- **Returning values:** A `BEFORE ... FOR EACH ROW` function must return `NEW`
  to allow the row operation to continue. Returning `NULL` cancels the row.

Example:

```sql
CREATE FUNCTION audit_row_change() RETURNS trigger AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (operation, new_data, changed_at)
        VALUES ('INSERT', row_to_json(NEW), now());
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (operation, old_data, new_data, changed_at)
        VALUES ('UPDATE', row_to_json(OLD), row_to_json(NEW), now());
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (operation, old_data, changed_at)
        VALUES ('DELETE', row_to_json(OLD), now());
        RETURN OLD;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER customers_audit
AFTER INSERT OR UPDATE OR DELETE ON customers
FOR EACH ROW EXECUTE FUNCTION audit_row_change();
```

---

## 3. Implementing Automated Auditing with `TG_OP`

- **Consolidate logic:** Use `TG_OP` inside the trigger function to detect the
  event type and handle `INSERT`, `UPDATE`, and `DELETE` in one place.
- **JSONB audit trails:** Store `old_data JSONB` and `new_data JSONB` in an
  audit table to retain the exact row state before and after the change.

Example audit table:

```sql
CREATE TABLE audit_log (
    id serial PRIMARY KEY,
    table_name text NOT NULL,
    operation text NOT NULL,
    old_data jsonb,
    new_data jsonb,
    changed_by text,
    changed_at timestamptz NOT NULL DEFAULT now()
);
```

Example trigger logic:

```sql
IF TG_OP = 'UPDATE' THEN
    INSERT INTO audit_log (table_name, operation, old_data, new_data)
    VALUES (TG_TABLE_NAME, TG_OP, row_to_json(OLD), row_to_json(NEW));
    RETURN NEW;
END IF;
```

---

## 4. Event Checking and Cascading Adjustments

- **Data validation:** Use `BEFORE` row-level triggers to enforce complex
  business rules or sanitize values before they are written.
- **Modify `NEW`:** Adjust incoming data directly inside the trigger function.
  For example, protect against invalid sensor readings.
- **Cascading logic:** Use `AFTER` triggers to run follow-up calculations or
  update related state after the primary row has been written.

Example validation:

```sql
CREATE FUNCTION enforce_temperature_bounds() RETURNS trigger AS $$
BEGIN
    IF NEW.temperature < -273.15 THEN
        NEW.temperature := -273.15;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_temperature_bounds_trigger
BEFORE INSERT OR UPDATE ON sensor_readings
FOR EACH ROW EXECUTE FUNCTION enforce_temperature_bounds();
```

Example cascading adjustment:

```sql
CREATE FUNCTION recalc_order_totals() RETURNS trigger AS $$
BEGIN
    PERFORM update_order_totals(NEW.order_id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER order_price_changed
AFTER UPDATE OF price ON order_items
FOR EACH ROW EXECUTE FUNCTION recalc_order_totals();
```

---

## 5. Advanced Bulk Processing (Transition Tables)

- **Use statement-level triggers** when bulk operations make row-level triggers too
  expensive.
- **Transition tables:** Use `REFERENCING NEW TABLE AS new_table` or
  `REFERENCING OLD TABLE AS old_table` in a `FOR EACH STATEMENT` trigger to
  process the whole batch at once.
- **Batch auditing:** Query `new_table` or `old_table` inside the trigger
  function to insert audit rows or perform set-based validations efficiently.

Example:

```sql
CREATE FUNCTION bulk_audit_changes() RETURNS trigger AS $$
BEGIN
    INSERT INTO audit_log (table_name, operation, new_data, changed_at)
    SELECT TG_TABLE_NAME, TG_OP, row_to_json(nt), now()
    FROM new_table nt;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER customer_bulk_audit
AFTER INSERT ON customers
REFERENCING NEW TABLE AS new_table
FOR EACH STATEMENT EXECUTE FUNCTION bulk_audit_changes();
```

---

## Example prompt

- "Show me how to build a single trigger function that audits insert, update,
  and delete activity for a table using JSONB audit records."
- "Explain when a `BEFORE` trigger should modify `NEW` and when an `AFTER`
  trigger should perform cascading calculations."
- "Demonstrate how to use transition tables for bulk auditing instead of row-by-row
  triggers."
