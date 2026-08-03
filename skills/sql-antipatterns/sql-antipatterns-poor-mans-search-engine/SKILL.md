---
name: sql-antipatterns-poor-mans-search-engine

description: Detects the Poor Man's Search Engine antipattern — using LIKE '%word%' or regular expressions to search text columns for keywords — which forces a full table scan, can't use a conventional index, and produces false matches (substring hits inside unrelated words), and guides toward full-text indexing (vendor features, a dedicated search engine, or a hand-built inverted index).
---

# SQL Antipatterns — Poor Man's Search Engine

This skill helps developers recognize when `LIKE '%keyword%'` or a regular
expression is being used as a keyword-search feature, and guides them
toward tools actually built for full-text search.

---

## 1. Recognize the Antipattern

- **The tell:** keyword search implemented as a wildcard pattern match or
  regex against a text column:

  ```sql
  SELECT * FROM Bugs WHERE description LIKE '%crash%';
  SELECT * FROM Bugs WHERE description REGEXP 'crash';
  ```

- **Listen for these phrases**:
  - "How do I insert a variable between two wildcards in a `LIKE`
    expression?" — building a keyword search directly out of pattern
    matching.
  - "How do I write a regex to check a string contains multiple words, or
    doesn't contain a word, or matches any form of a word?" — reaching for
    increasingly elaborate regex to reimplement what full-text search
    already does natively (stemming, boolean operators, ranking).
  - "Our site's search got unusably slow as the document collection grew."
    — the direct, expected consequence of this antipattern's scaling
    behavior.

---

## 2. Why It Breaks Down

- **No index can help.** SQL treats a column's value as atomic — matching
  is a full scan of the string. `LIKE '%word%'` and regex both share this
  limitation: a leading wildcard means the match could start anywhere,
  so there's no sorted structure to exploit (same underlying issue as
  the search cases in [[sql-antipatterns-index-shotgun]]). The full-scan
  cost, combined with per-row string matching being much more expensive
  than an integer/index comparison, makes this expensive at any real
  scale — and it gets worse, not better, as the document collection grows.
- **False matches from naive substring search.** `LIKE '%one%'` also
  matches `money`, `prone`, `lonely` — anything containing that substring,
  not the whole word. Working around this with word-boundary regex
  (`REGEXP '\\bone\\b'`) is possible on engines that support it, but it's
  extra complexity bolted onto a technique that was already the wrong tool.
- **No support for what real search needs.** Matching different word forms
  (`crash` should also find `crashed`, `crashing`), ranking results by
  relevance, or combining terms with boolean logic (`crash` AND NOT
  `save`) are all things a real full-text search feature does natively —
  reimplementing any of them on top of `LIKE`/regex means writing and
  maintaining a worse version of functionality that already exists.

---

## 3. Legitimate Uses

- `LIKE`/regex is a fine choice for a query that's genuinely run rarely —
  the cost of building and maintaining a full-text index (or standing up
  separate search infrastructure) can exceed the cost of occasionally
  running an inefficient scan, especially for ad hoc queries where an
  index might not even apply to the specific search anyway.
- For simple, narrow cases — a known short pattern, not general user-driven
  keyword search — plain pattern matching gets the right result without
  the overhead of a full search subsystem. The antipattern is reaching for
  `LIKE`/regex as the mechanism for user-facing, general-purpose keyword
  search, not using it at all.

---

## 4. The Fix: Use a Tool Built for Search

Pick based on what's already available and how much infrastructure you're
willing to run:

### a. The database's built-in full-text feature

Every major database brand has one. They're not portable — syntax and
capability vary — but they integrate cleanly with ordinary queries and
usually keep the index in sync automatically:

- **MySQL**: `FULLTEXT` index + `MATCH() ... AGAINST()`, with a boolean
  mode for `+required -excluded` terms:

  ```sql
  ALTER TABLE Bugs ADD FULLTEXT INDEX bugfts (summary, description);
  SELECT * FROM Bugs WHERE MATCH(summary, description) AGAINST ('crash');
  SELECT * FROM Bugs WHERE MATCH(summary, description)
      AGAINST ('+crash -save' IN BOOLEAN MODE);
  ```

- **PostgreSQL**: `TSVECTOR`/`tsquery` with a GIN index, easiest kept in
  sync via a generated column:

  ```sql
  CREATE TABLE Bugs (
      bug_id SERIAL PRIMARY KEY,
      summary VARCHAR(80),
      description TEXT,
      ts_bug_text TSVECTOR GENERATED ALWAYS AS
          (to_tsvector('english', COALESCE(summary,'') || COALESCE(description,''))) STORED
  );
  CREATE INDEX bugs_ts ON Bugs USING GIN(ts_bug_text);
  SELECT * FROM Bugs WHERE ts_bug_text @@ to_tsquery('crash');
  ```

  **PostgreSQL note:** this is worth calling out as a standout case among
  the vendor options — Postgres's full-text search is unusually complete
  for a feature built into a general-purpose relational database, not a
  bolted-on afterthought:

  - **Stemming and language-aware matching are built in** (`to_tsvector`
    with a language config like `'english'` normalizes `crash`,
    `crashed`, `crashing` to the same lexeme), addressing the "match
    different word forms" requirement from the objective in §1 without
    any external tooling.
  - **Ranking is native**: `ts_rank()`/`ts_rank_cd()` score matches by
    relevance, and `ts_headline()` generates highlighted excerpts —
    covering two more things the naive `LIKE`/regex approach has no
    answer for at all.
  - **Boolean and phrase queries are first-class** via `tsquery`
    operators — `to_tsquery('crash & !save')` for AND-NOT,
    `phraseto_tsquery('save button crash')` for phrase proximity —
    without a proprietary "boolean mode" string like MySQL's.
  - **Generated `STORED` columns (PG12+) keep the index trivially in
    sync** with underlying text changes — no trigger to write and
    maintain (compare to MySQL's `FULLTEXT`, which auto-updates too, but
    without the flexibility of controlling exactly how the vector is
    derived from multiple weighted source columns via `setweight()`).
  - Still not standard SQL, and still a real index/storage cost — the
    trade-off in §3 (is this query frequent and general-purpose enough to
    justify it) still applies. But of all the vendor-specific options in
    this section, Postgres's is the one most likely to fully replace a
    dedicated search engine for small-to-medium document collections,
    rather than being a partial stand-in for one.

- **SQL Server**: `CREATE FULLTEXT CATALOG`/`INDEX` plus `CONTAINS()`/
  `FREETEXT()`.
- **Oracle**: text indexing via `CONTEXT` (general text), `CTXCAT`
  (short-text + structured columns), `CTXXPATH` (XML), `CTXRULE`
  (document classification), or JSON search indexes — pick the index type
  that matches the shape of the content, not just `CONTEXT` by default.
- **SQLite**: FTS3/FTS4 (or newer) virtual tables — note these must be
  built into SQLite explicitly (not always on by default) and require
  copying indexed columns into the virtual table and keeping it in sync
  yourself.

Read the current documentation for whichever brand you're on before
committing — these features change across versions, and defaults
(language, stemming, sync behavior) vary widely.

### b. A dedicated search engine

Worth it when you need behavior consistent across database brands, more
advanced relevance ranking, distributed/high-scale search, or search
across many data sources beyond one database:

- **Sphinx Search** — integrates with MySQL/PostgreSQL, fast, good for
  high-scale search over infrequently-updated data, speaks a MySQL-like
  protocol so existing client tooling mostly works unmodified.
- **Apache Lucene / Solr** — mature, Java-based; Lucene is a library you
  build documents into directly, Solr wraps it with a REST-like server and
  data-import tooling (including pulling straight from an SQL query).
- **Elasticsearch / OpenSearch** — both built on Lucene, add
  analytics/visualization tooling around it. Elasticsearch and OpenSearch
  diverged after a 2021 licensing fork; expect their feature sets to keep
  drifting apart, so evaluate both against your actual needs rather than
  assuming feature parity.

All of these require keeping the external index in sync with the database
yourself (unless the integration handles it) — that's a real operational
cost to weigh against the payoff.

### c. Roll your own inverted index

When you want a database-independent solution without external
infrastructure: build a `Keywords` table and a `BugsKeywords`-style
intersection table mapping keywords to the rows that contain them.

```sql
CREATE TABLE Keywords (
    keyword_id SERIAL PRIMARY KEY,
    keyword    VARCHAR(40) NOT NULL,
    UNIQUE KEY (keyword)
);
CREATE TABLE BugsKeywords (
    keyword_id BIGINT UNSIGNED NOT NULL,
    bug_id     BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (keyword_id, bug_id),
    FOREIGN KEY (keyword_id) REFERENCES Keywords(keyword_id),
    FOREIGN KEY (bug_id)     REFERENCES Bugs(bug_id)
);
```

- The first search for a new keyword still costs a full scan (using
  `LIKE`/regex internally, once) — but the match is cached as rows in the
  intersection table, so every subsequent search for that keyword is a
  plain indexed join, not a rescan.
- Populate the intersection table lazily (search on demand, cache the
  result) or eagerly (pre-run searches for anticipated keywords so the
  first real user isn't the one who pays the scan cost).
- Keep it in sync with an `INSERT` trigger (index new rows against
  existing keywords) and, if descriptions are ever edited, an `UPDATE`
  trigger to reanalyze changed text.
- This is more code to own than a vendor feature or dedicated search
  engine, but it stays entirely within standard SQL/stored procedures —
  reasonable when you specifically want to avoid both proprietary
  extensions and external infrastructure.

---

## 5. Review Checklist

- Is a keyword search feature implemented as `LIKE '%word%'` or a regex
  against a text column, rather than a full-text index or search engine?
- Does the query need word-boundary gymnastics to avoid false substring
  matches (`money` matching a search for `one`)?
- Has search performance been measured as the underlying document/row
  count grows, or is degradation only being noticed after it's already a
  problem in production?
- Is the search need broad and general-purpose (user-facing keyword
  search) — favoring a real full-text solution — or narrow, rare, and
  simple enough that `LIKE`/regex is a reasonable, deliberate choice?
- If a full-text solution is already in place, is the index actually kept
  in sync with underlying data changes (via triggers, generated columns,
  or an import pipeline), or can it silently drift stale?

---

## Related guidance

PostgreSQL-specific remedy:

- database-postgresql-pgtrgm-fuzzy-matching
