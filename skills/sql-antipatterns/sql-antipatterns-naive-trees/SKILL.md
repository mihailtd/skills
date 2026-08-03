---
name: sql-antipatterns-naive-trees

description: Detects the Naive Trees antipattern — modeling hierarchical/tree data with only a parent_id Adjacency List and then fighting SQL to fetch arbitrary-depth ancestors or descendants — and guides toward recursive CTEs, Path Enumeration, Nested Sets, or Closure Table based on the actual access pattern.
---

# SQL Antipatterns — Naive Trees

This skill helps developers recognize when hierarchical data (org charts,
threaded comments, category trees, bill-of-materials) has been modeled with
a bare `parent_id` column and nothing else, and guides them to the tree
design that actually fits how the hierarchy is queried and modified.

---

## 1. Recognize the Antipattern

- **The tell:** a self-referencing `parent_id` column (Adjacency List) is the
  *only* structure used to represent a tree, and code is fighting to fetch
  whole subtrees or ancestor chains with plain SQL.

  ```sql
  CREATE TABLE Comments (
      comment_id SERIAL PRIMARY KEY,
      parent_id  BIGINT UNSIGNED,
      comment    TEXT NOT NULL,
      FOREIGN KEY (parent_id) REFERENCES Comments(comment_id)
  );
  ```

- **Listen for these phrases** — they signal Adjacency List is being asked
  to do more than it comfortably can:
  - "How many levels do we need to support in trees?" — a sign the fixed
    number of self-joins in a query can't handle arbitrary depth.
  - "I dread having to touch the code that manages the tree." — a sign an
    alternative model was picked but doesn't fit the app's actual operations.
  - "I need to run a script periodically to clean up orphaned rows." — a
    sign deletes/moves aren't keeping the tree structurally consistent.

---

## 2. Why It Breaks Down

- **Depth is hardcoded into the query shape.** Each level of the tree needs
  another self-join, and the number of joins in a query is fixed at write
  time:

  ```sql
  SELECT c1.*, c2.*, c3.*, c4.*
  FROM Comments c1
  LEFT OUTER JOIN Comments c2 ON c2.parent_id = c1.comment_id
  LEFT OUTER JOIN Comments c3 ON c3.parent_id = c2.comment_id
  LEFT OUTER JOIN Comments c4 ON c4.parent_id = c3.comment_id;
  -- can't see a 5th level, and COUNT()/SUM() across levels is awkward
  ```

- **The fallback is "fetch everything, filter in application code."** That
  means pulling far more rows than needed over the network and rebuilding
  tree structures in memory — expensive at any real scale (dozens of
  articles/day × hundreds of comments each adds up fast).
- **Deleting a subtree takes multiple round trips**, walking down level by
  level to find every descendant before deleting bottom-up to satisfy the
  foreign key. `ON DELETE CASCADE` only helps if descendants should always
  be deleted with their parent, never promoted or relocated.
- **Promoting/reparenting children of a deleted node is manual**: read the
  node's `parent_id`, `UPDATE` all its children to point to the grandparent,
  then delete the node — three dependent steps, no single statement.

---

## 3. Legitimate Uses

- If the tree has a small, effectively fixed depth (e.g. one level of
  categories → subcategories, never deeper) and you never need "give me
  every descendant regardless of depth," a plain Adjacency List is fine —
  don't build Path Enumeration/Nested Sets/Closure Table machinery for a
  hierarchy that will never need it. Confirm the actual depth requirement
  with the domain owner before assuming it's unbounded.
- If your database predates recursive query support, Adjacency List plus
  a depth-limited join or app-side traversal may be the pragmatic choice
  until you can upgrade.

---

## 4. The Fix: Pick the Tree Model That Matches the Access Pattern

There is no single universal replacement — each alternative trades off
referential integrity, read cost, and write cost differently. Ask "do we
read subtrees far more than we mutate the tree?" before choosing.

### a. Recursive Queries (keep Adjacency List, add `WITH RECURSIVE`)

The lowest-friction upgrade: keep the `parent_id` design — it's normalized
and has referential integrity — and use a recursive common table expression
to walk it in one query.

```sql
WITH RECURSIVE CommentTree AS (
    SELECT comment_id, parent_id, comment, 0 AS depth
    FROM Comments
    WHERE parent_id IS NULL
    UNION ALL
    SELECT c.comment_id, c.parent_id, c.comment, ct.depth + 1
    FROM CommentTree ct
    JOIN Comments c ON c.parent_id = ct.comment_id
)
SELECT * FROM CommentTree WHERE bug_id = 1234;
```

Supported by PostgreSQL 8.4+, MySQL 8.0+, SQL Server 2005+, Oracle 11g+
(older Oracle uses `START WITH ... CONNECT BY PRIOR` instead), DB2 UDB 8+,
SQLite 3.8.3+. Prefer this over the other three models whenever the engine
supports it and inserts/deletes are frequent — it keeps the schema
normalized while making the traversal easy.

### b. Path Enumeration

Store each node's full ancestor chain as a delimited string (like a Unix
path):

```sql
CREATE TABLE Comments (
    comment_id SERIAL PRIMARY KEY,
    path       VARCHAR(1000),   -- e.g. '1/4/6/'
    comment    TEXT NOT NULL
);

-- ancestors of comment #7 (path '1/4/6/7/')
SELECT * FROM Comments WHERE '1/4/6/7/' LIKE CONCAT(path, '%');

-- descendants of comment #4 (path '1/4/')
SELECT * FROM Comments WHERE path LIKE '1/4/%';
```

Cheap ancestor/descendant queries and great for UI breadcrumbs, but it has
the same weaknesses as [[sql-antipatterns-jaywalking]]: the database can't
validate that the path is well-formed or that every ID in it still exists,
maintaining it is on application code, and the column still has a length
ceiling. Use it only when you accept those trade-offs for read simplicity.

### c. Nested Sets

Encode each node with `nsleft`/`nsright` values from a depth-first
traversal, where a node's numbers span the numbers of all its descendants.

```sql
-- descendants of comment #4
SELECT c2.* FROM Comments c1
JOIN Comments c2 ON c2.nsleft BETWEEN c1.nsleft AND c1.nsright
WHERE c1.comment_id = 4;
```

Subtree reads are very fast and deleting a nonleaf node automatically
reattaches its descendants to its parent. But it has no referential
integrity, finding the *immediate* parent/child needs an extra self-join to
rule out intermediate ancestors, and every insert/move requires
renumbering every `nsleft`/`nsright` greater than the insertion point.
Only reach for this when reads of subtrees vastly outnumber writes.

### d. Closure Table

Add a second table storing every ancestor/descendant pair in the tree
(including each node's self-reference), not just direct parent-child pairs:

```sql
CREATE TABLE TreePaths (
    ancestor   BIGINT UNSIGNED NOT NULL,
    descendant BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (ancestor, descendant),
    FOREIGN KEY (ancestor) REFERENCES Comments(comment_id),
    FOREIGN KEY (descendant) REFERENCES Comments(comment_id)
);

-- descendants of comment #4
SELECT c.* FROM Comments c
JOIN TreePaths t ON c.comment_id = t.descendant
WHERE t.ancestor = 4;
```

The most versatile option: full referential integrity, straightforward
ancestor/descendant/subtree queries and moves, and the only design that
lets a node belong to more than one tree. The cost is an extra table that
grows faster than the number of nodes (one row per ancestor-descendant
pair, not per edge). Add a `path_length` column to make "immediate
child/parent" queries a simple equality filter.

---

## 5. Decision Table

| Design            | Ref. integrity | Delete | Insert | Query subtree | Query immediate child | Extra tables |
|-------------------|:--:|:--:|:--:|:--:|:--:|:--:|
| Adjacency List (alone) | Yes | Easy | Easy | **Hard** | Easy | 0 |
| Adjacency List + Recursive Query | Yes | Easy | Easy | Easy | Easy | 0 |
| Path Enumeration  | No | Easy | Easy | Easy | Easy | 0 |
| Nested Sets       | No | **Hard** | **Hard** | Easy | **Hard** | 0 |
| Closure Table     | Yes | Easy | Easy | Easy | Easy | 1 |

---

## 6. Review Checklist

- Does any query hardcode a fixed number of self-joins to walk the tree,
  or a comment/TODO about a "max depth"?
- Is a whole table being pulled into application memory just to assemble a
  subtree or compute an aggregate over it?
- Does deleting or reparenting a node take multiple dependent statements
  in application code instead of one?
- Is there a cleanup job for orphaned rows — a sign the tree isn't kept
  structurally consistent by the schema itself?
- Confirm what the code actually needs — arbitrary-depth reads, frequent
  subtree moves, multi-tree membership — before picking Recursive Query,
  Path Enumeration, Nested Sets, or Closure Table.

---

## Related guidance

PostgreSQL-specific remedy:

- database-postgresql-recursive-ctes
