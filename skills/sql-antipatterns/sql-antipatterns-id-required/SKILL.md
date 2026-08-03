---
name: sql-antipatterns-id-required

description: Detects the ID Required antipattern — treating "primary key" as always meaning an auto-increment integer column named id, which produces redundant keys, unenforced duplicate rows in intersection tables, and ambiguous joins — and guides toward primary keys tailored to what actually identifies each row.
---

# SQL Antipatterns — ID Required

This skill helps developers stop conflating "primary key" with "a column
named `id`," and choose the key — natural, compound, or pseudokey — that
actually fits each table.

---

## 1. Recognize the Antipattern

- **The tell:** every table gets a `SERIAL`/auto-increment column named
  `id` as primary key, by reflex, regardless of what would actually
  identify a row uniquely.

  ```sql
  CREATE TABLE Bugs (
      id          SERIAL PRIMARY KEY,
      description VARCHAR(1000)
  );
  ```

- **Listen for these phrases**:
  - "I don't think I need a primary key in this table." — conflates
    *primary key* (a constraint every table needs) with *pseudokey*
    (one specific way to satisfy it).
  - "How did I get duplicate many-to-many associations?" — a strong signal
    the intersection table has an `id` pseudokey but no `UNIQUE`/`PRIMARY
    KEY` constraint over the actual foreign key pair.
  - "I don't want a lookup table because I'd have to join every time." —
    a misunderstanding of normalization, not actually about pseudokeys.

---

## 2. Why It Breaks Down

- **Redundant keys.** If a table already has a natural unique identifier
  (a `bug_id` mnemonic string, a natural key), bolting on an `id` column
  anyway just gives every row two identities to keep straight for no
  benefit.
- **Silently permits duplicate rows in intersection tables.** This is the
  most damaging case. An `id` pseudokey satisfies "has a primary key" but
  does nothing to enforce that `(article_id, tag_id)` stays unique:

  ```sql
  CREATE TABLE ArticleTags (
      id         SERIAL PRIMARY KEY,
      article_id BIGINT UNSIGNED NOT NULL,
      tag_id     BIGINT UNSIGNED NOT NULL,
      FOREIGN KEY (article_id) REFERENCES Articles(id),
      FOREIGN KEY (tag_id)     REFERENCES Tags(id)
  );

  -- nothing stops this from happening three times
  INSERT INTO ArticleTags (article_id, tag_id) VALUES (1234, 327);
  ```

  Aggregate queries (`COUNT(*) ... GROUP BY tag_id`) then silently
  overcount, with no error anywhere — the bug surfaces only as "the
  numbers don't match what I expect."
- **Obscures meaning in joins.** A column named `id` in every table means
  a join has to alias every occurrence (`b.id AS bug_id, a.id AS
  account_id`) just to tell results apart — especially painful when the
  result becomes a JSON object or associative array, where a duplicate key
  name silently overwrites the earlier value.
- **Blocks the concise `USING` join syntax.** `JOIN BugsProducts USING
  (bug_id)` only works when both sides name the shared column the same
  way — impossible if the foreign key can never be named the same as the
  `id` it references. You're forced into verbose `ON` clauses everywhere.

---

## 3. Legitimate Uses

- If your ORM/framework assumes convention-over-configuration (an integer
  `id` primary key on every table), following that convention to keep the
  framework's other features working is a reasonable trade — most support
  overriding the primary key name when you do need to (e.g. Rails'
  `set_primary_key`).
- A pseudokey is the right call when the natural key is real but
  impractical to index (e.g. a full filesystem path) or when a chain of
  subordinate tables referencing a long compound key would otherwise force
  every dependent table to carry an ever-growing compound foreign key.
- Auto-incrementing sequences are the correct way to avoid the race
  condition in `SELECT MAX(id) + 1` under concurrent inserts — sequences
  allocate outside transaction scope, so two concurrent clients can never
  be handed the same value.

---

## 4. The Fix: Tailor the Key to the Table

- **A primary key is a constraint, not a data type or a naming
  convention.** It can be declared on any column, or set of columns, that
  can be indexed and made `NOT NULL` — an auto-increment integer is one
  option among several, not the definition of "primary key."
- **Name it for what it identifies**, and reuse that name in referencing
  foreign keys where practical, so joins can use `USING (bug_id)` and
  result columns are self-explanatory without aliasing:

  ```sql
  CREATE TABLE Bugs (
      bug_id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
      description VARCHAR(1000)
  );
  ```

- **Use a natural key when one genuinely, uniquely identifies the row**
  (and is stable — be honest about whether the business will actually
  respect that stability over time).
- **Use a compound key for intersection tables** — this is what actually
  prevents the duplicate-row bug, not an incidental `UNIQUE` constraint
  bolted on beside an unnecessary `id`:

  ```sql
  CREATE TABLE BugsProducts (
      bug_id     BIGINT UNSIGNED NOT NULL,
      product_id BIGINT UNSIGNED NOT NULL,
      PRIMARY KEY (bug_id, product_id),
      FOREIGN KEY (bug_id)     REFERENCES Bugs(bug_id),
      FOREIGN KEY (product_id) REFERENCES Products(product_id)
  );

  INSERT INTO BugsProducts VALUES (1234, 1);
  INSERT INTO BugsProducts VALUES (1234, 1); -- error: duplicate entry
  ```

  Don't avoid compound keys just because comparisons and foreign keys
  referencing them need to list every column — that's a minor typing cost,
  not a real design problem.
- **Every table still needs a primary key of some kind.** Skipping it
  doesn't remove the need to prevent duplicates and address rows
  individually — it just turns that into manual, error-prone work (`SELECT
  x FROM t GROUP BY x HAVING COUNT(*) > 1` run as a recurring chore instead
  of enforced by the schema), and it degrades row-based replication to
  full table scans per update in engines like MySQL.

---

## 5. Are You Going to Run Out of ID Values?

A frequent worry once auto-increment keys are in play: will a busy table
exhaust its integer range? Do the math instead of guessing:

- `INT`/`SERIAL` (signed 32-bit) maxes out at 2,147,483,647. At 10,000
  inserts/minute that's exhausted in about 149 days — genuinely too small
  for a busy table.
- Switching to unsigned only doubles the range (to ~298 days at the same
  rate) — still not enough headroom for most production tables.
- `BIGINT` (signed 64-bit) maxes out at 9,223,372,036,854,775,807 — the
  square of the 32-bit range, not merely double it. At the same 10,000/min
  rate that's roughly 1.75 billion years. If a table increments a `BIGINT`
  key by 1 per row starting at 1, running out isn't a real operational
  concern.

Do this estimate — rows-per-minute ÷ into the max value of the candidate
type — before assuming either "`INT` is obviously fine" or "we need some
exotic key scheme to avoid running out."

---

## 6. Review Checklist

- Does every table have an `id` pseudokey purely by convention, even ones
  with an obvious natural or compound key?
- Does an intersection table's *only* uniqueness constraint sit on a
  meaningless pseudokey, leaving the actual foreign key pair unconstrained?
- Do joins need column aliasing just to disambiguate two tables' `id`
  columns, where a descriptive name would've made the alias unnecessary?
- Is a table's auto-increment column sized (`INT` vs `BIGINT`) based on an
  actual insert-rate estimate, not a guess?
