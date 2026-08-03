---
name: database-redis-json-documents

description: Guides the agent on storing and querying nested/hierarchical data with Redis Stack's JSON document type (JSON.SET/JSON.GET, JSONPath array/filter/recursive-descent syntax, JSON.ARRAPPEND/JSON.OBJKEYS), choosing JSON over a flat Hash, and indexing JSON-specific shapes (array elements, multi-value fields, JSONPath in RETURN) with RediSearch.
---

# Redis Stack JSON Documents

You are an expert in Redis Stack's JSON document type. When a user needs to store data that's nested, hierarchical, or contains arrays — not a flat set of scalar fields — guide them toward the native JSON type instead of a Hash with serialized sub-values.

- **Choose JSON over Hash when the data has structure a flat field list can't represent**: nested objects, arrays, or a mix of types at different depths. A Hash's fields are flat, single-level key→scalar pairs — modeling a nested object in a Hash means either flattening it (losing structure) or serializing a sub-object into one field's string value (losing queryability into that sub-value). JSON documents store the structure natively, up to 128 levels deep, including objects, arrays, and geographical locations at any level.
- **Prefer Hash when the data is genuinely flat** — a JSON document carries more overhead than a Hash for simple field/value objects with no nesting, so don't default to JSON just because it's more flexible; pick based on the data's actual shape.
- **Store and update documents with `JSON.SET`, addressing any depth with a JSONPath**, not just the document root:

  ```
  JSON.SET city:653 $ '{"Name":"Madrid","CountryCode":"ESP","District":"Madrid","Population":2879052}'
  ```

  Because the path argument isn't limited to `$` (root), you can update or fetch a single nested field without reading/rewriting the whole document — this is a real advantage over a Hash-with-serialized-JSON-string approach, where any change requires deserializing, mutating, and re-serializing the entire blob client-side.
- **Retrieve only the fields needed with `JSON.GET` and a JSONPath**, instead of always fetching and parsing the whole document client-side:

  ```
  JSON.GET city:653 $.Name
  JSON.GET city:653 $.Name $.CountryCode
  ```

  This is the same bandwidth-discipline principle as `HMGET` over `HGETALL` for Hashes, applied to nested paths.
- **JSON documents are indexable and searchable exactly like Hashes** via RediSearch — the only difference is `ON JSON` instead of `ON HASH` in `FT.CREATE`, and JSONPath field paths (`$.Name AS name TEXT`) instead of flat field names. See `database-redis-search-indexing` for the full indexing/query syntax; don't treat JSON storage and JSON search as separate concerns to solve independently.
- **The document is stored internally as a tree**, which is what makes path-scoped reads/writes efficient — this is a genuine structural advantage over storing a JSON string in a plain Redis string value (`SET key '{"...": ...}'`), which requires reading and reparsing the entire string for any access, with no server-side path addressing at all.

## JSONPath Syntax Beyond a Bare Field Path

JSONPath supports real querying into a document's structure, not just "give me this one named field" — reach for these instead of fetching the whole document and picking it apart in application code:

- **Array indexing** with `[]`: `$.books[0]` gets the first element of the `books` array.
- **Filter expressions** select array elements matching a condition — `?()` introduces the filter, `@` refers to the current element being tested:

  ```
  JSON.GET author:1 '$.books[?(@.isbn=="8845294021")]'
  ```

  Chain a `.` after the filter to drill into a specific field of the matched element(s), rather than fetching the whole matched object when only one field is needed: `'$.books[?(@.isbn=="8845294021")].title'`.
- **Wildcard (`*`) iterates every element/field at that level** — `$.books.*.title` returns every book's title as an array, in one call, without knowing the array's length in advance.
- **Recursive descent (`..`) searches at every depth, not just one level** — `$..genre.*` collects every `genre` value anywhere in the document (both the author's own `genre` array and each book's `genre` array), merged into one result, instead of needing a separate query per level.
- **Mutate arrays in place with `JSON.ARRAPPEND`**, rather than reading the array, appending client-side, and writing the whole thing back:

  ```
  JSON.ARRAPPEND author:1 $.books '{"isbn":"886836204X","title":"Cujo"}'
  ```

- **Inspect an object's keys with `JSON.OBJKEYS`** when the shape of a nested object isn't known ahead of time (e.g. dynamic/user-defined attributes) — `JSON.OBJKEYS author:1 $.books[1]`.
- **Pretty-print for human inspection with `INDENT`/`NEWLINE`/`SPACE` on `JSON.GET`**, combined with `redis-cli --raw` — useful for debugging, not something application code should parse (application code should ask for exactly the path/fields it needs instead of pretty-printing and re-parsing).

## Indexing Array Elements and Multi-Value Fields

JSON's nested/array structure means RediSearch indexing has a few JSON-specific capabilities that don't apply to flat Hash fields — see `database-redis-search-indexing` for general schema/field-type design; this is what's specific to indexing *into* a JSON document's structure.

- **Index a field inside every element of an array with a wildcard path in the schema** — `$.books[*].year AS year NUMERIC SORTABLE` makes every book's `year`, across every author, searchable and sortable as one logical field, without flattening books into separate top-level documents:

  ```
  FT.CREATE author_idx ON JSON PREFIX 1 author:
      SCHEMA $.name AS name TEXT
             $.genre AS genre TAG
             $.books[*].year AS year NUMERIC SORTABLE
             $.books[*].isbn AS isbn TAG
             $.books[*].title AS title TEXT SORTABLE
  ```

- **Multi-value indexing: a single JSONPath that resolves to *multiple* values (an array, or a recursive-descent match spanning several locations) can be indexed as one field**, and every one of those values becomes independently searchable. This only works for JSONPath expressions that evaluate to an array — a scalar path indexes as a single value as usual. Combine this with recursive descent to unify genre tags declared at both the author level and the per-book level into one searchable field:

  ```
  SCHEMA $..genre.* AS genre TAG
  ```

  ```
  FT.SEARCH author_idx "@genre:{'romance novel'}" RETURN 1 name
  ```

  This finds an author via a genre tag that only exists on one of their *books*, not on the author record itself — something a single flat field per document couldn't express. Supported for `TEXT`, `TAG`, `GEO`, `NUMERIC`, and `VECTOR` field types.
- **`RETURN` accepts JSONPath expressions (with `AS` for a friendly output name), not just flat field names** — pull exactly the nested value(s) that matched, from wherever in the document they live, without a second round trip to re-fetch and re-parse the full document:

  ```
  FT.SEARCH author_idx '@isbn:{8806216465}' RETURN 6
      $.name AS Author '$.books[?(@.isbn=="8806216465")].title' AS Title
  ```

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def save_city_json(city_id: int, city: dict) -> None:
    """Store a nested object natively — no flattening or serialization needed."""
    await client.json().set(f"city:{city_id}", "$", city)

async def get_city_name(city_id: int) -> str:
    """Path-scoped read: only the needed field crosses the network."""
    result = await client.json().get(f"city:{city_id}", "$.Name")
    return result[0] if result else None

async def update_population(city_id: int, new_population: int) -> None:
    """Update one nested field in place — no read-modify-write round trip."""
    await client.json().set(f"city:{city_id}", "$.Population", new_population)

# Example of the kind of nested structure JSON handles that a Hash can't:
city_with_nested_data = {
    "Name": "Amsterdam",
    "CountryCode": "NLD",
    "Population": 731200,
    "Districts": ["Noord-Holland", "Centrum"],
    "Location": {"lat": 52.3676, "lon": 4.9041},
}

async def append_book(author_id: int, book: dict) -> int:
    """JSON.ARRAPPEND: mutate the array in place, no read-modify-write."""
    result = await client.json().arrappend(f"author:{author_id}", "$.books", book)
    return result[0]

async def all_book_titles(author_id: int) -> list[str]:
    """Wildcard path: every title in the array, in one call."""
    return await client.json().get(f"author:{author_id}", "$.books.*.title")

async def all_genres_any_level(author_id: int) -> list[str]:
    """Recursive descent: genre values from both the author and every book, merged."""
    return await client.json().get(f"author:{author_id}", "$..genre.*")

async def book_by_isbn(author_id: int, isbn: str) -> dict | None:
    """Filter expression: select the array element(s) matching a condition."""
    result = await client.json().get(f"author:{author_id}", f'$.books[?(@.isbn=="{isbn}")]')
    return result[0] if result else None
```
