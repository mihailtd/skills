---
name: sql-antipatterns-keyless-entry

description: Detects the Keyless Entry antipattern — skipping foreign key constraints and trying to enforce referential integrity in application code instead — and guides toward declaring FOREIGN KEY constraints with appropriate ON UPDATE/ON DELETE actions.
---

# SQL Antipatterns — Keyless Entry

This skill helps developers recognize when referential integrity is being
enforced (or not enforced) by application code instead of the database, and
guides them back to declared foreign key constraints.

---

## 1. Recognize the Antipattern

- **The tell:** tables have logical parent-child relationships
  (`Bugs.reported_by` → `Accounts.account_id`, `Bugs.status` →
  `BugStatus.status`) but no `FOREIGN KEY` constraint declares them. The
  relationship exists only in application code and developers' heads.
- **Listen for these phrases**:
  - "How do I query to check for a value that exists in one table and not
    the other?" — manually hunting for orphaned rows a constraint would
    have prevented outright.
  - "Is there a quick way to check a value exists in one table before I
    insert into a second table?" — reimplementing what a foreign key
    already does atomically, using the parent's index.
  - "Foreign keys? I was told not to use them, they slow down the
    database." — a performance objection usually made without measuring,
    and usually wrong once the cost of the alternative is counted.
- **A related giveaway:** the presence of scheduled "quality control"
  scripts whose job is to scan for and report/fix broken references. Their
  existence means the schema itself doesn't guarantee integrity.

---

## 2. Why It Breaks Down

- **"Write flawless code" isn't a strategy.** Enforcing integrity in
  application code means every insert/update/delete path — across every
  service, script, and ad hoc query tool session that ever touches the
  database — has to get it right, forever. One missed code path anywhere
  is enough to corrupt data.
- **Check-then-act has a race condition.** `SELECT` to confirm a parent
  row exists, then `INSERT` the child, leaves a window: another client can
  delete the parent in between. The only fix without a real constraint is
  table-level locking, which kills concurrency — the opposite of the
  performance benefit keyless design was supposed to buy.
- **Detection is expensive and always lagging.** A manual orphan-check
  query has to be written and run per relationship:

  ```sql
  SELECT b.bug_id, b.status
  FROM Bugs b LEFT OUTER JOIN BugStatus s ON b.status = s.status
  WHERE s.status IS NULL;
  ```

  Run it too often and it costs performance; run it too rarely and bad
  data sits live in production, corrupting reports and workflows, until
  someone notices.
- **Correction is sometimes impossible.** An invalid lookup value (like a
  bogus `status`) can be reset to a sensible default. An invalid
  `reported_by` pointing at a deleted account often can't be — there's no
  principled value to substitute, only a guess.
- **Multi-table updates become a catch-22.** Deleting a row with
  dependents requires deleting children first, in the right order, and
  that logic has to be kept in sync by hand as child tables are added.
  Updating a value that both a parent and its children need to agree on
  *simultaneously* is worse — there's no ordering of two separate
  `UPDATE` statements that doesn't violate the relationship at some
  intermediate moment.

---

## 3. Legitimate Uses

- Some engines take extra locks on a parent row while a dependent update
  is in flight; if that locking behavior is a genuine bottleneck for your
  workload, dropping the constraint is a real (if costly) trade-off, not a
  reflexive one.
- During active data-cleanup efforts it can be pragmatic to temporarily
  tolerate known orphaned rows while the correct reference is still being
  researched, rather than block all writes until it's resolved.
- Some schema-change tooling (e.g. `pt-online-schema-change` for MySQL)
  works by renaming tables, which forces every foreign key pointing at the
  altered table to be dropped and recreated around the rename. Teams that
  lean heavily on this kind of tooling sometimes trade away FKs to avoid
  that operational overhead — a real cost, but one to weigh consciously
  against what's given up.
- You may simply be stuck on a database/storage engine that doesn't
  support foreign keys at all (MySQL's MyISAM, SQLite before 3.6.19). In
  that case the quality-control-script approach in this chapter is a
  compensating control, not a preference.
- If a design can't support foreign keys at all (fully polymorphic
  associations, EAV), treat that as a signal to question the design
  itself — see the related antipatterns it usually implies.

---

## 4. The Fix: Declare the Constraints

Let the database refuse bad data at the moment it's written, instead of
detecting it after the fact:

```sql
CREATE TABLE Bugs (
    reported_by BIGINT UNSIGNED NOT NULL,
    status      VARCHAR(20) NOT NULL DEFAULT 'NEW',
    FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
    FOREIGN KEY (status)      REFERENCES BugStatus(status)
);
```

This applies uniformly to every path that touches the table — application
code, scripts you didn't write, ad hoc queries run by hand — with no way
to bypass it. Fewer lines of integrity-checking code also means fewer bugs;
industry data puts defect rates at roughly 15-50 bugs per 1,000 lines of
code, so code you don't have to write is code that can't be buggy.

### Use `ON UPDATE`/`ON DELETE` to resolve the catch-22

Cascading actions let the database make coordinated multi-table changes
atomically — something application code can't replicate as a pair of
separate statements:

```sql
CREATE TABLE Bugs (
    reported_by BIGINT UNSIGNED NOT NULL,
    status      VARCHAR(20) NOT NULL DEFAULT 'NEW',
    FOREIGN KEY (reported_by) REFERENCES Accounts(account_id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT,
    FOREIGN KEY (status) REFERENCES BugStatus(status)
        ON UPDATE CASCADE
        ON DELETE SET DEFAULT
);
```

- `RESTRICT` blocks deleting a parent row that still has dependents —
  correct when there's no sensible way to handle orphaning the children.
- `CASCADE` propagates a parent key change (or deletion) to matching child
  rows automatically.
- `SET DEFAULT` resets child rows to a fallback value when their parent
  disappears, instead of leaving them pointing nowhere.

Pick the action per relationship based on what "the parent is gone" should
mean for that specific child column — there's no single right default.

### The overhead is smaller than the alternative

Weigh the small cost of the constraint/index against everything it removes:
no more pre-flight `SELECT` checks before writes, no table locks to protect
multi-table changes, and no periodic scripts to find and fix orphans that
already made it into production.

---

## 5. Review Checklist

- Does every logical parent-child relationship in the schema have a
  declared `FOREIGN KEY`, or does at least one rely on "the application
  handles it"?
- Are there scheduled or ad hoc scripts whose job is to detect/repair
  broken references? That's a symptom, not a solution.
- Do inserts/deletes do a `SELECT`-then-act check for referential
  integrity instead of letting a constraint enforce it atomically?
- For each foreign key, has `ON UPDATE`/`ON DELETE` behavior been chosen
  deliberately (`CASCADE`, `RESTRICT`, `SET DEFAULT`, `SET NULL`), or left
  at a default that doesn't match what should happen to the children?
- If foreign keys genuinely can't be used, is that because of a real
  storage-engine limitation, or because the schema itself (EAV,
  polymorphic associations) is the actual problem to fix?
