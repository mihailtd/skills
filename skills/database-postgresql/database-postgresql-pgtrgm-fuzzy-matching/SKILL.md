---
name: database-postgresql-pgtrgm-fuzzy-matching

description: Guides the implementation of fuzzy string matching, similarity scoring, and advanced LIKE/Regex optimization using the pg_trgm (Trigram) extension in PostgreSQL.
---

# PostgreSQL `pg_trgm` Fuzzy Matching

This skill helps developers implement typo-tolerant search, similarity scoring,
and accelerated wildcard and regex text queries using PostgreSQL's `pg_trgm`
extension.

---

## 1. Empowering Fuzzy Search with `pg_trgm`

- **The concept:** `pg_trgm` is an official PostgreSQL contrib extension that
  splits strings into trigrams, which are sequences of three characters.
- **Installation:** Enable it in the database with `CREATE EXTENSION pg_trgm;`.
- **The benefit:** It is ideal for partial matches, misspellings, and approximate
  search behavior, allowing queries to return useful results even when user input
  contains typos.

Example:

```sql
CREATE EXTENSION pg_trgm;
```

---

## 2. Similarity and Distance Operators

- **Similarity matching (`%`):** Use the `%` operator to find rows where the
  target text is similar to the search term.
- **Distance ranking (`<->`):** Use the `<->` operator to order results by
  similarity distance. A lower value means a closer semantic match.

Examples:

```sql
SELECT *
FROM users
WHERE name % 'Jon Doe';

SELECT *, name <-> 'Jon Doe' AS distance
FROM users
ORDER BY distance
LIMIT 10;
```

---

## 3. Supercharging `LIKE` and Regular Expressions

- **B-tree limitation:** Standard B-tree indexes cannot accelerate `LIKE` queries
  with leading wildcards like `LIKE '%search%'`.
- **Trigram solution:** `pg_trgm` enables efficient indexing for wildcard searches
  and can also speed up simple regular expressions with the `~` operator.
- **ILIKE support:** Use trigram indexes to improve case-insensitive wildcard
  queries as well.

Examples:

```sql
SELECT *
FROM products
WHERE description ILIKE '%wireless bluetooth%';

SELECT *
FROM documents
WHERE title ~ 'data[[:space:]]*science';
```

---

## 4. Index Implementation (GIN vs. GiST)

- **Create the index:** Use either `gin_trgm_ops` or `gist_trgm_ops` for trigram
  indexing.
- **Recommended choice:** For large datasets, prefer a GIN index because it
  typically delivers better performance for text search operations.

Examples:

```sql
CREATE INDEX idx_users_name_trgm
ON users USING GIN (name gin_trgm_ops);

CREATE INDEX idx_documents_content_trgm
ON documents USING GIST (content gist_trgm_ops);
```

---

## Example prompt

- "Show me how to use `pg_trgm` to handle typos in search queries and rank
  results by similarity."
- "Explain why `pg_trgm` lets `LIKE '%term%'` queries use an index and how to
  build that index."
- "Demonstrate how to speed up fuzzy regex and ILIKE searches with PostgreSQL
  trigram indexes."
