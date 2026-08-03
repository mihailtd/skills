---
name: sql-antipatterns-sql-injection

description: Detects SQL Injection — building SQL statements by concatenating or interpolating unvalidated application data (user input, or even data read back from the database) into the query string before it's parsed — which lets an attacker change the statement's syntax, not just its data. Covers why escaping/parameters/stored procedures/frameworks each only cover part of the problem, and guides toward query parameters for values, whitelist-mapping for identifiers/keywords, and input filtering everywhere else.
---

# SQL Antipatterns — SQL Injection

This skill helps developers recognize every shape SQL injection takes —
not just the classic "drop the table" case — and guides them to the right
defense for each part of a dynamic SQL statement: query parameters for
values, whitelist mapping for identifiers/keywords, and filtering for
everything else. No single technique covers all cases; use the one that
fits each part of the query.

---

## 1. Recognize the Antipattern

- **The tell:** any SQL statement built by concatenating strings or
  interpolating application data directly into the query text, before the
  database parses it:

  ```python
  bugid = request.args.get("bugid")
  sql = f"SELECT * FROM Bugs WHERE bug_id = {bugid}"
  cursor.execute(sql)
  ```

- The core mechanism: injection works by changing the *syntax* of the
  statement before it's parsed, not just supplying an unexpected data
  value. Once attacker-controlled text becomes part of the SQL source
  itself, the attacker can append clauses, change conditions, or comment
  out the rest of the statement.
- **Assume it's present until proven otherwise.** SQL injection is common
  enough — and easy enough to introduce without noticing — that any
  application built with dynamic SQL should be assumed vulnerable unless
  it's had a dedicated review specifically for this class of bug.

---

## 2. Why It Breaks Down

- **Classic statement-splicing.** A value like `1234; DELETE FROM Bugs`
  interpolated into a query turns one statement into two.
- **Condition-widening, often subtler and more dangerous than splicing.**
  `userid = 123 OR TRUE` interpolated into `WHERE account_id = {userid}`
  turns a single-row update into a table-wide one:

  ```sql
  UPDATE Accounts SET password_hash = SHA2('xyzzy', 256)
  WHERE account_id = 123 OR TRUE;
  ```

  No syntax error, no obviously "hacky" string — just a condition that's
  always true.
- **It doesn't require malicious intent to cause damage.** Legitimate data
  containing a quote character (a project named "O'Hare", a customer's
  last name "O'Hara") breaks unescaped string interpolation by accident.
  The failure mode ranges from an embarrassing syntax error to, in the
  right combination, a real vulnerability.
- **Second-order injection: data that's "already in the database" isn't
  automatically safe.** Reading a value back out of your own database and
  interpolating it into a *second* query is exactly as risky as
  interpolating fresh user input — the value's origin doesn't matter, only
  whether it was ever validated/escaped for the context it's now being
  used in:

  ```python
  cursor.execute("SELECT last_name FROM Accounts WHERE account_id = 123")
  for row in cursor:
      # UNSAFE — row["last_name"] could be "O'Hara" or worse, and it was
      # never validated for use inside this second query
      sql2 = f"SELECT * FROM Bugs WHERE MATCH(description) AGAINST ('{row['last_name']}')"
      cursor.execute(sql2)
  ```

- **No single popular "fix" actually covers every case** — each of the
  following is necessary in its own scope, but none is sufficient alone:
  - *Escaping quote characters* only protects string literals, and only if
    you remember to apply it on every interpolation, every time. It does
    nothing for a value interpolated as a bare numeric literal (no quotes
    to escape) or as an identifier.
  - *Query parameters* are strong for literal values, but a parameter can
    only ever bind to a single value — never a list, a table name, a
    column name, or a keyword like `ASC`/`DESC` (see §4 for why, and what
    to use instead for those cases).
  - *Stored procedures* are just as vulnerable as application code if they
    build dynamic SQL internally via string concatenation — using a stored
    procedure doesn't itself confer safety.
  - *Data access frameworks/ORMs* only protect the parts of a query built
    through their safe APIs. Any raw/string-built SQL path they expose is
    exactly as unsafe as writing it by hand — the framework's presence
    doesn't retroactively make careless string-building safe.

---

## 3. Legitimate Uses

None. Unlike most antipatterns in this collection, there is no acceptable
trade-off here — this is a security defect, not a design compromise with
situational upsides. Every part of an application that builds SQL
dynamically needs a defense from §4; there's no context where skipping it
is the right call.

---

## 4. The Fix: Match the Defense to What's Being Made Dynamic

### Query parameters — for literal values

The strongest, simplest defense for anything that's a single data value.
The database parses the SQL statement (with placeholders) once; parameter
values are bound afterward and can never change the statement's syntax,
no matter what they contain:

```python
sql = """UPDATE Accounts SET password_hash = SHA2(%s, 256)
         WHERE account_id = %s"""
cursor.execute(sql, [password, userid])
```

Even a maliciously-crafted parameter value like `123 OR TRUE` is bound as
a single opaque string — at worst it fails to match any row, because
`account_id` is numeric and the string gets cast (or fails to cast)
according to normal rules, never interpreted as SQL syntax:

```sql
-- what actually runs, conceptually — the value never becomes code
UPDATE Accounts SET password_hash = SHA2('xyzzy', 256)
WHERE account_id = '123 OR TRUE';
```

**A prepared query's placeholders are not the same thing as automatic
quoting.** The database parses the query text once; after that, nothing
changes its syntax — parameter values are supplied separately and never
merged back into the SQL text form. This is exactly why they're safe, but
it also means: don't put a placeholder inside quotes (`'?'`) expecting the
quotes to matter — the placeholder itself already behaves like a string
literal token. `WHERE bug_id = '?'` compares `bug_id` to the *literal
string* `"?"`, not to a bound parameter, and will never match. Likewise,
to combine a parameter with wildcard characters for `LIKE`, don't write
`'%?%'` — either build the pattern in the query itself:

```sql
SELECT * FROM Bugs WHERE summary LIKE CONCAT('%', ?, '%')
```

or format the full pattern in application code and pass it as one
parameter, with the placeholder unquoted in the query:

```python
query = "SELECT * FROM Bugs WHERE summary LIKE %s"
cursor.execute(query, [f"%{word}%"])
```

**What parameters can't do** — because a parameter is always exactly one
opaque value, never part of the statement's structure:

- **A list of values isn't one parameter.** `bug_id IN (%s)` bound to
  `"1234,3456,5678"` compares against the single string
  `'1234,3456,5678'`, not three integers. Build one placeholder per list
  item instead:

  ```python
  placeholders = ",".join(["%s"] * len(bug_list))
  sql = f"SELECT * FROM Bugs WHERE bug_id IN ({placeholders})"
  cursor.execute(sql, bug_list)
  ```

- **Table names, column names, and SQL keywords can't be parameters at
  all** — `FROM %s` bound to `"Bugs"` produces `FROM 'Bugs'`, a syntax
  error (or a no-op string constant, for something like `ORDER BY %s`),
  never the identifier you meant. See the whitelist-mapping technique
  below for these.

Despite the extra setup, parameterized queries are usually *easier* to
write correctly than manual string-building — every escaped quote and
concatenated fragment is a chance to miscount a delimiter; a parameter
placeholder has no quoting to get wrong.

### Whitelist mapping — for identifiers and keywords

When the dynamic part of a query is a column name, table name, or keyword
(sort direction, etc.) — none of which a parameter can represent — don't
interpolate the raw input at all. Map the user's choice through an
explicit, hardcoded lookup, and use only the mapped value in the query:

```python
sortorders = {"status": "status", "date": "date_reported"}
directions = {"up": "ASC", "down": "DESC"}

sortorder = sortorders.get(request.args.get("order"), "bug_id")
direction = directions.get(request.args.get("dir"), "ASC")

sql = f"SELECT * FROM Bugs ORDER BY {sortorder} {direction}"
cursor.execute(sql)
```

This works because the only strings that ever reach the query are ones
you wrote yourself — user input only ever selects a key, never supplies
the value used in the SQL text. It also gives you free input validation
(anything not a recognized key falls through to the default) and decouples
your public-facing option names from your actual schema's column names.

### Filtering — for everything else

When neither of the above applies directly, constrain the input before
it's used:

- **Cast to the expected type** for simple cases: `int(request.args.get("bugid"))`
  guarantees the value can only ever be an integer by the time it's
  interpolated.
- **Match against a strict allow-list pattern** (e.g. `^\w+$` for a
  supposed identifier) and reject or replace anything that doesn't match,
  rather than trying to strip out "bad" characters after the fact.
- **Treat every value pulled from your own database the same as fresh
  external input**, if it's ever going to be used inside another dynamic
  SQL statement — see the second-order injection case in §2.

### Escaping — as a fallback, not a first choice

When parameters genuinely can't be used (e.g. a rare case where the query
optimizer picks a bad plan for a parameterized value — see the book's note
on `EXPLAIN`-driven investigation in [[sql-antipatterns-index-shotgun]]),
escape the value with your database driver's dedicated, tested escaping
function — never a hand-rolled one, and never a function meant for a
different context (HTML-entity encoding is not SQL escaping). Escaping
only protects quoted string literals; it does nothing for identifiers,
keywords, or values interpolated as bare numeric literals.

---

## 5. Review Process

A focused code review is the most reliable way to catch injection flaws
before they ship:

1. Find every SQL statement built via string concatenation, formatting, or
   interpolation of a variable.
2. For each one, trace where the dynamic content originates — user input,
   files, environment variables, third-party/web-service responses, or
   even a value previously read from your own database (second-order
   case).
3. Treat every one of those sources as untrusted by default.
4. Confirm each dynamic fragment is handled by the matching defense from
   §4 — parameter, whitelist mapping, or filter — not just "some SQL
   library is involved so it's probably fine."
5. Don't skip stored procedures or other server-side code paths — dynamic
   SQL built there is exactly as exposed as application-side code.

Ongoing query logging or APM tooling that surfaces unexpected query shapes
can help catch exploitation attempts (or residual vulnerabilities) in
production, as a complement to review — not a substitute for it.

---

## 6. Review Checklist

- Is any SQL statement built by concatenating or interpolating a variable
  into the query string before execution?
- For each dynamic literal value, is it passed as a bound query parameter
  — not interpolated, and not merely escaped as the only defense?
- For each dynamic identifier or keyword (table/column name, sort
  direction, etc.), is it resolved through an explicit whitelist mapping,
  never interpolated directly?
- Are values read back from the database ever interpolated into a
  *second* SQL statement without being treated as untrusted input?
- Do any stored procedures build dynamic SQL internally via string
  concatenation of their input parameters?
- Are query-parameter placeholders ever mistakenly wrapped in quotes in
  the SQL text?
- Has this code path had a dedicated review specifically for injection —
  not just a general code review — before shipping?

---

## Related guidance

PostgreSQL-specific remedy:

- database-postgresql-secure-search-path
