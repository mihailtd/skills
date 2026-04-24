---
name: postgresql-legacy-database-federation-fdw
description: Guides the implementation of Foreign Data Wrappers (oracle_fdw and mysql_fdw) to securely connect, query, and migrate data from legacy remote databases directly within PostgreSQL.
---

# PostgreSQL Legacy Database Federation with FDW

This skill helps developers integrate legacy Oracle and MySQL systems into
PostgreSQL using Foreign Data Wrappers, enabling secure querying, migration, and
schema federation.

---

## 1. The SQL/MED Standard

- **The concept:** Foreign Data Wrappers let PostgreSQL access external database
  data as if it were stored in local tables.
- **Modernizing connections:** Prefer FDWs over legacy `dblink` because FDWs are
  built on SQL/MED (Management of External Data) and provide better transparency,
  query planning, and performance.

---

## 2. Integrating Oracle via `oracle_fdw`

- **Prerequisites:** `oracle_fdw` requires the Oracle client library (OCI driver)
  installed on the PostgreSQL server.
- **Enable the extension:** `CREATE EXTENSION oracle_fdw;`.
- **Define the server:** Configure the remote Oracle instance with
  `CREATE SERVER` and the `dbserver` option.
- **Map users:** Create a user mapping between local PostgreSQL users and remote
  Oracle credentials.

Example:

```sql
CREATE EXTENSION oracle_fdw;

CREATE SERVER oraserver
FOREIGN DATA WRAPPER oracle_fdw
OPTIONS (dbserver '//dbserver.example.com/ORADB');

CREATE USER MAPPING FOR postgres
SERVER oraserver
OPTIONS (user 'orauser', password 'orapass');
```

- **Schema import:** Use `IMPORT FOREIGN SCHEMA` into a dedicated local schema to
  automatically generate foreign tables and avoid manual definition errors.
- **Limitations:** `oracle_fdw` does not migrate Oracle stored procedures or
  PL/SQL logic automatically; those objects require manual conversion.

Example import:

```sql
IMPORT FOREIGN SCHEMA remote_schema
FROM SERVER oraserver
INTO oracle_foreign;
```

---

## 3. Integrating MySQL via `mysql_fdw`

- **Enable the extension:** Install `mysql_fdw` using the platform package
  manager and enable it with `CREATE EXTENSION mysql_fdw;`.
- **Define the server:** Configure the MySQL host, port, and database name.
- **Map users:** Create user mappings to provide MySQL credentials for PostgreSQL
  users.

Example:

```sql
CREATE EXTENSION mysql_fdw;

CREATE SERVER mysql_server
FOREIGN DATA WRAPPER mysql_fdw
OPTIONS (host 'mysql.example.com', port '3306', dbname 'adventureworks');

CREATE USER MAPPING FOR postgres
SERVER mysql_server
OPTIONS (username 'user', password 'password');
```

- **Manual foreign tables:** Expose MySQL tables with `CREATE FOREIGN TABLE`
  and define columns explicitly for correct type mapping.

Example:

```sql
CREATE FOREIGN TABLE salesorderheader (
    SalesOrderID integer,
    OrderDate timestamp,
    TotalDue numeric
)
SERVER mysql_server
OPTIONS (schema_name 'sales', table_name 'salesorderheader');
```

---

## 4. Query Optimization and Seamless Data Migration

- **Remote execution:** FDWs push down filters, joins, and aggregates to the
  remote database whenever possible, ensuring minimal network traffic.
- **Performance:** Proper remote server and foreign table definitions help
  maximize pushdown and avoid transporting unnecessary data.
- **Data migration:** To permanently migrate rows, use SQL to copy from the FDW
  table into a native PostgreSQL table.

Example migration:

```sql
INSERT INTO local_table
SELECT * FROM foreign_table;
```

- **Security and maintenance:** Manage remote credentials and server options
  carefully, and update mappings when passwords or endpoints change.

---

## Example prompt

- "Show me how to connect PostgreSQL to Oracle using `oracle_fdw` and import a
  remote schema into a local foreign schema."
- "Explain how to expose a MySQL table via `mysql_fdw` and migrate the data into
  a native PostgreSQL table."
- "Describe why FDWs are better than `dblink` for legacy database federation and
  how they optimize remote query execution."
