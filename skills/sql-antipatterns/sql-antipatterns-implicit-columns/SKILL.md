---
name: sql-antipatterns-implicit-columns

description: Detects the Implicit Columns antipattern — using SELECT * or column-less INSERT statements — which silently breaks when columns are added, dropped, renamed, or reordered, produces name collisions in joins, and wastes bandwidth fetching unneeded data, and guides toward always naming columns explicitly.
---

# SQL Antipatterns — Implicit Columns

This skill helps developers recognize when `SELECT *` or an `INSERT`
without a column list is standing in for explicitly named columns, and
guides them toward always spelling columns out — the small extra typing
buys real protection against silent breakage.

---

## 1. Recognize the Antipattern

- **The tell:** a query fetches all columns implicitly instead of naming
  what it actually needs:

  ```sql
  SELECT * FROM Bugs;
  INSERT INTO Accounts VALUES (DEFAULT, 'bkarwin', 'Bill', 'Karwin', ...);
  ```

- The motivation is understandable — less typing — but every convenience
  it buys comes with a corresponding way for the query to silently break
  or misbehave as the schema evolves. See §2.

---

## 2. Why It Breaks Down

- **Column name collisions in joins produce silent data loss.** If two
  joined tables both have a column with the same name (e.g. both `Books`
  and `Authors` have a `title` column), `SELECT *` returns both, but
  code that reads results by name (an associative array, a dict, a JSON
  object) can only keep one — the other is silently overwritten. The
  classic failure mode: a book's `title` looks `NULL` in application code
  because the *author's* `title` (a salutation, usually empty) silently
  won the collision.
- **`INSERT` without a column list depends entirely on column order
  matching the table definition.** Add a column to the table and every
  existing implicit-column `INSERT` either errors (`SQLSTATE 21S01: Column
  count doesn't match value count`) — the "good" failure — or, worse,
  silently shifts every subsequent value into the wrong column if the
  count happens to still match. Reference explicit column *positions*
  after a `SELECT *` (`row[10]`) and the same risk hits reads: drop or
  reorder a column and the value your code reads from position 10 is now
  a different column entirely, with no error at all.
- **A query's result shape becomes unpredictable across schema changes.**
  Nothing about `SELECT *`/implicit `INSERT` in application code documents
  which columns it actually depends on — so a schema change anywhere in
  the table can silently break code far away from the change, and the
  failure (wrong value in the wrong field) can surface long after the
  actual mistake, making it hard to trace back.
- **Wasted bandwidth.** Every unneeded column fetched is data that has to
  travel from the database to the application, for every row, on every
  query. This is easy to overlook in development against small datasets,
  but at production scale — many concurrent queries fetching more than
  they display — it becomes real, measurable network cost. ORMs that
  default to `SELECT *` for hydrating objects (e.g. Active Record-style
  patterns) inherit this cost by default, even when an override exists
  that most projects never bother to use.
- **There's no "all columns except these few" syntax in SQL.** If you
  actually want to trim just a handful of unwanted (e.g. bulky `TEXT`)
  columns from a wide table, you still have to name the columns you *do*
  want — SQL has no `SELECT * EXCEPT (...)` construct in standard syntax.
  Wanting to avoid typing a short exclude-list doesn't actually save
  typing once you realize there's no shortcut for it; you end up naming
  columns explicitly either way.

---

## 3. Legitimate Uses

- Ad hoc, single-use queries — quick diagnostic checks, exploring data
  interactively — genuinely benefit less from the durability that explicit
  columns buy. There's no future breakage to guard against if the query is
  never run again.
- If a query is deliberately meant to adapt automatically as columns are
  added/dropped/renamed (rare, but real), a wildcard may be exactly the
  right tool — just budget for the extra debugging effort this trades in
  return, since the failure modes in §2 become a feature you're opting
  into rather than an accident.
- **Per-table wildcards in a join** are a reasonable middle ground: use
  `t.*` to take everything from one table while still naming specific
  columns from the other side, when one side's shape is stable and
  well-understood and you don't want to enumerate it in full:

  ```sql
  SELECT b.*, a.first_name, a.email
  FROM Bugs b JOIN Accounts a ON (b.reported_by = a.account_id);
  ```

- If shorter, more readable queries are genuinely a higher priority than
  runtime efficiency or schema-change resilience for a given use case,
  that's a legitimate trade-off to make deliberately — just recognize it
  as a trade-off, not a free convenience.
- The extra bytes in a longer, fully-spelled-out query string are rarely
  the actual bandwidth cost that matters — the *rows returned* usually
  dwarf the query text itself. Don't let a wildcard's "shorter query"
  argument stand in for "less network usage" — measure which one actually
  matters before deciding based on it.

---

## 4. The Fix: Name Columns Explicitly

Spell out the columns you need in both `SELECT` and `INSERT`:

```sql
SELECT bug_id, date_reported, summary, description, resolution,
       reported_by, assigned_to, verified_by, status, priority, hours
FROM Bugs;

INSERT INTO Accounts (account_name, first_name, last_name, email,
                       password_hash, portrait_image, hourly_rate)
VALUES ('bkarwin', 'Bill', 'Karwin', 'bill@example.com',
        SHA2('xyzzy'), NULL, 49.95);
```

This is the poka-yoke ("mistake-proofing") principle from
[[sql-antipatterns-keyless-entry]] applied to column lists:

- A column repositioned in the table doesn't reposition in your query's
  result — your query's column order is now independent of the table's
  physical column order.
- A newly added column doesn't silently appear in results you weren't
  expecting, and doesn't shift an implicit `INSERT`'s values into the
  wrong slots.
- A dropped column that your query still references produces an
  immediate, precise error pointing at the exact query that needs fixing
  — instead of a silently wrong value discovered much later, far from the
  actual cause. This is the "fail early" principle: a loud failure at the
  point of the actual problem beats a quiet failure somewhere downstream.

As a side effect, explicitly listing columns naturally prompts you to ask
"do I actually need all of these?" — which tends to trim queries down to
only the columns actually used, cutting the same bandwidth cost that
motivated reaching for a wildcard in the first place, just achieved
correctly instead of accidentally.

Note that the moment a query needs a computed expression, a column
alias, or to exclude specific columns, you have to abandon the wildcard
anyway — there's no way to keep using `*` once you need to treat any one
column individually. Starting with explicit columns means that
transition never has to happen later.

---

## 5. Review Checklist

- Does any `SELECT *` join two tables that could plausibly share a column
  name, risking a silent overwrite in code that reads results by name?
- Does any `INSERT` omit its column list, relying on positional order
  matching the table definition exactly?
- Does any application code read query results by ordinal position
  (`row[10]`) rather than by column name?
- Would a schema change (add/drop/rename/reorder a column) silently break
  this query's consumers, or would it fail loudly at the actual point of
  mismatch?
- Is a wildcard fetching columns the query's consumer never actually uses
  — bandwidth spent for no benefit?
- If a wildcard is used deliberately (ad hoc query, intentional
  schema-adaptive behavior, single-table wildcard in a join), is that a
  conscious choice, or just the path of least typing?
