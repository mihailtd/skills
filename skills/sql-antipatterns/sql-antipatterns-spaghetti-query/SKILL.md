---
name: sql-antipatterns-spaghetti-query

description: Detects the Spaghetti Query antipattern — forcing a complex, multi-part reporting task into a single monolithic SQL query — which silently produces unintended Cartesian products (counts multiplied together when two joined tables share no real relationship) and is hard to write, debug, and extend, and guides toward splitting into several simpler queries.
---

# SQL Antipatterns — Spaghetti Query

This skill helps developers recognize when "can I do this in a single
query?" is being treated as a goal in itself, and guides them to split a
complex reporting task into several simple, correct queries instead of one
query that's hard to verify and often silently wrong.

---

## 1. Recognize the Antipattern

- **The tell:** a request for several loosely related statistics gets
  answered with one query joining every table involved, on the assumption
  that fewer queries is automatically better:

  ```sql
  SELECT COUNT(bp.product_id) AS how_many_products,
         COUNT(dev.account_id) AS how_many_developers,
         COUNT(b.bug_id)/COUNT(dev.account_id) AS avg_bugs_per_developer,
         COUNT(cust.account_id) AS how_many_customers
  FROM Bugs b
  JOIN BugsProducts bp ON (b.bug_id = bp.bug_id)
  JOIN Accounts dev    ON (b.assigned_to = dev.account_id)
  JOIN Accounts cust   ON (b.reported_by = cust.account_id)
  WHERE cust.email NOT LIKE '%@example.com'
  GROUP BY bp.product_id;
  ```

  This looks efficient and "elegant" but is a strong candidate for
  producing quietly wrong numbers — see §2.
- **Listen for these phrases**:
  - "Why are my sums/counts impossibly large?" — the signature symptom of
    an unintended Cartesian product multiplying row counts together.
  - "I've been fighting this one query all day." — a sign the task should
    have been broken into smaller pieces, the same instinct you'd apply in
    any other programming language when a function gets unmanageable.
  - "We can't add anything to this report without a huge SQL rewrite." —
    a monolithic query has become a maintenance liability.
  - "Try adding another `DISTINCT`." — patching over an unintended
    Cartesian product by deduplicating its output, instead of removing the
    join that caused it. This masks the symptom, not the cause, and costs
    extra work generating and then discarding the duplicate rows.

---

## 2. Why It Breaks Down

- **Unintended Cartesian products silently multiply counts.** If two
  tables are joined into the same query but have no real relationship to
  each other (both only relate to a third table, not to each other), every
  row on one side pairs with every row on the other:

  ```sql
  SELECT b.bug_id, COUNT(t.tag) AS count_tags, COUNT(bp.product_id) AS count_products
  FROM Bugs b
  LEFT OUTER JOIN Tags t          ON (t.bug_id = b.bug_id)
  LEFT OUTER JOIN BugsProducts bp ON (bp.bug_id = b.bug_id)
  WHERE b.bug_id = 1234
  GROUP BY b.bug_id;
  -- both counts come back as 8, even though there are only 4 tags and 2 products
  ```

  `Tags` and `BugsProducts` are both joined to `Bugs`, but not to each
  other — so each of the 4 tag rows pairs with each of the 2 product rows,
  producing 8 combined rows, and both `COUNT()`s report 8. The numbers
  look plausible enough to ship unnoticed unless you already know the
  right answer.
- **`COUNT(DISTINCT ...)` hides the symptom, not the cause.** Deduplicating
  gets the right number in this specific case, but doesn't work in general
  — if the data legitimately has repeated values that should be counted
  more than once, `DISTINCT` silently drops the ones you needed. Removing
  the accidental Cartesian product (not compensating for it) is the actual
  fix.
- **Complex queries are hard to write, debug, and extend.** A query
  combining many unrelated calculations costs real time to get right the
  first time, and every future report tweak means re-untangling all of it
  again. Whoever wrote it is implicitly signed up to maintain it
  indefinitely, including after they've moved to a different project.
- **The "fewer queries is always faster" assumption doesn't hold.** It's
  only true when the queries being compared are of similar complexity. A
  single monster query with many joins, correlated subqueries, and
  aggregation can cost the optimizer far more than several simple queries
  combined — "one query" is not a proxy for "efficient."

---

## 3. Legitimate Uses

- Some reporting/BI tools or visual components assume a single query as
  their data source and don't give you the option to post-process multiple
  result sets in application code. If you're constrained to that kind of
  tool, a more complex query may be a real requirement, not a choice — but
  where the reporting need is too complex for one query to express safely,
  producing multiple separate reports is a better trade than forcing an
  unsafe single query, and worth pushing back on the requirement for.
- If you need results from multiple logical calculations combined and
  sorted together as one ordered result, doing that sort in SQL is usually
  more efficient (and less code) than merging and sorting several result
  sets in application code — that's a legitimate reason to keep things in
  one query, distinct from cramming unrelated aggregates together.
- `CROSS JOIN` (an explicit, deliberate Cartesian product) is a real,
  useful tool — e.g. generating a sequence of numbers by cross-joining a
  small digits table with itself:

  ```sql
  SELECT 10*digit10.num + digit1.num AS num
  FROM integers AS digit1
  CROSS JOIN integers AS digit10;  -- generates 0..99
  ```

  The antipattern is an *accidental* Cartesian product from a missing join
  condition, not the deliberate, well-understood use of `CROSS JOIN`.

---

## 4. The Fix: Divide and Conquer

Prefer the simpler of two queries that produce the same result — this is
just the law of parsimony applied to SQL. When a Cartesian product shows
up because two tables genuinely have no join condition between them,
that's the schema telling you the calculations don't belong in one query:

```sql
SELECT b.bug_id, COUNT(t.tag) AS count_tags
FROM Bugs b LEFT OUTER JOIN Tags t ON (b.bug_id = t.bug_id)
WHERE b.bug_id = 1234 GROUP BY b.bug_id;

SELECT b.bug_id, COUNT(bp.product_id) AS count_products
FROM Bugs b LEFT OUTER JOIN BugsProducts bp ON (b.bug_id = bp.bug_id)
WHERE b.bug_id = 1234 GROUP BY b.bug_id;
```

Splitting a "give me four unrelated statistics" request into four small,
independently-correct queries is a better trade than one intricate query
that's hard to verify:

```sql
SELECT COUNT(*) AS how_many_products FROM Products;

SELECT COUNT(DISTINCT assigned_to) AS how_many_developers
FROM Bugs WHERE status = 'FIXED';

SELECT AVG(bugs_per_developer) AS average_bugs_per_developer
FROM (SELECT dev.account_id, COUNT(*) AS bugs_per_developer
      FROM Bugs b JOIN Accounts dev ON (b.assigned_to = dev.account_id)
      WHERE b.status = 'FIXED'
      GROUP BY dev.account_id) t;

SELECT COUNT(*) AS how_many_customer_bugs
FROM Bugs b JOIN Accounts cust ON (b.reported_by = cust.account_id)
WHERE b.status = 'FIXED' AND cust.email NOT LIKE '%@example.com';
```

This isn't a compromise — it has real advantages over the monolithic
version: each query is independently easy to verify against expected
results; adding a new statistic to the report means adding one more simple
query instead of re-threading an already-complex one; the optimizer
generally handles several straightforward queries more reliably than one
elaborate one; and it's far easier to explain in a code review or hand off
to a teammate.

### When you need many similar statements, generate them with SQL

If splitting a task produces many structurally-similar statements (e.g. a
per-row `UPDATE` that can't be expressed as one set-based statement),
write a query that generates the SQL text, rather than hand-writing dozens
of near-identical statements or forcing them into one unsafe combined
statement:

```sql
SELECT CONCAT('UPDATE Inventory SET last_used = ''', MAX(u.usage_date), '''',
              ' WHERE inventory_id = ', u.inventory_id, ';') AS update_statement
FROM ComputerUsage u
GROUP BY u.inventory_id;
```

The output is a ready-to-run script of individual `UPDATE` statements —
often faster to produce and easier to verify (spot-check a few generated
statements) than a single clever statement engineered to handle every row
differently in one pass.

---

## 5. Review Checklist

- Do two joined tables in a query actually share a real relationship with
  each other, or are they only both related to a third table (a Cartesian
  product waiting to happen)?
- Do reported counts/sums look suspiciously large, round, or identical
  across unrelated metrics in the same query?
- Is `DISTINCT` being used to paper over duplicated rows from a join,
  rather than fixing the join that produced them?
- Would splitting one complex report query into several simple ones make
  each one independently checkable against a known-correct expected value?
- Is a single query being kept complex only because a reporting tool
  requires one data source — or is that constraint being assumed rather
  than confirmed?
- For a many-similar-statements problem, would generating the SQL
  statements (via a query or script) be safer and faster than either
  hand-writing them all or forcing them into one combined statement?

---

## Related guidance

PostgreSQL-specific remedy:

- database-postgresql-cte-complex-reporting
