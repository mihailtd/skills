---
name: database-redis-search-indexing

description: Guides the agent on creating and querying RediSearch secondary indexes over Hash or JSON documents in Redis Stack — field types (TEXT, TAG SORTABLE, NUMERIC SORTABLE), combined filters, the FT.AGGREGATE pipeline (GROUPBY/REDUCE/APPLY/FILTER/SORTBY, cursor-based pagination) for analytics and faceted search, scoped/ephemeral/TEMPORARY indexes, and safely evolving an index in production with FT.ALTER or an FT.ALIAS zero-downtime swap — as the replacement for hand-rolled indexes or full-keyspace scans.
---

# Redis Stack Search and Indexing (RediSearch)

You are an expert in Redis Stack's search module (RediSearch). When a user needs to filter, full-text search, sort, or aggregate Redis data on non-key attributes, and Redis Stack is available, guide them to create a proper index instead of scanning the keyspace or maintaining a hand-rolled index (see `database-redis-manual-secondary-indexing` for why to avoid that path once RediSearch is an option). This skill covers `FT.CREATE` schema design and `FT.AGGREGATE` — for the `FT.SEARCH` query-string syntax itself (boolean logic, exact phrases, wildcards, fuzzy matching, the DIALECT gotcha), see `database-redis-search-query-syntax`; for search-experience features (highlighting, stemming control, synonyms, spellcheck, autocomplete, phonetic matching), see `database-redis-search-ux-features`; for how full-text results get ranked (TF-IDF, SCORE_FIELD, SCORER, WEIGHT), see `database-redis-search-relevance-scoring`.

- **Create an index with `FT.CREATE` against a key prefix, not against individual keys.** The index automatically covers every existing and future key matching the prefix — there's no manual re-indexing step when data changes; the server keeps the index synchronously consistent with every write to a matching key.

  ```
  FT.CREATE city_idx ON HASH PREFIX 1 city:
  SCHEMA Name AS name TEXT
         CountryCode AS countrycode TAG SORTABLE
         Population AS population NUMERIC SORTABLE
  ```

- **Choose the field type based on the query shape you actually need, not just the data type:**
  - **`TEXT`** — full-text search with tokenization and relevance scoring. Use for free-text fields users search by keyword (names, descriptions), not for exact-match lookups.
  - **`TAG`** — exact-match/categorical filtering (e.g. a country code, a status enum). Cheaper and more precise than `TEXT` for values that are never searched as free text. Add `SORTABLE` when you'll also need to sort results by this field.
  - **`NUMERIC`** — range queries (`<`, `>`, `BETWEEN`). Add `SORTABLE` to sort by it efficiently without a separate pass.
  - Getting the field type wrong doesn't error, but produces the wrong query semantics — e.g. indexing a country code as `TEXT` would tokenize and stem it like prose instead of matching it exactly.
- **The same schema/query syntax works identically for JSON documents** — swap `ON HASH` for `ON JSON`, and express field paths with JSONPath (`$.Name AS name TEXT`) instead of flat Hash field names. This means the choice between Hash and JSON storage (see `database-redis-json-documents`) doesn't constrain how you search the data.
- **Combine filters, and limit returned fields, in one `FT.SEARCH` call**, rather than fetching whole documents and filtering client-side:

  ```
  FT.SEARCH city_idx '@name:Madrid @countrycode:{ESP}' RETURN 1 name
  FT.SEARCH city_idx '@countrycode:{ESP}' FILTER population 2000000 +inf RETURN 1 name
  FT.SEARCH city_idx '@countrycode:{ESP}' RETURN 1 name SORTBY name LIMIT 0 3
  ```

  `RETURN` limits network transfer to just the fields the caller needs (the same bandwidth discipline as `HMGET` over `HGETALL`); `SORTBY`/`LIMIT` push pagination and ordering to the server instead of the client.
- **Use `FT.AGGREGATE` for GROUP BY-style analytics** (summing/counting/averaging across groups) instead of pulling all matching documents back and aggregating in application code:

  ```
  FT.AGGREGATE city_idx * GROUPBY 1 @countrycode
      REDUCE SUM 1 @population AS sum
      SORTBY 2 @sum DESC LIMIT 0 3
  ```

  This is the direct analogue of `SELECT country, SUM(population) FROM city GROUP BY country ORDER BY sum DESC LIMIT 3` — reach for it whenever the requirement is "compute an aggregate per group," not a full result set to aggregate elsewhere.
- **`FT.AGGREGATE` is a pipeline, not a single clause — stages run in the order you write them**, and later stages see the output of earlier ones, not the original documents:
  - **`APPLY "<expr>" AS <name>`** computes a derived value per row/group from an expression (arithmetic, functions like `floor()`, `timefmt()` for formatting a Unix timestamp with `strftime`-style codes) — the result becomes a new field usable by any later stage in the same pipeline:

    ```
    FT.AGGREGATE country_idx *
        GROUPBY 1 @continent
        REDUCE SUM 1 @population AS tot_pop
        REDUCE SUM 1 @surfacearea AS tot_sur
        APPLY "floor(@tot_pop/@tot_sur)" AS people_per_km2
        SORTBY 2 @people_per_km2 DESC
        LIMIT 0 1
    ```

  - **`FILTER "<expr>"` filters *after* grouping/reducing/applying**, not before — this is what makes it different from a `WHERE`-style pre-filter or the query string itself: it operates on the computed group-level fields (`@tot_pop`, `@people_per_km2`), which don't exist until the earlier pipeline stages produced them:

    ```
    FT.AGGREGATE country_idx *
        GROUPBY 1 @continent
        REDUCE SUM 1 @population AS tot_pop
        REDUCE SUM 1 @surfacearea AS tot_sur
        APPLY "floor(@tot_pop/@tot_sur)" AS people_per_km2
        FILTER "@continent=='Europe'"
    ```

  - **`SORTBY` in a pipeline can sort by multiple fields at once**, each with its own direction — give the argument count as `2 * (number of sort fields)`: `SORTBY 4 @update DESC @creation DESC` sorts by `update` descending, then `creation` descending as a tiebreaker.
  - **`REDUCE MAX 1 @field AS alias` combined with `GROUPBY` is the aggregation-pipeline way to get "the latest/highest per group"** — e.g. `GROUPBY 1 @author REDUCE MAX 1 @update AS last_updated` gets each author's most recent update timestamp in one call, no per-author follow-up query needed.
- **Paginate large aggregations with a server-side cursor (`WITHCURSOR`), not repeated `LIMIT` offsets.** A `LIMIT`-paginated aggregation re-runs the *entire* pipeline on every page request; a cursor computes the pipeline once, holds the full result set server-side, and hands it to the client in batches:

  ```
  FT.AGGREGATE city_idx * WITHCURSOR COUNT 5 GROUPBY 1 @countrycode
  ```

  The response includes a cursor ID; keep calling `FT.CURSOR READ <index> <cursor_id>` (optionally overriding the batch size with `COUNT`) until it returns cursor ID `0`, meaning the iteration is exhausted. An idle cursor is freed automatically after 300 seconds by default (tune with `MAXIDLE`), on full consumption, or on demand via `FT.CURSOR DEL` — but a cursor left open holds server-side resources for as long as it's alive, so don't open one and abandon it; always either drain it or explicitly delete it.
- **RediSearch supports more query types beyond exact/range/full-text** — phonetic matching, autocomplete suggestions, geo search (a `GEO` field type, queryable with `[lon lat radius unit]` — see `database-redis-geospatial` for when to use this versus the standalone `GEOADD`/`GEOSEARCH` commands), and spellchecking are all available; if a requirement sounds like one of these, look for the matching RediSearch feature rather than assuming it needs to be built on top of `TEXT`/`TAG`/`NUMERIC` primitives.
- **Faceted search is `FT.AGGREGATE ... GROUPBY` on a `TAG` field, not a separate feature.** "Search by keyword, then let the user filter/browse by attribute (category, brand, color)" — the classic retail-site facet UI — is just an aggregation query counting documents per tag value, combined with a full-text/exact-match filter in the same call:

  ```
  FT.AGGREGATE product_idx * GROUPBY 1 @color REDUCE COUNT 0 AS items SORTBY 2 @items DESC
  FT.AGGREGATE product_idx '@name:polo @brand:{lacoste}' GROUPBY 1 @color REDUCE COUNT 0 AS items SORTBY 2 @items DESC
  ```

  Model every attribute a user might want to facet/filter by as a `TAG SORTABLE` field on the schema up front — retrofitting a new facet later means re-indexing, not just changing a query.
  - **When one document can carry *multiple* values for the same facet** (e.g. laundry care instructions like `"machinewash,iron"` on one product), index the field as a `TAG` with an explicit `SEPARATOR` and split it back into individual values in the aggregation pipeline with `APPLY "split(@field)" AS <alias>` before grouping — grouping on the raw multi-value tag string would treat `"machinewash,iron"` as one distinct value instead of counting toward both `machinewash` and `iron`:

    ```
    FT.CREATE product_idx PREFIX 1 item: SCHEMA laundry TAG SEPARATOR "," SORTABLE

    FT.AGGREGATE product_idx *
        APPLY "split(@laundry)" AS laundry_recommend
        GROUPBY 1 @laundry_recommend
        REDUCE COUNT 0 AS num_per_ctg
        SORTBY 2 @num_per_ctg ASC
    ```
- **Create a narrow, temporary index (an "ephemeral index") instead of indexing an entire dataset, when only a small, session-scoped slice of it needs to be searchable.** Point `FT.CREATE` at a prefix specific to one user/session (e.g. `user:241245`, not `user:`), so the index only ever covers that slice — and drop it (optionally with the underlying documents) once the session ends:

  ```
  FT.CREATE user:241245:idx ON HASH PREFIX 1 user:241245
      SCHEMA Name AS name TEXT Id AS id TEXT Quantity AS quantity NUMERIC

  FT.SEARCH user:241245:idx toner RETURN 1 name

  FT.DROPINDEX user:241245:idx DD
  ```

  This is the right call for search scoped to one active user's data (a shopping cart, a search history) where indexing the *entire* dataset — including inactive/archived users' data that will never be searched — would waste memory and indexing overhead for no benefit. Don't default to one global index when the actual search scope is always a narrow, known slice.
  - **`FT.CREATE ... TEMPORARY <seconds>` automates the same lifecycle** — the index (and, unlike a manual `FT.DROPINDEX`, only the index — the underlying documents are untouched) is dropped automatically after that many seconds of inactivity, with no explicit cleanup call needed on session end. Prefer this over a manual create/drop pair when there's no reliable single "session ended" hook to call `FT.DROPINDEX` from (e.g. the client might just disconnect without a clean logout).

## Evolving an Index in Production

Schema changes are routine, but a RediSearch index isn't automatically kept in sync with a schema change the way document storage is (Hash/JSON are schemaless — you can write a new field any time) — the *index* needs to be told about new searchable attributes.

- **`FT.ALTER` adds a new field to an existing index and rescans existing documents to populate it** — the least disruptive option when the change is purely additive:

  ```
  FT.ALTER city_idx SCHEMA ADD District AS district TAG
  ```

  **`FT.ALTER` cannot remove or modify an existing field** — only add. If a field needs to change type or be removed, the index has to be dropped and recreated, which is exactly the disruption the next technique avoids.
- **`FT.ALIASADD`/`FT.ALIASUPDATE` decouple the index name your application uses from the index actually being queried, enabling a zero-downtime schema migration.** Build a new index with the corrected/extended schema under a new name, then repoint the alias your application already uses — the application never needs to know the underlying index name changed:

  ```
  FT.ALIASADD city_alias_idx city_idx        -- app queries city_alias_idx, not city_idx, from day one
  -- later: build a replacement index with the new schema
  FT.CREATE city_new_idx ON HASH PREFIX 1 city:
      SCHEMA Name AS name TEXT CountryCode AS countrycode TAG SORTABLE
             District AS district TAG Population AS population NUMERIC SORTABLE
  FT.ALIASUPDATE city_alias_idx city_new_idx  -- app's queries now transparently hit city_new_idx
  ```

  This is the RediSearch equivalent of a database view used to rename/repoint a table without breaking callers — the payoff only materializes if the application was pointed at the *alias* from the start, so establish that convention early rather than retrofitting it once a breaking index change is already needed.
- **Prefer `FT.ALTER` for simple additive changes; reach for the alias-swap technique when the change is removal/retyping, or when the migration needs to be validated (build + spot-check the new index) before cutting traffic over.**

## Useful Utility/Debugging Commands

- **`FT._LIST`** — list every index currently defined on the server.
- **`FT.TAGVALS <index> <field>`** — list every distinct value currently stored for a `TAG` field, useful for populating a filter UI's option list or sanity-checking what's actually in the index.
- **`FT.EXPLAINCLI <index> "<query>"`** — show the execution plan for a query (which clauses become which internal operations, e.g. `INTERSECT`/`TAG`/`NUMERIC` nodes) — check this when a query is slower than expected, the same instinct as `EXPLAIN` in a relational database.

## Code Examples

```python
from redis import asyncio as aioredis
from redis.commands.search.field import TextField, TagField, NumericField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType
from redis.commands.search.query import Query
from redis.commands.search.aggregation import AggregateRequest
import redis.commands.search.reducers as reducers

client = aioredis.from_url("redis://localhost")

async def create_city_index() -> None:
    schema = (
        TextField("$.Name", as_name="name"),
        TagField("$.CountryCode", as_name="countrycode", sortable=True),
        NumericField("$.Population", as_name="population", sortable=True),
    )
    await client.ft("city_idx").create_index(
        schema,
        definition=IndexDefinition(prefix=["city:"], index_type=IndexType.JSON),
    )

async def spanish_cities_over(min_population: int) -> list[str]:
    query = (
        Query(f"@countrycode:{{ESP}} @population:[{min_population} +inf]")
        .return_fields("name")
    )
    result = await client.ft("city_idx").search(query)
    return [doc.name for doc in result.docs]

async def top_countries_by_population(limit: int = 3) -> list[dict]:
    req = (
        AggregateRequest("*")
        .group_by("@countrycode", reducers.sum("@population").alias("sum"))
        .sort_by("-@sum")
        .limit(0, limit)
    )
    result = await client.ft("city_idx").aggregate(req)
    return [dict(zip(row[0::2], row[1::2])) for row in result.rows]

async def facet_counts_by_color() -> dict:
    """Faceted search: count matching documents per TAG value."""
    req = AggregateRequest("*").group_by("@color", reducers.count().alias("items")).sort_by("-@items")
    result = await client.ft("product_idx").aggregate(req)
    return {row[1]: row[3] for row in result.rows}

async def create_ephemeral_user_index(user_id: str) -> None:
    """Index scoped to one user's data only — cheap to create and drop per session."""
    schema = (TextField("Name", as_name="name"), TextField("Id", as_name="id"))
    await client.ft(f"user:{user_id}:idx").create_index(
        schema,
        definition=IndexDefinition(prefix=[f"user:{user_id}"], index_type=IndexType.HASH),
    )

async def drop_ephemeral_user_index(user_id: str) -> None:
    """Drop the index (and, with delete_documents=True, the underlying keys) on logout."""
    await client.ft(f"user:{user_id}:idx").dropindex(delete_documents=True)

async def population_density_by_continent() -> dict:
    """Pipeline: two REDUCEs feed an APPLY expression, then sort by the derived field."""
    req = (
        AggregateRequest("*")
        .group_by(
            "@continent",
            reducers.sum("@population").alias("tot_pop"),
            reducers.sum("@surfacearea").alias("tot_sur"),
        )
        .apply(people_per_km2="floor(@tot_pop/@tot_sur)")
        .sort_by("-@people_per_km2")
        .limit(0, 1)
    )
    result = await client.ft("country_idx").aggregate(req)
    return dict(zip(result.rows[0][0::2], result.rows[0][1::2]))

async def iterate_all_countrycodes_via_cursor(index: str, batch_size: int = 5) -> list[str]:
    """Cursor pagination: pipeline runs once, results are drained in batches.
    Uses raw commands — confirm your redis-py version's high-level cursor
    wrapper (AggregateRequest.cursor / cursor_read) before relying on it,
    the API around WITHCURSOR has shifted across client versions."""
    reply = await client.execute_command(
        "FT.AGGREGATE", index, "*", "WITHCURSOR", "COUNT", batch_size, "GROUPBY", 1, "@countrycode",
    )
    rows, cursor_id = reply[0], reply[1]
    all_rows = rows[1:]  # first element is the result count
    while cursor_id:
        reply = await client.execute_command("FT.CURSOR", "READ", index, cursor_id, "COUNT", batch_size)
        rows, cursor_id = reply[0], reply[1]
        all_rows += rows[1:]
    return all_rows

async def add_district_field_to_index() -> None:
    """FT.ALTER: additive-only schema change, rescans existing documents."""
    await client.execute_command("FT.ALTER", "city_idx", "SCHEMA", "ADD", "District", "AS", "district", "TAG")

async def swap_index_via_alias(alias: str, new_index: str) -> None:
    """Zero-downtime cutover — the application only ever queries `alias`."""
    await client.ft().aliasupdate(alias, new_index)
```
