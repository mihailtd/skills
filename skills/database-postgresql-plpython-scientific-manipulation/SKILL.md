---
name: postgresql-plpython-scientific-manipulation
description: Guides the creation and management of PL/Python stored procedures in PostgreSQL for advanced mathematical, scientific, and AI-driven data manipulation.
---

# PostgreSQL PL/Python Scientific Manipulation

This skill helps developers use PL/Python in PostgreSQL to perform advanced
mathematical, scientific, or AI-driven computations directly inside the database.

---

## 1. Enable the PL/Python Language (The Untrusted Caveat)

- **Installation:** PostgreSQL can be extended with embedded languages like
  Python. Install the OS package such as `postgresql-plpython3` and enable it
  with `CREATE EXTENSION plpython3u;`.
- **The untrusted warning:** PL/Python is only available as an untrusted
  language, `plpython3u`.
- **Security implications:** Untrusted functions can perform system-level
  operations and run with the database server daemon’s OS permissions.
  Therefore only superusers may create PL/Python functions, and use it with
  extreme caution.

---

## 2. Writing PL/Python Functions for Math and AI

- **The concept:** Bringing Python to the data avoids expensive network round
  trips for large datasets and enables direct use of scientific libraries.
- **Syntax example:** Write functions in Python 3 with proper indentation inside
  dollar-quoted SQL strings.
- **Library usage:** You can import standard modules such as `math` and external
  libraries if they are installed on the database server.

Example:

```sql
CREATE OR REPLACE FUNCTION calculate_scientific_model(val float)
RETURNS numeric AS $$
    if val <= 0:
        plpy.error('invalid input for model')
    import math
    return math.sqrt(val) * 0.42
$$ LANGUAGE plpython3u;
```

---

## 3. Fetching and Manipulating Data (The SPI Interface)

- **Data access:** Use the `plpy` module and the Server Programming Interface
  (SPI) to execute SQL inside the Python function.
- **Large datasets:** For massive scientific datasets, prefer `plpy.cursor()`
  over `plpy.execute()` to fetch rows in chunks and avoid exhausting memory.

Example:

```python
cursor = plpy.cursor("SELECT val FROM telemetry_data")
while True:
    rows = cursor.fetch(1000)
    if not rows:
        break
    # perform matrix calculations here
```

---

## 4. Managing Output and Error Handling

- **Logging and diagnostics:** Use `plpy.notice()` to emit messages for debugging
  and status reporting.
- **Exception handling:** Catch `plpy.SPIError` and other exceptions to recover
  gracefully or log errors without immediately aborting the entire transaction.

Example:

```python
try:
    result = plpy.execute("SELECT ...")
except plpy.SPIError as e:
    plpy.notice(f'Processing error: {e}')
    raise
```

---

## Example prompt

- "Create a PL/Python function that reads telemetry data in chunks and computes a
  rolling average using NumPy."
- "Explain how to safely enable and use `plpython3u` for scientific processing
  in PostgreSQL and how to avoid security risks."
