---
name: sql-antipatterns-fear-of-the-unknown

description: Detects the Fear of the Unknown antipattern — treating NULL as an ordinary value (or vice versa) in arithmetic, string concatenation, equality comparisons, and IN/NOT IN lists, where SQL's three-valued logic silently produces NULL/unknown instead of the expected result — and guides toward IS NULL, IS DISTINCT FROM, COALESCE(), and NOT NULL constraints used deliberately.
---

# SQL Antipatterns — Fear of the Unknown

This skill helps developers reason correctly about SQL's three-valued logic
(`TRUE`/`FALSE`/`UNKNOWN`) instead of treating `NULL` as if it behaved like
zero, an empty string, or `FALSE` — and instead of avoiding `NULL` entirely by
smuggling in a fake sentinel value.

---

## 1. Recognize the Antipattern

Two mirror-image mistakes, both from misunderstanding what `NULL` means:

- **Treating `NULL` like an ordinary value.** Expecting arithmetic, string
  concatenation, or equality against `NULL` to behave the way it would
  against zero, `''`, or `FALSE`.
- **Treating an ordinary value like `NULL`.** Refusing to allow `NULL` at
  all and picking a sentinel (`-1`, `'N/A'`, a placeholder "unassigned"
  row) to mean "missing," because `NULL`'s behavior feels too error-prone
  to deal with.

**Listen for these phrases**:

- "How do I find rows where no value is set in this column?" — a sign
  `WHERE col = NULL` was tried and silently matched nothing.
- "Some users' full names show up blank in the app, but the data's right
  there in the database." — string concatenation with a `NULL` field
  producing a `NULL` (blank) result.
- "The report only totals bugs that have a priority set!" — an aggregate
  query's `WHERE`/join condition silently excluding `NULL` rows because a
  `<>` comparison against `NULL` is never true.
- "We can't reuse this sentinel value anymore, we need a meeting to pick a
  new one and migrate the data." — the direct cost of having used an
  ordinary value to mean "missing" instead of `NULL`.

---

## 2. Why It Breaks Down

SQL uses three-valued logic: every comparison is `TRUE`, `FALSE`, or
`UNKNOWN` (what `NULL` produces), and a `WHERE` clause only keeps rows
where the condition evaluates to `TRUE` — `UNKNOWN` is treated like
`FALSE` for filtering purposes, but doesn't behave like `FALSE` under
negation.

### NULL in scalar expressions

`NULL` represents "unknown," not zero or empty. An unknown value combined
with anything is still unknown:

| Expression | Expected | Actual | Why |
|---|---|---|---|
| `NULL = 0` | `TRUE` | `NULL` | Null is not zero. |
| `NULL + 12345` | `12345` | `NULL` | Ten more than an unknown is still unknown. |
| `NULL \|\| 'string'` | `'string'` | `NULL` | Null is not an empty string. |
| `NULL = NULL` | `TRUE` | `NULL` | Unknown whether one unspecified value equals another. |
| `NULL <> NULL` | `FALSE` | `NULL` | Also unknown whether they differ. |

This is why `SELECT hours + 10 FROM Bugs` returns `NULL` for any row with
no `hours` estimate, and why concatenating a nullable `middle_initial`
into a name expression blanks out the *entire* name for anyone without
one — not just that one field.

### NULL in boolean expressions

`NULL` is neither `TRUE` nor `FALSE` — it doesn't collapse to `FALSE` under
negation the way people expect:

| Expression | Expected | Actual | Why |
|---|---|---|---|
| `NULL AND FALSE` | `FALSE` | `FALSE` | Any value AND FALSE is false — this one's actually intuitive. |
| `NULL AND TRUE` | `TRUE` | `NULL` | Null is not true. |
| `NULL OR FALSE` | `FALSE` | `NULL` | Null is not false. |
| `NOT (NULL)` | `TRUE` | `NULL` | Negating unknown is still unknown. |

### Equality/inequality never match NULL — including negation

```sql
SELECT * FROM Bugs WHERE assigned_to = 123;
SELECT * FROM Bugs WHERE NOT (assigned_to = 123);
```

Neither query returns rows where `assigned_to IS NULL` — a comparison
against `NULL` is `UNKNOWN`, not `TRUE` or `FALSE`, so it never satisfies a
`WHERE` clause either way. `WHERE assigned_to = NULL` and `WHERE
assigned_to <> NULL` are both always empty for the same reason — this is
one of the most common SQL mistakes and it produces no error, just a
silently wrong result set.

### `NOT IN` with a NULL in the list is a full trap

```sql
SELECT * FROM Bugs WHERE status IN (NULL, 'NEW');      -- matches 'NEW' rows, as expected
SELECT * FROM Bugs WHERE status NOT IN (NULL, 'NEW');   -- matches NOTHING, ever
```

`IN (NULL, 'NEW')` is shorthand for `status = NULL OR status = 'NEW'` — the
`NULL` comparison is just always `UNKNOWN`, so it doesn't hurt; the query
still matches on `'NEW'`. But `NOT IN (NULL, 'NEW')` expands (by De
Morgan's law) to `NOT (status = NULL) AND NOT (status = 'NEW')`. The first
term is `NOT (UNKNOWN)`, which is still `UNKNOWN` — and `UNKNOWN AND
anything` can never become `TRUE`. **If any value in a `NOT IN (...)` list
can be `NULL`, the entire condition matches zero rows, silently, for every
row in the table.** This is easy to hit accidentally when the list comes
from a subquery that might return a `NULL`.

### Avoiding NULL with a sentinel value doesn't avoid the problem

Declaring a column `NOT NULL` and using `-1` or a placeholder row to mean
"unknown" doesn't remove the need for special-case handling — it just
moves it, worse:

```sql
INSERT INTO Bugs (assigned_to, hours) VALUES (-1, -1);

SELECT AVG(hours) FROM Bugs WHERE hours <> -1;  -- must remember to filter every time
```

- `SUM()`/`AVG()` and similar aggregates now silently include the sentinel
  unless every query remembers to exclude it — this is exactly the
  "special-case code" a `NOT NULL` sentinel was meant to avoid, just
  relocated and easier to forget.
- The sentinel must be chosen per column, since a value that's "unused" in
  one column might be legitimate in another, and every choice has to be
  documented and remembered by everyone touching the schema.
- For a foreign key column (like `assigned_to`), there's no valid sentinel
  integer at all — you're forced to create a fake placeholder row in the
  referenced table (an "unassigned" account) purely so the sentinel value
  passes referential integrity, which is backwards.

---

## 3. Legitimate Uses

- `NULL` genuinely needs to be treated as an ordinary value at the
  *boundary* between SQL and something that has no native null
  representation: text-file import/export formats (e.g. `\N` in MySQL's
  `mysqlimport`), or user-input widgets that need an explicit mapping from
  "empty" to null (e.g. .NET's `ConvertEmptyStringToNull`).
- If you need to distinguish *more than one* kind of "missing" — e.g. "bug
  was never assigned" versus "bug was assigned to someone who has since
  left the project" — a single `NULL` can't carry that distinction. That's
  a real case for a deliberate, explicit status/flag value (or a separate
  column) rather than overloading `NULL` to mean two different things.
- Declaring a column `NOT NULL` is correct — even celebrated — whenever a
  row genuinely can't make sense without a value there (e.g.
  `Bugs.reported_by` — every bug was reported by someone). That's not
  "avoiding null," that's using it precisely: null only where a value is
  genuinely optional.

---

## 4. The Fix: Use NULL Precisely

- **Search for `NULL` with `IS NULL`/`IS NOT NULL`**, never `=`/`<>`:

  ```sql
  SELECT * FROM Bugs WHERE assigned_to IS NULL;
  SELECT * FROM Bugs WHERE assigned_to IS NOT NULL;
  ```

- **Use `IS DISTINCT FROM` when you need null-safe inequality**, especially
  with parameterized queries where the parameter itself might be null —
  it always returns `TRUE`/`FALSE`, never `UNKNOWN`:

  ```sql
  SELECT * FROM Bugs WHERE assigned_to IS DISTINCT FROM ?;
  -- equivalent to, but safer/shorter than:
  SELECT * FROM Bugs WHERE assigned_to IS NULL OR assigned_to <> ?;
  ```

  Support varies by brand — PostgreSQL, DB2, and Firebird support it;
  Oracle and SQL Server don't (check current docs, this changes); MySQL
  offers a proprietary `<=>` operator for the same purpose (`IS NOT
  DISTINCT FROM`).

- **Declare `NOT NULL` where a missing value would be nonsensical**, and
  let the database enforce it uniformly instead of relying on application
  code to catch it. Not every `NOT NULL` column needs (or should have) a
  `DEFAULT` — it's fine for a column to require a value with no sensible
  default to fall back to.

- **Use `COALESCE()` to substitute a display-time default without storing
  a fake value**, when an expression needs to tolerate a null input:

  ```sql
  SELECT first_name || COALESCE(' ' || middle_initial || ' ', ' ') || last_name
  AS full_name
  FROM Accounts;
  ```

  Standard SQL; some brands offer an equivalent under another name
  (`NVL()`, `ISNULL()`).

- **Never put a `NULL` into a `NOT IN (...)` list.** If the list comes from
  a subquery, either filter nulls out of it explicitly (`WHERE col IS NOT
  NULL`) or use `NOT EXISTS` instead of `NOT IN`, which doesn't have this
  failure mode.

---

## 5. Review Checklist

- Does any `WHERE`/`JOIN ON` condition compare a nullable column with
  `=`/`<>` where `IS [NOT] NULL` or `IS DISTINCT FROM` was actually needed?
- Does string concatenation or arithmetic involving a nullable column need
  a `COALESCE()` to avoid the whole expression collapsing to null?
- Is there a `NOT IN (subquery)` where the subquery could ever return a
  `NULL`? Prefer `NOT EXISTS`, or explicitly filter nulls from the
  subquery.
- Is a sentinel value (`-1`, `'N/A'`, a placeholder row) standing in for
  "missing" in a `NOT NULL` column, and are aggregate queries against that
  column remembering to exclude it every time?
- Are `NOT NULL` constraints declared wherever a missing value would
  genuinely be nonsensical for that row to exist — enforced by the
  database, not left to application code discipline?
