---
name: sql-antipatterns-polymorphic-associations

description: Detects the Polymorphic Associations antipattern — a "dual-purpose" foreign key column paired with a string column naming which parent table it points to, so no real FOREIGN KEY constraint can exist — and guides toward reversing the reference with per-parent intersection tables or a common supertable.
---

# SQL Antipatterns — Polymorphic Associations

This skill helps developers recognize when a child table is trying to
reference "whichever parent table applies to this row" via a foreign-key
column plus a sibling string column naming the target table, and guides
them to a design where every reference actually names one table and can
be enforced.

---

## 1. Recognize the Antipattern

- **The tell:** a foreign-key-shaped column is paired with a second column
  that stores the *name of the table it points to* on a per-row basis:

  ```sql
  CREATE TABLE Comments (
      comment_id  SERIAL PRIMARY KEY,
      issue_type  VARCHAR(20),  -- "Bugs" or "FeatureRequests"
      issue_id    BIGINT UNSIGNED NOT NULL,
      comment     TEXT,
      FOREIGN KEY (author) REFERENCES Accounts(account_id)
      -- note: no FOREIGN KEY on issue_id — it can't name one table
  );
  ```

  Also called a "promiscuous association." Not limited to modeling
  subtypes of one concept (`Bugs`/`FeatureRequests`) — the same shape
  shows up between unrelated tables too, e.g. an `Addresses` table with a
  `parent` column of `"Users"` or `"Orders"`.
- **Listen for these phrases**:
  - "This tagging schema lets you associate a tag with *any* resource in
    the database." — the same suspicious unlimited-flexibility claim seen
    with [[sql-antipatterns-entity-attribute-value]]; it usually means a
    rule is being broken to get there.
  - "You can't declare foreign keys in our design." — a hard stop: no
    referential integrity means the schema itself is the problem.
  - "What's the `entity_type`/`issue_type` column for? Oh, that tells you
    which table this other column points to." — a foreign key must
    reference the same table on every row; a column that needs a sibling
    column to say *which* table it means isn't one.
  - Frameworks bake this in by name — Rails' `belongs_to ..., polymorphic:
    true`, Hibernate's inheritance mapping strategies — so "we use Active
    Record polymorphic associations" is a direct signal too.

---

## 2. Why It Breaks Down

- **No foreign key can be declared.** `FOREIGN KEY (issue_id) REFERENCES
  Bugs(issue_id) OR FeatureRequests(issue_id)` isn't valid SQL — a
  constraint must name exactly one table. `issue_id` and `issue_type`
  therefore have zero enforcement: nothing stops `issue_id` from pointing
  at a row that doesn't exist, and nothing validates that `issue_type`
  names a real table at all.
- **No entity-relationship diagram convention exists for it** — teams end
  up inventing their own notation, which is itself a symptom that the
  relationship isn't one SQL (or ER modeling) actually has a shape for.
- **A join can't switch tables per row.** SQL requires every table in a
  query to be named up front; it can't join to `Bugs` for some rows and
  `FeatureRequests` for others within the same join. The workaround is an
  outer join to *every* possible parent, filtered by the type column:

  ```sql
  SELECT *
  FROM Comments AS c
  LEFT OUTER JOIN Bugs AS b
      ON b.issue_id = c.issue_id AND c.issue_type = 'Bugs'
  LEFT OUTER JOIN FeatureRequests AS f
      ON f.issue_id = c.issue_id AND c.issue_type = 'FeatureRequests';
  ```

  Every added parent type means editing every query that does this.
- **Usage metadata propagates like weeds.** Once one parent type needs a
  qualifier (e.g. an address's `"billing"` vs `"shipping"` role for
  `Users` but a different meaning for `Orders`), you end up bolting on a
  separate `<parent>_usage` column per parent type, each mostly null.
- **This is the same root problem as EAV**, just for table names instead
  of column names: a piece of *metadata* (which table/column something
  belongs to) is being stored as a *data* value in a string column, so the
  database has no way to reason about or enforce it.

---

## 3. Legitimate Uses

- You should generally avoid this antipattern outright — prefer real
  constraints over application code standing in for referential integrity.
- It may be effectively unavoidable if you're building on an ORM/framework
  that implements polymorphic associations natively (Rails Active Record,
  Hibernate inheritance mappings). A mature, reputable framework has at
  least encapsulated the application logic that maintains the association
  correctly, which is a meaningfully smaller risk than reimplementing the
  same pattern by hand from scratch.

---

## 4. The Fix: Reverse the Reference, or Add a Common Supertable

The core insight: Polymorphic Associations point the foreign key the wrong
direction. Fix it by either having each parent point at the child instead,
or by giving the parents a real shared ancestor to point at.

### a. Reverse the reference with per-parent intersection tables

Instead of one child table guessing which parent it belongs to, create one
intersection table per parent, each with a normal, single-table foreign key
to both sides:

```sql
CREATE TABLE BugsComments (
    issue_id   BIGINT UNSIGNED NOT NULL,
    comment_id BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (issue_id, comment_id),
    FOREIGN KEY (issue_id)   REFERENCES Bugs(issue_id),
    FOREIGN KEY (comment_id) REFERENCES Comments(comment_id)
);

CREATE TABLE FeaturesComments (
    issue_id   BIGINT UNSIGNED NOT NULL,
    comment_id BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (issue_id, comment_id),
    FOREIGN KEY (issue_id)   REFERENCES FeatureRequests(issue_id),
    FOREIGN KEY (comment_id) REFERENCES Comments(comment_id)
);
```

This drops `Comments.issue_type` entirely — metadata (which parent a
comment belongs to) is now expressed by which intersection table has the
row, which the schema itself enforces.

- **Constrain "one parent per child" explicitly.** An intersection table
  naturally allows many-to-many, but a comment should belong to exactly
  one issue. Add `UNIQUE (comment_id)` on each intersection table so a
  given comment can appear at most once *within* that table:

  ```sql
  CREATE TABLE BugsComments (
      issue_id   BIGINT UNSIGNED NOT NULL,
      comment_id BIGINT UNSIGNED NOT NULL,
      UNIQUE KEY (comment_id),
      PRIMARY KEY (issue_id, comment_id),
      FOREIGN KEY (issue_id)   REFERENCES Bugs(issue_id),
      FOREIGN KEY (comment_id) REFERENCES Comments(comment_id)
  );
  ```

  Note this only prevents duplication *within* one intersection table —
  it can't stop the same comment from also having a row in
  `FeaturesComments`. Preventing "belongs to both a bug and a feature"
  is still an application-level rule to enforce.
- **Querying one direction is direct:**

  ```sql
  SELECT * FROM BugsComments AS b
  JOIN Comments AS c USING (comment_id)
  WHERE b.issue_id = 1234;
  ```

  The other direction still needs an outer join per possible parent — no
  worse than the antipattern's query, but now backed by real referential
  integrity:

  ```sql
  SELECT *
  FROM Comments AS c
  LEFT OUTER JOIN (BugsComments JOIN Bugs AS b USING (issue_id))
      USING (comment_id)
  LEFT OUTER JOIN (FeaturesComments JOIN FeatureRequests AS f USING (issue_id))
      USING (comment_id)
  WHERE c.comment_id = 9876;
  ```

- **Merge results across parent tables** with `UNION` (list `NULL`
  placeholders for columns unique to each parent, matched in the same
  order) or `COALESCE()` across the outer-joined columns, when the caller
  needs one row regardless of which parent matched. Both forms are
  complex enough to be worth wrapping in a view.

### b. Give the parents a common supertable

Model this like Class Table Inheritance (see
[[sql-antipatterns-entity-attribute-value]]): a base table generates the
shared primary key, and each parent's primary key is also a foreign key
back to it. The child table then references the base table directly — one
real foreign key, no type column needed.

```sql
CREATE TABLE Issues (
    issue_id SERIAL PRIMARY KEY
);
CREATE TABLE Bugs (
    issue_id BIGINT UNSIGNED PRIMARY KEY,
    FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
);
CREATE TABLE FeatureRequests (
    issue_id BIGINT UNSIGNED PRIMARY KEY,
    FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
);
CREATE TABLE Comments (
    comment_id SERIAL PRIMARY KEY,
    issue_id   BIGINT UNSIGNED NOT NULL,
    comment    TEXT,
    FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
);
```

Because `Bugs.issue_id` and `FeatureRequests.issue_id` share the same
value space as `Issues.issue_id`, you can join `Comments` straight to
`Bugs` or `FeatureRequests` without touching `Issues` at all, unless
`Issues` itself carries shared attributes:

```sql
SELECT * FROM Bugs AS b
JOIN Comments AS c USING (issue_id)
WHERE b.issue_id = 1234;
```

Prefer this option when the parent types are genuinely subtypes of one
concept (as `Bugs`/`FeatureRequests` are, both "issues") — it's the
cleaner fit. Reach for reversed intersection tables instead when the
parents are unrelated entities that only incidentally share a child (like
`Users`/`Orders` both having `Addresses`), where inventing a shared
supertype would be artificial.

---

## 5. Review Checklist

- Is there a foreign-key-shaped column paired with a sibling column that
  stores which table it references?
- Is there a comment/table column explicitly *missing* a `FOREIGN KEY`
  declaration because "it depends on the row"?
- Do queries use an outer join to every possible parent table, filtered by
  a type column, to simulate a single join?
- Are per-parent "usage" or qualifier columns multiplying (one per parent
  type, mostly null) to compensate for the lost per-parent structure?
- Would the parent tables actually share a sensible common supertype
  (favor a supertable), or are they unrelated entities that only
  incidentally share a child (favor reversed intersection tables)?
