---
name: database-redis-search-query-syntax

description: Reference for the RediSearch FT.SEARCH query language — term matching, field modifiers, AND/OR/negation, exact phrases, stop words, prefix/infix/suffix and fuzzy matching, numeric/tag filter syntax, and the DIALECT setting that changes how queries parse. Use when writing or debugging an FT.SEARCH query string, as a companion to database-redis-search-indexing (which covers FT.CREATE schema/field types).
---

# RediSearch Query Syntax

You are an expert in the RediSearch query language used by `FT.SEARCH`. When a user is writing, debugging, or getting unexpected results from a search query string, use this skill for the query-string syntax itself — see `database-redis-search-indexing` for schema/field-type design and `FT.AGGREGATE`, and `database-redis-search-ux-features` for highlighting, stemming control, synonyms, spellcheck, autocomplete, and phonetic matching.

## Query vs. Search — know which one is actually needed

- **A query is deterministic**: filtering on a `TAG`/`NUMERIC` field returns exactly the documents matching the criteria, full stop — there's no notion of "how well" something matched.
- **A search is not deterministic**: full-text (`TEXT`) matching returns documents ranked by relevance (term frequency, scoring), not a fixed yes/no set — two searches for slightly different terms can return overlapping but different result sets, and result order matters.
- This distinction should drive field-type choice (see `database-redis-search-indexing`'s `TEXT` vs `TAG` guidance) and how results are consumed — don't treat a relevance-ranked `TEXT` search result as if it were an exhaustive, deterministic filter.

## The DIALECT Setting Changes Query Semantics — Check It First When Results Look Wrong

- **`DIALECT` is a real, silent source of "my query returns the wrong number of results" bugs** — the same query string can parse differently depending on which dialect is active, with no error to indicate the ambiguity was resolved a particular way.
- Check or set it explicitly:

  ```
  FT.CONFIG GET DEFAULT_DIALECT
  FT.CONFIG SET DEFAULT_DIALECT 2
  ```

  Or override per-query without changing the default: append `DIALECT <n>` to the `FT.SEARCH` call.
- **The most impactful practical difference (dialect 1 vs. 2): the scope of a negation on a multi-word phrase.** `@region:"-North America"` under dialect 1 negates the *entire phrase* ("North America" as a unit); under dialect 2 it negates only the first term ("North"), while "America" still has to match. These produce very different result counts (234 vs. 22 in a real dataset) with **no syntax error either way** — this is the dangerous part, since nothing flags that the query was ambiguous.
  - To negate a multi-word phrase unambiguously regardless of dialect, wrap it in parentheses: `@region:"-(North America)"`.
- **Wildcard queries require different syntax depending on dialect** — dialect 2 requires prefixing the pattern with `w` (e.g. `@name:w'*ndura?'`) where dialect 1 does not. Confirm the active dialect before assuming a wildcard pattern's syntax is portable across a codebase that might run against different Redis Stack configurations.
- **When a query returns a surprising result count, check `DIALECT` before assuming the data or the rest of the query is wrong** — it's a cheap, high-value first check given how silent the behavior difference is.

## Term Matching

- **A bare term searches every `TEXT` field in the schema**: `FT.SEARCH country_idx Italy` matches "Italy" in any indexed text field, not just one specific field.
- **Restrict to one field with `@field:term`** — nearly always what's actually wanted once the schema has more than one `TEXT` field, to avoid unintended cross-field matches:

  ```
  FT.SEARCH country_idx '@name:Italy' NOCONTENT
  ```

  Use `NOCONTENT` (or `RETURN 0`) when only the matching key names are needed, not the field values — cuts the response payload for existence-style checks.
- **Search the same term across multiple specific fields** with a pipe inside the field list: `@name|localname:Ital*` — different from a bare unscoped term search, which hits *every* `TEXT` field, not just the ones you name.
- **Exact phrase match requires an extra layer of quoting inside the query string**, not just wrapping the whole query in quotes at the shell level: `'"United Emirates"'` (a quoted phrase within the query) only matches that exact contiguous phrase, while `'United Emirates'` (terms without inner quotes) matches both terms present anywhere, in any order, by default. If "United Arab Emirates" is stored, only the second form matches it.
- **Stop words are excluded from the index by default** (`a`, `is`, `in`, `the`, `and`, `of`, `to`, and similar low-information words) — this is *why* an exact-phrase search containing one can outright error: `'"Trinidad and Tobago"'` fails with a syntax error because `and` was never indexed, not because the phrase syntax is wrong. Recreate the index with `STOPWORDS 0` if the domain's data genuinely needs these words searchable (e.g. exact legal/official names that include them) — this is a schema-time decision, not something fixable at query time.

## Boolean Logic

- **AND (intersection) is implicit — space-separated terms both must match, order-independent by default**:

  ```
  FT.SEARCH country_idx 'United Emirates Arab' RETURN 0
  ```

  - Add `INORDER` to require the terms appear in the given order.
  - Add `SLOP <n>` to bound how many non-matching terms may appear *between* the matched terms (`SLOP 0` with `INORDER` effectively demands adjacency; increase it to allow intervening words).
  - Combine multiple field-scoped clauses by just placing them next to each other — `@region:europe @headofstate:carlo` is an implicit AND across two different fields.
- **OR (union) uses the pipe (`|`)**, either as separate clauses or the more compact form inside one field reference:

  ```
  FT.SEARCH country_idx "@region:europe | @region:america" NOCONTENT LIMIT 0 100
  FT.SEARCH country_idx "@region:(europe|america)" NOCONTENT LIMIT 0 100
  ```

  These compose with AND normally — e.g. `"@name:(Spain|Italy) @region:'Southern Europe'"` is "(Spain OR Italy) AND Southern Europe."
- **Negate a clause with `-` on the field modifier.** This works both combined with other clauses and as a standalone "everything except" query:

  ```
  FT.SEARCH country_idx "@region:europe -@region:'Southern Europe'" RETURN 1 name LIMIT 0 100
  FT.SEARCH country_idx "-@region:'Southern Europe'" RETURN 1 name LIMIT 0 100
  ```

  Remember the dialect-dependent negation-scope gotcha above when the negated value is a multi-word phrase.

## Pattern Matching

- **Prefix/infix/suffix matching uses `*`**, case-insensitively:

  ```
  @name:'hond*'    -- prefix: starts with "hond"
  @name:'*dura*'   -- infix: contains "dura" anywhere
  @name:'*uras'    -- suffix: ends with "uras"
  ```

- **Single-character wildcard matching uses `?`** (and `*` can still mean "zero or more of any character" within the same pattern): `@name:'*ndura?'`. Remember dialect 2 requires the `w` prefix noted above for this to parse as a wildcard pattern rather than literal characters.
- **Fuzzy (approximate) matching uses `%` delimiters, repeated to set the allowed Levenshtein distance** — the number of single-character edits (insert/delete/substitute) tolerated between the search term and a match:

  ```
  @name:%hondras%     -- distance 1 (one typo tolerated)
  @name:%%hondur%%    -- distance 2
  ```

  Use this for user-entered search terms where typos are expected (a search box), not for exact identifiers/codes where an approximate match would be actively wrong.

## Numeric and Tag Filter Syntax

(Field-type setup is covered in `database-redis-search-indexing` — this is the query-string syntax for using them.)

- **Numeric range**: either inline as a field modifier or as a separate `FILTER` argument — equivalent, pick whichever reads more clearly for the query being built:

  ```
  @population:[50000000 +inf]
  FILTER population 50000000 +inf
  ```

  `-inf`/`+inf` express unbounded ends. Bounds are inclusive by default; wrap a bound in parentheses to make it exclusive: `FILTER population (39441700 +inf` excludes exactly `39441700`.
- **Tag exact-match and union**: braces select exact tag values, and `|` inside the braces unions them:

  ```
  @governmentform:{monarchy}
  @governmentform:{monarchy|republic}
  ```

- **Combine tag and numeric filters with sorting in one call** rather than filtering further client-side:

  ```
  FT.SEARCH country_idx '@governmentform:{republic} @population:[-inf 100000]' RETURN 2 name surfacearea LIMIT 0 3 SORTBY surfacearea ASC
  ```

## Code Examples

```python
from redis import asyncio as aioredis
from redis.commands.search.query import Query

client = aioredis.from_url("redis://localhost")

async def exact_field_match(term: str) -> list[str]:
    query = Query(f"@name:{term}").no_content()
    result = await client.ft("country_idx").search(query)
    return [doc.id for doc in result.docs]

async def fuzzy_search(term: str, distance: int = 1) -> list[str]:
    """distance 1 -> %term%, distance 2 -> %%term%%, etc."""
    fuzzy_term = ("%" * distance) + term + ("%" * distance)
    query = Query(f"@name:{fuzzy_term}").return_fields("name")
    result = await client.ft("country_idx").search(query)
    return [doc.name for doc in result.docs]

async def population_range_excluding_region(min_population: int, excluded_region: str) -> list[dict]:
    query_str = f'-@region:"{excluded_region}" @population:[{min_population} +inf]'
    query = Query(query_str).return_fields("name", "population")
    result = await client.ft("country_idx").search(query)
    return [{"name": d.name, "population": d.population} for d in result.docs]

async def government_form_union(*forms: str) -> list[str]:
    tag_expr = "|".join(forms)
    query = Query(f"@governmentform:{{{tag_expr}}}").return_fields("name")
    result = await client.ft("country_idx").search(query)
    return [d.name for d in result.docs]
```
