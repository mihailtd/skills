---
name: database-postgresql-fdw-federation

description: Guides the setup and management of Foreign Data Wrappers (FDWs) using postgres_fdw to federate data, map users, and query multiple remote PostgreSQL clusters efficiently.
---

# PostgreSQL FDW Federation

This skill helps developers set up and manage `postgres_fdw` to access remote
PostgreSQL data transparently, securely, and with modern SQL/MED federation.

---

## 1. Understand Foreign Data Wrappers and SQL/MED

- **The concept:** Foreign Data Wrappers allow local PostgreSQL to query external
  data sources as if they were local tables.
- **The standard:** `postgres_fdw` is built on SQL/MED (Management of External
  Data), making remote access more standardized and robust.
- **Deprecating `dblink`:** Prefer `postgres_fdw` over legacy `dblink` for better
  transparency, security, and query pushdown.

---

## 2. The Core Setup Architecture

- **Enable the extension:** Install and enable `postgres_fdw` locally with
  `CREATE EXTENSION postgres_fdw;`.
- **Define the remote server:** Use `CREATE SERVER` to declare the remote host,
  port, and database name in the `OPTIONS` clause.

Example:

```sql
CREATE SERVER remote_server
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host '192.168.1.10', dbname 'remote_db', port '5432');
```

- **Create user mappings:** Map local users to remote credentials with
  `CREATE USER MAPPING`. Use `CURRENT_USER` for dynamic local identity mapping
  or specify a local user explicitly.

Example:

```sql
CREATE USER MAPPING FOR CURRENT_USER
SERVER remote_server
OPTIONS (user 'remote_user', password 'remote_password');
```

---

## 3. Exposing Remote Data (Foreign Tables vs. Schema Import)

- **Manual foreign tables:** Use `CREATE FOREIGN TABLE` when you need precise
  control over remote table definitions and column types.
- **Schema import:** Use `IMPORT FOREIGN SCHEMA` to automatically pull table
  definitions from the remote server and reduce manual errors.

Example:

```sql
CREATE FOREIGN TABLE remote_customers (
    id integer,
    name text,
    email text
)
SERVER remote_server
OPTIONS (schema_name 'public', table_name 'customers');
```

Example schema import:

```sql
IMPORT FOREIGN SCHEMA public
FROM SERVER remote_server
INTO foreign_public;
```

---

## 4. Querying, Performance, and Troubleshooting

- **Transparent querying:** Query foreign tables with standard SQL, just like local
  tables.
- **Foreign scans:** The planner uses `Foreign Scan` and pushes down filters,
  joins, and aggregates to the remote server whenever possible.
- **Performance:** Good pushdown behavior minimizes network traffic and improves
  performance.
- **Changes and passwords:** Use `ALTER SERVER` or `ALTER USER MAPPING` to update
  connection options without tearing down the federation.

Examples:

```sql
ALTER SERVER remote_server OPTIONS (SET host 'db.example.com');

ALTER USER MAPPING FOR CURRENT_USER
SERVER remote_server
OPTIONS (SET password 'new_secure_password');
```

- **Troubleshoot:** If pushdown is not happening or performance is poor, verify
  remote permissions, user mapping, and the foreign table definition.

---

## Example prompt

- "Show me how to federate two PostgreSQL databases using `postgres_fdw` and
  import a remote schema."
- "Explain how to map my local user to remote credentials and update the
  password without recreating the foreign table."
- "Describe why `postgres_fdw` is a better choice than `dblink` for modern
  PostgreSQL federation."