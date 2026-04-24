---
name: postgresql-native-text-manipulation
description: Guides the implementation of intense text manipulation, advanced searching, and custom data validation natively within PostgreSQL using Regex, pg_trgm, Full-Text Search, and Domains, bypassing external languages entirely.
---

# PostgreSQL Native Text Manipulation

This skill helps developers build powerful text processing, validation, and search
capabilities inside PostgreSQL using only native SQL features and extensions.

---

## 1. Data Type Abstraction and Validation (Domains)

- **Native validation:** PostgreSQL can enforce strict text quality rules without
  external scripting languages.
- **Create a domain:** Use `CREATE DOMAIN` with a `CHECK` constraint to centralize
  validation logic and reuse it across tables.
- **Regex constraints:** Enforce text formats such as email validation using
  POSIX regular expressions.

Example:

```sql
CREATE DOMAIN email_type AS text
CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]+$');
```

- **Usage:** Then declare columns with `email_type` to ensure every inserted value
  meets the same native regex rule.

---

## 2. Pattern Matching with Regular Expressions

- **POSIX operators:** Use `~` for case-sensitive regex matching and `~*` for
  case-insensitive matching.
- **Complex replacements:** Use `regexp_replace()` to transform strings based on
  regex patterns.
- **Pattern extraction:** Use `substring()` with regex to capture and extract
  specific text segments directly in SQL.

Examples:

```sql
SELECT *
FROM documents
WHERE body ~* 'data[[:space:]]+science';

SELECT regexp_replace(phone, '[^0-9]', '', 'g') AS digits_only
FROM users;

SELECT substring(address FROM '([0-9]+)\s+([A-Za-z ]+)')
FROM locations;
```

---

## 3. Fuzzy Searching and `pg_trgm`

- **Enable pg_trgm:** Install the extension and enable it with
  `CREATE EXTENSION pg_trgm;`.
- **Similarity matching:** Use the `%` operator to find strings that are similar
  to a search term, handling typos and misspellings.
- **Distance ranking:** Use the `<->` operator to compute trigram distance, where
  lower values mean higher similarity.
- **Accelerated LIKE and regex:** Build a GIN or GiST index with
  `gin_trgm_ops` to speed up `LIKE '%search%'` queries and complex regex searches.

Examples:

```sql
CREATE INDEX idx_users_name_trgm ON users USING GIN (name gin_trgm_ops);

SELECT *
FROM products
WHERE name % 'accomodation'
ORDER BY similarity(name, 'accommodation') DESC;

SELECT *
FROM phrases
WHERE phrase <-> 'machine learning' < 0.3
ORDER BY phrase <-> 'machine learning';
```

---

## 4. Semantic Full-Text Search (FTS)

- **When to use FTS:** Use Full-Text Search for document-style queries where word
  stems and relevance matter more than exact text matches.
- **Lexemes and dictionaries:** Convert text to lexemes with `to_tsvector()` and
  user search input to a query with `to_tsquery()`.
- **Search execution:** Match vectors and queries with the `@@` operator.
- **Debugging tokenization:** Use `ts_debug()` to inspect how PostgreSQL parses and
  tokenizes the text under a given dictionary.

Examples:

```sql
SELECT *
FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'data & science');

SELECT ts_rank(to_tsvector('english', summary),
               to_tsquery('english', 'analytics')) AS rank,
       title
FROM reports
WHERE to_tsvector('english', summary) @@ to_tsquery('english', 'analytics')
ORDER BY rank DESC;

SELECT *
FROM ts_debug('english', 'PostgreSQL full text search example');
```

---

## Example prompt

- "Show me how to implement email and phone validation in PostgreSQL using domains
  and native regex constraints."
- "Explain how to use `pg_trgm` for fuzzy search and index `LIKE '%term%'`
  queries without PL/Perl or external scripts."
- "Demonstrate semantic full-text search in PostgreSQL and how to debug tokenization
  with `ts_debug()`."
