---
name: database-postgresql-copy-bulk-loading

description: Guides the usage of the highly optimized COPY command for high-speed bulk data loading, bidirectional extraction, and graceful error handling during imports in PostgreSQL.
---

# PostgreSQL COPY Bulk Loading

This skill helps developers use PostgreSQL's `COPY` command for high-speed bulk
loading, exporting data, and handling import errors gracefully.

---

## 1. Understand the Power of the `COPY` Command

- **The bulk loading standard:** `COPY` is optimized for bulk loading and should
  be used instead of many individual `INSERT` statements for large data loads.
- **Bidirectional capabilities:** `COPY FROM` loads data into a table, while
  `COPY TO` extracts data from a table to a file or standard output.

---

## 2. Formatting and Implementation

- **CSV integration:** Use `WITH (FORMAT csv)` to import or export comma-separated
  values cleanly.
- **Headers and delimiters:** Control file format with options like
  `HEADER on`, `DELIMITER ';'`, and `QUOTE '"'`.
- **External program pipes:** Use `FROM PROGRAM` or `TO PROGRAM` to stream data
  directly to and from shell processes.

Example:

```sql
COPY my_table (id, name, email)
FROM '/tmp/data.csv'
WITH (FORMAT csv, HEADER true, DELIMITER ',');
```

---

## 3. Managing Permissions and Client-Side Execution

- **Privilege restriction:** Regular `COPY` on server-side files requires
  superuser privileges or membership in `pg_write_server_files`.
- **The `\copy` solution:** Use `\copy` from `psql` to stream files from the
  client machine without needing server file privileges.

Example:

```sql
\copy my_table (id, name, email) FROM '/local/path/data.csv' WITH (FORMAT csv, HEADER true);
```

---

## 4. Handling Errors Gracefully (PostgreSQL 17+)

- **Legacy behavior:** Historically, `COPY` aborted the entire load on the first
  invalid row.
- **`ON_ERROR` enhancement:** PostgreSQL 17 adds `WITH (ON_ERROR 'ignore')`
  to skip invalid rows and continue the load, while emitting a `NOTICE` with the
  number of skipped rows.

Example:

```sql
COPY my_table (id, name, email)
FROM '/tmp/data.csv'
WITH (FORMAT csv, HEADER true, ON_ERROR 'ignore');
```

---

## Example prompt

- "Load a large CSV into PostgreSQL using COPY and ignore rows with invalid data."
- "Export a table to CSV with custom delimiter and header row using COPY."
