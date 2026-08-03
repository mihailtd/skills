---
name: sql-antipatterns-reserved-words

description: Explains SQL syntax errors caused by using a reserved keyword (order, from, table, year_month, ...) as an unquoted identifier, and how to fix them with the database's delimited-identifier syntax (double quotes, backticks, or square brackets) — or better, avoid reserved words as identifiers in the first place.
---

# SQL Antipatterns — Reserved Words as Identifiers

This skill helps diagnose a specific, confusing class of SQL syntax error:
a column or table name that happens to be a reserved keyword, used
unquoted.

---

## 1. Recognize the Situation

- A query that looks syntactically correct fails with a parser error
  pointing at a word that seems like an ordinary identifier:

  ```
  ERROR 1064: You have an error in your SQL syntax; ... near 'order'.
  ```

  ```sql
  SELECT * FROM Bugs WHERE order = 123;
  ```

- The column name (`order` here) collides with an SQL reserved keyword
  (`ORDER`, as in `ORDER BY`). SQL is case-insensitive for keywords by
  default, so `order` and `ORDER` are the same token to the parser —
  there's nothing wrong with the query's structure, only with using a
  keyword where an identifier was intended without marking it as one.
- Frequently-collided words beyond `order`: `by`, `default`, `from`,
  `rows`, `row_number`, `table`, `to`, `year_month`, and others — every SQL
  dialect has its own reserved list, and it grows as new syntax features
  are added.

---

## 2. Why It Happens

- Reserved keywords exist so the parser never has to guess whether a
  token is a language construct or a user identifier — if `order` could be
  either, `SELECT * FROM t ORDER BY x` would be ambiguous to parse. This
  is normal in essentially every programming language (Java reserves
  `class`, `while`, `try`, etc.), and SQL is no exception.
- The trap is specific to SQL among languages you might compare it to:
  languages with sigils (PHP/Perl's `$var`) never collide, because
  variables and keywords are visually distinct even before parsing. SQL
  identifiers look exactly like keywords unless explicitly delimited.
- **Upgrades can turn previously-safe identifiers into collisions.** New
  database versions add new syntax, which means new reserved words. A
  column name that's been fine for years can suddenly need delimiting
  after an upgrade, with no change on your side.

---

## 3. The Fix: Delimit the Identifier — or Rename It

If you must use a reserved word as an identifier, mark it explicitly so the
parser treats it as one:

```sql
-- SQL standard: double quotes
SELECT * FROM Bugs WHERE "order" = 123;

-- MySQL default: backticks
SELECT * FROM Bugs WHERE `order` = 123;

-- SQL Server default: square brackets
SELECT * FROM Bugs WHERE [order] = 123;
```

SQLite accepts all three styles, for compatibility with code written
against other engines. Delimited identifiers also let you use whitespace
or punctuation in a name (`"the order"`, `"the-order"`) — something no
programming-language identifier syntax normally allows.

- **Prefer renaming the column/table over delimiting it everywhere.**
  Delimiters have to be applied correctly in every single query that
  touches the identifier, forever — one missed spot reintroduces the
  error. A name that was never reserved in the first place (`order_num`,
  `display_order`) removes the problem instead of managing it.
- When reviewing schema or migration changes, treat any new column/table
  name as worth a quick check against the target database's reserved-word
  list, especially names that read as ordinary English verbs or nouns
  (`order`, `group`, `table`, `to`) — those are exactly the ones likely to
  already be taken by the language.

---

## 4. Review Checklist

- Does a syntax error point at a token that looks like a perfectly normal
  identifier? Check it against the database's reserved-word list before
  assuming the query's structure is wrong.
- Are any existing column/table names reserved words that only work today
  because they're always delimited? That's fragile — a missed delimiter
  anywhere reintroduces the failure.
- Before naming a new column or table, has the name been checked against
  reserved words for the target database brand and version — not just the
  brand in use today, since upgrades can add new reserved words later?
