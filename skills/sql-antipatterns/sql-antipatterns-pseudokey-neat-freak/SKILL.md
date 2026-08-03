---
name: sql-antipatterns-pseudokey-neat-freak

description: Detects the Pseudokey Neat-Freak antipattern — treating gaps in an auto-generated primary key as a problem to fix, by reusing the lowest unused value or renumbering existing rows to stay contiguous — which risks race conditions, breaks external references, cascades painfully through foreign keys, and reintroduces gaps immediately anyway. Guides toward accepting gaps as normal and using ROW_NUMBER() (including PARTITION BY for per-group numbering) when a true gap-free row number is actually needed.
---

# SQL Antipatterns — Pseudokey Neat-Freak

This skill helps developers (and the people who ask them to "clean up" a
database) recognize that gaps in a pseudokey sequence are normal and
harmless, and guides them toward `ROW_NUMBER()` for the cases where a
genuinely gap-free, consecutive number is actually needed.

---

## 1. Recognize the Antipattern

- **The tell:** discomfort at seeing a primary key sequence with holes in
  it (`1, 2, 4, ...` — where did `3` go?), followed by an attempt to either
  reuse the lowest unused value for new rows, or renumber existing rows to
  close the gap:

  ```sql
  -- find the lowest unused id to reuse for the next insert
  SELECT b1.bug_id + 1
  FROM Bugs b1
  LEFT OUTER JOIN Bugs AS b2 ON (b1.bug_id + 1 = b2.bug_id)
  WHERE b2.bug_id IS NULL
  ORDER BY b1.bug_id LIMIT 1;

  -- or: renumber an existing row down into the gap
  UPDATE Bugs SET bug_id = 3 WHERE bug_id = 4;
  ```

- **Listen for these phrases**:
  - "What happened to bug_id 4?" — misplaced anxiety over a normal,
    harmless gap.
  - "How can I query for the first unused ID?" — almost always in service
    of reassigning it, which is the antipattern.
  - "How do I reuse an auto-generated ID after a rollback?" — a
    misunderstanding of how pseudokey generation works (see §2).
  - "What if we run out of numbers?" — see
    [[sql-antipatterns-id-required]] §5 for why this fear is usually
    unfounded for a properly sized key type; it's not a real justification
    for reuse.

---

## 2. Why It Breaks Down

- **Gaps are an expected, structural side effect of how pseudokeys work,
  not a sign of data loss.** A pseudokey generator hands out a value
  that's strictly greater than the last one it generated — not "the
  highest value currently in the table." Any deleted row, rolled-back
  insert, or failed transaction leaves a permanent gap, by design.
  Pseudokey allocation deliberately happens *outside* transaction scope
  specifically so concurrent inserts never block each other or collide —
  that's what makes gaps an acceptable, even necessary, trade-off (see
  [[sql-antipatterns-id-required]] §2's note on sequences avoiding the
  `MAX()+1` race condition).
- **Finding "the lowest unused value" has the exact same race condition it
  was meant to replace.** Both the "reuse the lowest gap" and "renumber
  down into the gap" approaches require finding an unused value via query,
  then using it — and if two clients do this concurrently, they can both
  compute the same "next" value and collide, the very problem sequences
  exist to solve. This makes the "fix" strictly worse than doing nothing:
  slower, and reintroducing a bug the schema had already eliminated.
- **Renumbering cascades painfully, or breaks referential integrity
  silently.** Any child table referencing the renumbered row needs its
  foreign key updated too — automatic only if `ON UPDATE CASCADE` was
  declared; otherwise it's disable constraints, update every dependent
  row by hand, and re-enable, a disruptive and error-prone process.
- **The fix doesn't even last.** The pseudokey generator has no idea a
  value was freed up by renumbering — the next auto-generated value is
  still one greater than the last value *it* produced, so a fresh gap
  opens immediately where the renumbered row used to be. You're
  committing to redo this "cleanup" forever.
- **Reusing a "freed" key value can silently reassign meaning to
  unrelated data.** If key `789` belonged to a deleted account and gets
  reused for a new one, anything that still references `789` from outside
  the immediate transaction — an old email thread, an external audit log,
  a cached link — now points at the wrong entity. The new owner of `789`
  inherits consequences that were never theirs.
- **Renumbering breaks anything outside the database that depends on the
  old values staying stable** — printed asset labels, external system
  references, prior reports that cited specific IDs. This is the actual
  disaster in the chapter's opening story: consolidating "gapless" asset
  IDs invalidated a quarter's worth of already-published financial
  reporting built on the old numbers.

---

## 3. Legitimate Uses

There's no good reason to renumber or reuse a true pseudokey — if the
values in a key carry enough meaning that people expect them to be
contiguous or otherwise significant, that's a signal the column is
actually a **natural key**, not a pseudokey, and natural keys are
routinely allowed to change or be reassigned as their real-world meaning
changes. The antipattern is specifically about treating an auto-generated,
semantically-meaningless surrogate key as if it needed to look tidy.

---

## 4. The Fix: Accept Gaps, Use ROW_NUMBER() When You Actually Need Consecutive Numbers

- **A pseudokey's only job is to identify a row uniquely** — nothing in
  that job requires consecutive values. Don't conflate a primary key with
  a row number; they answer different questions (a primary key identifies
  one row in one table; a row number describes a row's position within a
  particular query's result set, which can vary by sort order, filters,
  and joins).
- **When you genuinely need gap-free, consecutive numbers** — most
  commonly for pagination — generate them at query time with the
  `ROW_NUMBER()` window function, rather than trying to keep the stored
  primary key itself gap-free:

  ```sql
  SELECT t1.* FROM (
      SELECT a.account_name, b.bug_id, b.summary,
             ROW_NUMBER() OVER (ORDER BY a.account_name, b.date_reported) AS rn
      FROM Accounts a JOIN Bugs b ON (a.account_id = b.reported_by)
  ) AS t1
  WHERE t1.rn BETWEEN 51 AND 100;
  ```

  This is supported by essentially every modern SQL database and produces
  numbers with no gaps, regardless of what the underlying primary keys
  look like.
- **GUIDs are an alternative pseudokey generation strategy**, not a fix
  for "tidiness" — they trade away sequential-looking values entirely in
  exchange for collision-free generation across multiple database servers
  with no coordination needed:

  ```sql
  CREATE TABLE Bugs (
      bug_id UNIQUEIDENTIFIER DEFAULT NEWID()
      -- ...
  );
  ```

  Trade-offs: values are long, unwieldy to type, carry no ordering
  information (you can't infer recency from a greater GUID the way you can
  with an incrementing integer), and cost more storage (16+ bytes vs. a
  typical 4-byte integer) with corresponding index overhead. Choose GUIDs
  for their actual benefit (distributed generation), not as an indirect
  way to stop worrying about gaps.

### When the request comes from a manager or stakeholder, not a technical requirement

Since this often starts as a request from someone uneasy about how gaps
*look*, not a real technical need:

- Explain plainly that gaps are a normal, harmless side effect of how
  unique IDs are generated safely under concurrent access — not a sign of
  lost data.
- Give a realistic cost estimate for renumbering: computing new values,
  handling collisions, cascading through every foreign key and external
  system that references the old values, and the risk of exactly the kind
  of reporting mismatch described in §2. Most requests for this kind of
  "cleanup" don't survive contact with an honest cost estimate.
- If the values in the key really do need to carry visible meaning to
  stakeholders, that's a sign to expose a natural key (or a separate,
  presentational identifier) alongside the pseudokey — not to make the
  pseudokey itself behave like one.

---

## 5. Per-Group Numbering ("Restart at 1 for Each Subgroup")

A related request: an auto-increment-like column that restarts at 1 for
each subgroup (rankings per team, invoice numbers per customer, comment
numbers per bug). Implementing this as an actual stored, incrementing
column per group has the same problems as pseudokey renumbering, plus
its own:

- Allocating the next value per group means checking the most recent
  value already used *within that group* before inserting — serializing
  concurrent inserts within the same group and creating a real bottleneck.
- An alternative — spinning up a separate sequence generator per subgroup
  on demand — can explode into as many sequence objects as there are
  subgroups (in the worst case, as many as there are rows).
- Deletions or rolled-back inserts reintroduce gaps within the subgroup
  numbering too, for exactly the same reasons as the top-level case —
  "restart at 1 per group" doesn't grant immunity to any of the underlying
  issues.

The fix is the same idea as §4, scoped per group: use `ROW_NUMBER() OVER
(PARTITION BY ...)` at query time instead of maintaining a real stored
per-group counter:

```sql
SELECT bug_id, author, comment,
       ROW_NUMBER() OVER (PARTITION BY bug_id ORDER BY comment_date) AS comment_number
FROM Comments;
```

This produces gap-free, restarting-per-group numbers on demand, computed
fresh from the current data, with none of the concurrency or
gap-reintroduction problems of maintaining a real stored counter per
group.

---

## 6. Review Checklist

- Is there code (or a recurring manual process) whose purpose is finding
  and reusing the "lowest unused" primary key value?
- Does anything renumber existing rows' primary keys to remove gaps, and
  if so, does it handle every foreign key reference (via `ON UPDATE
  CASCADE` or equivalent), and every external system that might reference
  the old values?
- Is a pseudokey value ever reused for a genuinely different logical
  entity after the original row was deleted?
- Where a true gap-free sequence is actually needed (pagination, display
  ranking), is it produced with `ROW_NUMBER()` at query time, rather than
  by keeping the stored primary key itself artificially contiguous?
- Is a "restart numbering per group" requirement implemented as a real
  stored counter (with its concurrency and gap problems), instead of
  `ROW_NUMBER() OVER (PARTITION BY ...)`?
- If stakeholders expect the key's values to look meaningful or tidy, is
  that better solved by exposing a natural key/display identifier, rather
  than making the pseudokey behave like one?
