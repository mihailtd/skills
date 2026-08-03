---
name: sql-antipatterns-normalization

description: Reference for the rules of relational normalization — what makes a table a true relation, first through fifth normal form plus Boyce-Codd/Domain-Key/sixth normal form, and the common myths (normalization isn't about performance, pseudokeys, or "separating everything"). Use when designing a schema, deciding whether a denormalization trade-off is deliberate, or explaining why a design violates a specific normal form.
---

# SQL Antipatterns — Rules of Normalization

This skill is the theoretical backing behind most of the other skills in
this package: a quick reference for what makes a table a proper relation,
what each normal form actually requires, and the common myths about
normalization that lead teams to denormalize (or over-normalize) for the
wrong reasons.

---

## 1. What "Relational" Actually Means

"Relational" doesn't refer to relationships *between* tables — it refers to
the relationship between columns *within* one table (a mathematical
relation: a subset of the combinations of values from several domains,
one domain per column — e.g. the valid pairings of baseball teams to their
home cities, out of every conceivable team/city combination).

Before any normal form applies, a table has to actually be a relation:

- **Rows have no top-to-bottom order.** A query's row order is
  unspecified unless you add `ORDER BY` — the same set of rows is the same
  result regardless of the order they came back in.
- **Columns have no left-to-right order.** Which column you list first is
  a naming/display choice, not meaning — this is the theoretical grounding
  for why [[sql-antipatterns-implicit-columns]] is wrong to reference
  columns by ordinal position instead of name.
- **No duplicate rows.** Once a fact is stated, restating it identically
  adds nothing. A primary key (or a `NOT NULL` `UNIQUE` key) is what makes
  "duplicate" well-defined — without one, you can't even tell two rows
  apart to say they're duplicates. Duplication *among non-key columns* is
  fine (two different baseball teams can share a city) — it's the whole
  row, identified by its key, that must be unique.
- **Every column has one type, one meaning, and one value per row.**
  A column's meaning must be constant across every row. This is exactly
  what [[sql-antipatterns-entity-attribute-value]] and
  [[sql-antipatterns-polymorphic-associations]] break: an EAV
  `attr_value` column holds a different *kind* of fact depending on which
  `attr_name` sits next to it on that row, and a polymorphic `issue_id`
  column means a different thing depending on the sibling `issue_type`
  value. Neither column has one consistent meaning across all its rows.
- **Rows have no hidden components.** A row is fully described by its
  declared columns' data values — not by physical storage details like an
  internal row ID or object ID. This is why
  [[sql-antipatterns-pseudokey-neat-freak]] insists a primary key is not
  the same concept as a row number: a key is a data value with meaning
  *to your application* (uniqueness), not a storage-position artifact.
  Database-specific pseudocolumns that expose internal storage details
  (Oracle's `ROWNUM`, PostgreSQL's `OID`) aren't part of the relation
  itself, even when a brand happens to expose them.

Only once a table satisfies all of the above can it be evaluated against
the numbered normal forms below.

---

## 2. Myths About Normalization

These claims are common, stated with confidence, and wrong:

- **"Normalization makes a database slower; denormalization makes it
  faster."** False in general. Normalization can mean more joins for some
  queries, but denormalizing to help one query shape almost always costs
  another — e.g. a comma-separated product list per bug
  ([[sql-antipatterns-jaywalking]]) makes "products for this bug" cheap
  but "bugs for this product" expensive. Model in normal form first;
  denormalize only after measuring a real, specific bottleneck — the same
  Measure-first discipline as [[sql-antipatterns-index-shotgun]]'s MENTOR
  approach, applied to schema instead of indexes.
- **"Normalization means pushing data into child tables addressed by a
  pseudokey."** False — pseudokeys are a convenience/performance/storage
  choice (see [[sql-antipatterns-id-required]]), unrelated to
  normalization as a concept. You can have a fully normalized schema built
  entirely on natural keys, with no surrogate keys anywhere.
- **"Normalization means separating attributes as much as possible, like
  EAV."** False — and backwards. EAV ([[sql-antipatterns-entity-attribute-value]])
  is not an especially "normalized" design; it actively violates the "one
  column, one meaning" relation requirement in §1. Normalization is about
  eliminating redundancy and anomalies, not about maximizing how many
  tables/columns a fact is spread across.
- **"Nobody needs to normalize past third normal form."** False, and not
  just theoretically — one study found more than 20% of business
  databases satisfying the first three normal forms while violating the
  fourth. That's common enough to be worth checking for deliberately, not
  a corner case to wave away.

---

## 3. What Normalization Is Actually For

Three objectives — notably, **performance is not one of them**:

1. Represent real-world facts in a way that can be understood correctly.
2. Avoid storing facts redundantly, which is what allows two copies of the
   same fact to silently disagree (an anomaly).
3. Support integrity constraints that keep the data consistent.

The performance case for normalization is indirect but real: an
unnormalized schema tends to accumulate inconsistent, duplicated data over
time, which then costs real engineering effort to detect and clean up, and
costs the business real money when decisions get made on faulty data.
Weigh that ongoing cost against whatever join overhead normalization
supposedly avoids.

A table that satisfies a given normal form's rule is "in" that normal form.
Each successive normal form is progressively stricter — a table in a higher
normal form also satisfies every normal form below it.

---

## 4. First Normal Form (1NF)

**Requirements:** the table must be a valid relation (§1), and must have
no repeating groups — no row may hold multiple values drawn from the same
domain, whether packed into one column or spread across a set of
similarly-purposed columns.

Two antipatterns in this package are exactly a 1NF violation:

- **Multiple values in one column** — [[sql-antipatterns-jaywalking]]'s
  comma-separated list.
- **Multiple values spread across sibling columns** —
  [[sql-antipatterns-multicolumn-attributes]]'s `tag1`/`tag2`/`tag3`.

**The fix in both cases:** a separate table, one value per row:

```sql
CREATE TABLE BugsTags (
    bug_id BIGINT UNSIGNED NOT NULL,
    tag    VARCHAR(20) NOT NULL,
    PRIMARY KEY (bug_id, tag),
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
);
```

---

## 5. Second Normal Form (2NF)

**Requirement:** on top of 1NF — if the table has a *compound* primary
key, every non-key column must depend on the *whole* key, not just part
of it. (If the primary key is a single column, 1NF and 2NF are
equivalent — this rule only bites with compound keys.)

Example violation — `coiner` (who first coined a tag) depends only on
`tag`, not on the full `(bug_id, tag)` key, so it's redundantly repeated
on every row that uses that tag:

```sql
-- violates 2NF: coiner depends only on `tag`, not on (bug_id, tag)
CREATE TABLE BugsTags (
    bug_id BIGINT UNSIGNED NOT NULL,
    tag    VARCHAR(20) NOT NULL,
    tagger BIGINT UNSIGNED NOT NULL,
    coiner BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (bug_id, tag)
);
```

This lets an update change the recorded `coiner` for `tag = 'crash'` on
one row without updating it everywhere else `'crash'` appears — an
anomaly. **Fix:** move the partially-dependent column to a table keyed on
just the part of the key it actually depends on:

```sql
CREATE TABLE Tags (
    tag    VARCHAR(20) PRIMARY KEY,
    coiner BIGINT UNSIGNED NOT NULL
);
CREATE TABLE BugsTags (
    bug_id BIGINT UNSIGNED NOT NULL,
    tag    VARCHAR(20) NOT NULL,
    tagger BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (bug_id, tag),
    FOREIGN KEY (tag) REFERENCES Tags(tag)
);
```

---

## 6. Third Normal Form (3NF)

**Requirement:** every non-key column must depend on the key — *only* the
key, not on another non-key column. If column B's value is determined by
column A, and A isn't part of the key, B doesn't belong on this table.

```sql
-- violates 3NF: assigned_email depends on assigned_to, not on bug_id
CREATE TABLE Bugs (
    bug_id         SERIAL PRIMARY KEY,
    assigned_to    BIGINT UNSIGNED,
    assigned_email VARCHAR(100),
    FOREIGN KEY (assigned_to) REFERENCES Accounts(account_id)
);
```

`assigned_email` is really an attribute of the *account*, not of the bug —
it belongs on `Accounts`, addressed by `account_id`, where it has no
redundancy.

---

## 7. Boyce-Codd Normal Form (BCNF)

A slightly stronger 3NF: the "must depend only on the key" rule applies
even when a table has **multiple candidate keys** (more than one column
or column-set that could each independently serve as the primary key).
3NF only constrains non-key columns; BCNF applies the same constraint
regardless of which candidate key you happened to designate as primary.

Example: if each bug can have at most one tag *per tag type* (impact,
subsystem, fix), then both `(bug_id, tag)` and `(bug_id, tag_type)` are
candidate keys. If `tag_type` is stored as a plain attribute alongside
`tag` rather than being derived from a `Tags` table that owns that
mapping, you can get the same tag associated with inconsistent types
across rows. **Fix:** move `tag_type` into the `Tags` table, so it's owned
by (i.e., truly depends on) exactly one thing — the tag itself, not the
association row.

---

## 8. Fourth Normal Form (4NF)

**Requirement:** don't let one table carry more than one independent
many-to-many relationship at once.

```sql
-- violates 4NF: three independent many-to-many relationships crammed into one table
CREATE TABLE BugsAccounts (
    bug_id      BIGINT UNSIGNED NOT NULL,
    reported_by BIGINT UNSIGNED,
    assigned_to BIGINT UNSIGNED,
    verified_by BIGINT UNSIGNED,
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
    -- ...
);
```

This can't even have a sensible primary key (you'd need all four columns,
but `assigned_to`/`verified_by` are nullable, and key columns must be
`NOT NULL`), and it forces redundant rows whenever the three relationships
have different cardinalities per bug. **Fix:** one intersection table per
independent many-to-many relationship:

```sql
CREATE TABLE BugsReported (
    bug_id BIGINT NOT NULL, reported_by BIGINT NOT NULL,
    PRIMARY KEY (bug_id, reported_by)
);
CREATE TABLE BugsAssigned (
    bug_id BIGINT NOT NULL, assigned_to BIGINT NOT NULL,
    PRIMARY KEY (bug_id, assigned_to)
);
CREATE TABLE BugsVerified (
    bug_id BIGINT NOT NULL, verified_by BIGINT NOT NULL,
    PRIMARY KEY (bug_id, verified_by)
);
```

This is the same underlying idea as [[sql-antipatterns-jaywalking]]'s
fix, generalized to the case of *multiple simultaneous* many-to-many
relationships instead of just one.

---

## 9. Fifth Normal Form (5NF)

Any table that's already in BCNF and has no compound key is
automatically in 5NF. 5NF matters specifically for tables with compound
keys that conflate more than one independent relationship in a subtler
way than 4NF catches:

```sql
-- violates 5NF: mixes "who's assigned to this bug" with
-- "which products this engineer works on" in one table
CREATE TABLE BugsAssigned (
    bug_id      BIGINT UNSIGNED NOT NULL,
    assigned_to BIGINT UNSIGNED NOT NULL,
    product_id  BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (bug_id, assigned_to)
);
```

This only records which products an engineer happens to be *currently
assigned to via a bug* — not which products they're generally available to
work on — and it redundantly repeats the engineer/product association
every time that engineer is assigned another bug on the same product.
**Fix:** separate the "assigned to this bug" fact from the "qualified for
this product" fact — they're independent relationships:

```sql
CREATE TABLE BugsAssigned (
    bug_id BIGINT UNSIGNED NOT NULL, assigned_to BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (bug_id, assigned_to)
);
CREATE TABLE EngineerProducts (
    account_id BIGINT UNSIGNED NOT NULL, product_id BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (account_id, product_id)
);
```

Now "which products can this engineer work on" is representable
independent of any specific bug assignment.

---

## 10. Further Normal Forms

- **Domain-Key Normal Form (DKNF):** every constraint on the table must
  follow logically from domain constraints (a column's allowed values) and
  key constraints (uniqueness) alone. 3NF, 4NF, 5NF, and BCNF are all
  subsumed by DKNF. A constraint that relates *two non-key columns to each
  other* — e.g. "if `status` is `NEW` or `DUPLICATE`, then `hours` must be
  0 and `verified_by` must be null" — falls outside domain/key
  constraints, so it needs a `CHECK` constraint or trigger rather than
  being expressible through table structure alone.
- **Sixth Normal Form (6NF):** eliminates join dependencies entirely,
  typically to support tracking every attribute's history of change over
  time (temporal databases). Taken to its logical conclusion, this can
  mean nearly every column gets its own history table — technically
  eliminating redundancy, but making ordinary queries laborious (many
  joins just to reconstruct one "current" row). Overkill for most
  applications; used deliberately in data-warehousing techniques like
  Anchor Modeling, where querying "what did this look like at time T" is
  a first-class requirement, not an afterthought.

---

## 11. Using This as a Design Tool

- When something about a design feels wrong but you can't articulate why,
  check it against the relation criteria in §1 first, then the normal
  forms in order — the specific rule it breaks usually names the actual
  problem precisely (an anomaly waiting to happen), not just a vague
  "code smell."
- Recognizing *which* normal form a design breaks tells you exactly what
  kind of anomaly to expect (a column update needs to touch multiple rows,
  a delete loses information it shouldn't, an insert requires a
  placeholder value) — and points at the matching antipattern skill in
  this package for the concrete fix.
- Don't treat "more normalized" as an unconditional goal. Model in normal
  form as the default, then denormalize only for a measured, specific
  reason — see [[sql-antipatterns-storing-prices]] for a case that looks
  like a normalization violation but isn't (a deliberately captured
  point-in-time fact), and [[sql-antipatterns-metadata-tribbles]] for
  what actually goes wrong when "keep tables small" gets pursued the
  wrong way.
