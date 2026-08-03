---
name: sql-antipatterns-see-no-evil

description: Detects the See No Evil antipattern — ignoring database API error/exception return values, and debugging by staring at the application code that builds SQL instead of the actual generated SQL string — which lets failures surface as blank screens or baffling crashes far from their real cause. Guides toward checking every database call's outcome and always inspecting the literal SQL text (and its error message) when debugging a query.
---

# SQL Antipatterns — See No Evil

This skill helps developers recognize two related habits that hide real
failures: skipping error/exception handling on database API calls, and
debugging a generated SQL query by reading the code that builds it instead
of the SQL text itself. Both trade a small amount of code/effort now for
much more debugging pain later, at a worse time and further from the
actual cause.

---

## 1. Recognize the Antipattern

- **The tell, ignored errors:** database API calls (connect, execute, ...)
  whose return value or thrown exception is never checked:

  ```python
  cnx = mysql.connector.connect(user='scottt', database='test')  # typo unnoticed
  cursor = cnx.cursor()
  query = '''SELCET bug_id, summary FROM Bugs WHERE assigned_to = %s'''  # typo unnoticed
  cursor.execute(query, parameters)
  for row in cursor:
      print(row)
  ```

  Every one of these calls can fail — a bad password, an unreachable
  server, a typo in the SQL — and none of it is visible in the code path
  above; the failure surfaces later, disconnected from its cause, often as
  a blank screen or an opaque crash the user reports back to you.
- **The tell, debugging the builder instead of the built query:** staring
  at the application code (string concatenation, conditionals, variable
  interpolation) that *produces* an SQL string, instead of looking at the
  actual resulting SQL text:

  ```python
  query = '''SELECT * FROM Bugs'''
  if bug_id > 0:
      query = query + '''WHERE bug_id = %s'''  # missing separator
  cursor.execute(query, parameters)
  ```

  The bug (`SELECT * FROM BugsWHERE bug_id = 1234` — no space before
  `WHERE`) is invisible in the code, which looks perfectly reasonable, but
  obvious the instant you look at the concatenated string itself.
- **Listen for these phrases**:
  - "My program crashes after I query the database." — often because a
    failed query's result was used as if it had succeeded (calling a
    method on what's actually null/None, iterating a cursor that never
    ran).
  - "Can you help me find my SQL error? Here's my [application] code..." —
    the reflex to share the code that *builds* the query, rather than the
    query itself, when the query is what actually needs inspecting.
  - "I don't bother cluttering up my code with error handling." — treating
    error handling as optional ceremony rather than a substantial, normal
    part of robust code (some estimates put up to half of a robust
    application's code in error detection, classification, reporting, and
    compensation).

---

## 2. Why It Breaks Down

- **An unhandled connection failure or query error doesn't stay
  contained** — it propagates until something breaks in a way the user
  actually sees: an unhandled exception, a blank page, a crash with a
  stack trace that means nothing to them (and possibly leaks internals to
  them, which is its own concern). By the time it's reported to you,
  you're working backward from a vague symptom instead of a precise
  location.
- **"That's not supposed to happen anyway" is not a reason to skip
  checking.** Every one of these calls has a real, mundane way to fail:
  wrong credentials, unreachable host, a typo in SQL syntax, a
  misspelled column. None of that is exotic — it's the normal cost of
  writing software that talks to a network service.
- **Debugging the query-building code instead of the query itself is
  solving the wrong problem.** Whitespace/concatenation bugs, unbalanced
  parentheses, and misspelled keywords are usually invisible in the
  *logic* that assembles a string, but immediately visible once you look
  at the actual resulting SQL text. Staring at string-concatenation logic
  to find a syntax bug is like trying to solve a jigsaw puzzle without
  looking at the picture on the box.

---

## 3. Legitimate Uses

- Skipping error handling is reasonable when there's genuinely nothing
  actionable to do with the error — e.g. checking the return status of a
  connection `close()` call right before the process exits anyway, where
  the resources get reclaimed regardless of that call's outcome.
- In languages with exceptions, it's fine for a given piece of code to let
  an exception propagate rather than catch it locally, *as long as
  something up the call stack is actually responsible for handling it*.
  Not handling an error is different from no one handling it — the
  antipattern is the latter.

---

## 4. The Fix: Check Every Call, Inspect the Actual SQL

### Check return values and exceptions on every database API call

```python
try:
    cnx = mysql.connector.connect(user='scott', database='test')
except mysql.connector.Error as err:
    if err.errno == errorcode.ER_ACCESS_DENIED_ERROR:
        print("Something is wrong with your user name or password")
    elif err.errno == errorcode.ER_BAD_DB_ERROR:
        print("Database does not exist")
    else:
        print(err)

cursor = cnx.cursor()
try:
    cursor.execute(query, parameters)
except mysql.connector.Error as err:
    print(err)
```

Log the real exception/error for developers to inspect, and show a
friendlier message to the user — don't let raw database errors leak to
end users, but don't discard them either. This is the same "fail early,
fail loud, at the actual point of the problem" principle behind
[[sql-antipatterns-keyless-entry]] and [[sql-antipatterns-implicit-columns]]
— a precise error at the point of failure is far cheaper to act on than a
symptom discovered much later, far away.

### Debug by inspecting the actual SQL text, not the code that built it

- **Build the query into a variable first**, rather than constructing it
  inline as an argument to the execute call — this gives you a concrete
  value you can print, log, or inspect before it's ever sent to the
  database.
- **Send it somewhere you can actually read it**: a log file, your IDE's
  debugger console, a dedicated diagnostic output — not mixed into the
  application's normal output.
- **Never emit the SQL query into user-facing output**, including HTML
  comments in a web page — page source is visible to any user, and a
  visible query string hands an attacker real structural knowledge of
  your schema for free (this is exactly the reconnaissance
  [[sql-antipatterns-sql-injection]] depends on).
- **If you use an ORM**, remember the SQL it generates isn't visible in
  your source code at all — find and enable its query-logging feature
  (most support logging generated SQL to a file or the application
  server's error log) rather than trying to debug by reading ORM calls.
- **Database servers themselves usually offer query logging** independent
  of the application — if application-side logging isn't available or
  isn't enough, check what your database brand provides for observing
  queries as they actually execute server-side.

### Reading SQL syntax error messages

Most SQL engines report where in the query text the parser got confused —
learning to read that position precisely turns a vague "syntax error" into
a direct pointer at the fix (this is the same skill that resolves a
reserved-word collision like the one in
[[sql-antipatterns-reserved-words]] — the error message points right at
the offending token):

```
SELECT * FROM Bugs ORDER date_reported;
ERROR 1064 (42000): ... right syntax to use near 'date_reported' at line 1
-- missing BY keyword between ORDER and the column name
```

```
SELECT * FROM Bugs WHERE status = 'NEW' AND WHERE assigned_to = 123;
ERROR 1064 (42000): ... right syntax to use near 'WHERE assigned_to = 1' ...
-- WHERE should appear once; the second one shouldn't be there
```

```
SELECT * FROM Bugs WHERE (status = 'NEW';
ERROR 1064 (42000): ... right syntax to use near '' at line 1
-- an empty quoted excerpt means the parser ran out of query at the
-- error point — here, an unclosed parenthesis
```

Error message quality and clarity vary a lot by database brand (some are
notoriously unhelpful) — but even a terse message is usually still telling
you a precise location in the query text; take the time to read it against
the actual query string rather than guessing.

---

## 5. Review Checklist

- Does any database API call (connect, prepare, execute, fetch) proceed
  without checking whether it succeeded?
- When a query fails, does the failure surface with enough information to
  act on (logged exception with query context), or does it propagate as a
  generic crash/blank output far from the actual cause?
- When debugging a "my query doesn't work" report, is the actual generated
  SQL text being inspected — or only the application code that assembles
  it?
- Is generated SQL logged somewhere reviewable (app log, IDE console,
  ORM query log, database server log) rather than never being visible at
  all?
- Does any user-facing output (including HTML comments) ever leak the
  literal SQL query text?
- When reading a syntax error message, has the reported position actually
  been checked against the real query string, rather than guessing at the
  cause from the code that built it?
