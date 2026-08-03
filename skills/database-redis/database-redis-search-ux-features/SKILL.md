---
name: database-redis-search-ux-features

description: Guides the agent on RediSearch's search-experience features beyond raw filtering — result highlighting/summarizing, stemming control (VERBATIM/NOSTEM), synonym groups, spellcheck suggestions, auto-complete (with fuzzy matching), and phonetic matching — for building a search UI users can actually use with typos, partial input, and unfamiliar terminology.
---

# RediSearch Search UX Features

You are an expert in RediSearch's search-experience features. When a user is building a search interface (not just a filtered data query) and needs to handle typos, partial input, unfamiliar terminology, or wants to present results the way a search engine does (highlighted excerpts, suggestions), guide them to the matching built-in RediSearch feature instead of reimplementing it in application code.

## Highlighting and Summarizing Results

- **`HIGHLIGHT` wraps matched search terms in the tags you specify**, so the frontend doesn't need to re-implement match detection to bold/mark the terms that made a document match:

  ```
  FT.SEARCH kb_idx "delete keys" RETURN 1 content HIGHLIGHT TAGS "<b>" "</b>" LIMIT 0 1
  ```

  The tags are literal text you choose — `<b>`/`</b>` for HTML rendering, or any other markup appropriate to how the result will actually be displayed (not necessarily HTML).
- **`SUMMARIZE` (combined with `HIGHLIGHT`) returns a short excerpt around the matching terms instead of the full field value** — the right choice whenever a result list shows a preview/snippet rather than full content, since it avoids transferring (and requiring the client to truncate) entire document bodies just to show a few lines of context.
- **Highlighted terms include stemmed variants, not just the literal search term** — searching "delete" also highlights "deleting," "deletes," etc. wherever they appear, because of stemming (see below). This is usually what you want for a natural search experience, but don't be surprised when a highlighted term isn't a literal string match.

## Stemming — On by Default, With Escape Hatches

- **RediSearch stems `TEXT` fields by default**, using the Snowball stemmer (covers most European languages) — this is *why* searching "delete" also matches documents containing "deleting," "deletes," "deleted": the index stores base word forms, not just literal terms.
- **Disable stemming for one query with `VERBATIM`** when the search needs to match only the literal term(s), not their variants — e.g. matching an exact product code or technical term that happens to look like an inflectable word.
- **Disable stemming for one field permanently with `NOSTEM` at index-creation time** when that field's values should never be treated as prose to be stemmed — e.g. a title field that's really more of an identifier.
- Choosing between these is about scope: `VERBATIM` is a per-query override (the field stays stemmed for everyone else), `NOSTEM` is a per-field, permanent decision baked into the schema.

## Synonyms

- **A synonym group makes different words match the same query**, closing the gap between the vocabulary your data uses and the vocabulary users actually search with — e.g. a document says "delete" but a user searches "removal":

  ```
  FT.SYNUPDATE kb_idx del_group delete deletion remove removal purge
  ```

  Every term in the group becomes interchangeable for search purposes — searching any one of them matches documents containing any other.
- **This is an operational, iterative feature, not a one-time setup.** Track search queries that return zero results (e.g. accumulate them in a Redis Set/List/Sorted Set, with a Sorted Set letting you rank by how often each failed term recurs), review them periodically, and add synonym groups for the recurring gaps. Treat a growing list of zero-result searches as a concrete, measurable signal of where the search vocabulary doesn't match user vocabulary — don't guess at synonym groups in the abstract.

## Spellcheck

- **`FT.SPELLCHECK` proposes corrections for query terms that don't closely match anything in the index**, scored by confidence, without requiring the terms to be searched first:

  ```
  FT.SPELLCHECK kb_idx "Reds Stak" DISTANCE 1
  ```

  A term already close enough to an indexed term (or an exact match) returns no suggestion for that term — an empty response for the whole query means every term looked fine as typed.
- **Use this as a pre-search step for a "did you mean...?" prompt, or to auto-correct transparently before running the actual `FT.SEARCH`** — the choice between showing a suggestion and silently correcting is a UX call, not a technical constraint; both are supported by feeding the spellcheck output back into a corrected query.
- **`DISTANCE` bounds how many character edits (Levenshtein distance) a suggestion is allowed to be from the original term** — the same edit-distance concept as fuzzy matching in `database-redis-search-query-syntax`, just applied to suggest corrections instead of directly matching approximate terms.
- **The spellcheck dictionary is managed separately from the index's document content**, via `FT.DICTADD`/`FT.DICTDEL`/`FT.DICTDUMP` — use these to tune precision (add domain-specific terms that would otherwise look like typos, remove terms that generate bad suggestions) rather than accepting whatever the index alone infers as "correct."

## Auto-Completion

- **Auto-complete is a separate dictionary structure (`FT.SUGADD`/`FT.SUGGET`), not derived automatically from an index** — build it deliberately from the specific set of values you want to suggest (e.g. tag names, product categories), not the entire body of searchable text:

  ```
  FT.SUGADD tag_suggestions "scalability" 1
  FT.SUGGET tag_suggestions "scala"
  ```

  The trailing number in `SUGADD` is a weight — use it to bias which suggestions rank higher for a given prefix (e.g. popularity-weighted suggestions).
- **A plain `SUGGET` requires the typed prefix to match the start of a suggestion exactly** — a typo in the prefix itself (not just later in the word) returns nothing. Add `FUZZY` to tolerate typos in the prefix too:

  ```
  FT.SUGGET tag_suggestions "scalbi" FUZZY
  ```

- **`FUZZY` costs more per lookup** (it traverses the suggestion dictionary rather than doing a direct prefix lookup) — negligible for a dictionary sized for realistic autocomplete use (hundreds to low thousands of entries), but worth being aware of if the suggestion dictionary is unusually large. Don't reach for `FUZZY` by default on every keystroke if the dictionary is large and a plain prefix match already serves most users well; reserve it for when typo tolerance is a real, measured need.

## Phonetic Matching

- **Phonetic matching finds results based on how a term *sounds*, not how it's spelled** — indexed by adding `PHONETIC <language-code>` to a `TEXT` field at index-creation time (e.g. `PHONETIC "dm:en"` for English, using the Double Metaphone algorithm):

  ```
  FT.CREATE city_phonetic_idx ON HASH PREFIX 1 "city:" SCHEMA Name AS name TEXT PHONETIC "dm:en"
  FT.SEARCH city_phonetic_idx Rawma RETURN 1 Name
  ```

  This matches "Roma" against a search for "Rawma" — a misspelling close enough phonetically, even though it isn't within a small Levenshtein edit distance of the correct spelling (fuzzy matching wouldn't necessarily catch it; phonetic matching is a different, complementary tool for a different class of misspelling).
- **This is a per-field, index-time decision, not a query-time option** — a field needs `PHONETIC` in its schema definition before phonetic matches against it are possible; there's no way to phonetically search a field that wasn't indexed that way, without rebuilding the index (see `database-redis-search-indexing`'s index-evolution section for how to do that without downtime).
- **Use phonetic matching specifically for name-like fields** (people, places, brands) where users are likely to guess a spelling rather than know it exactly — it's a poor fit for structured/coded values (IDs, SKUs) where phonetic similarity has no real meaning.

## Code Examples

```python
from redis import asyncio as aioredis
from redis.commands.search.query import Query

client = aioredis.from_url("redis://localhost")

async def search_with_highlights(index: str, query_str: str, field: str) -> list[dict]:
    query = (
        Query(query_str)
        .return_fields(field)
        .highlight(fields=(field,), tags=("<b>", "</b>"))
    )
    result = await client.ft(index).search(query)
    return [{"id": d.id, field: getattr(d, field)} for d in result.docs]

async def add_synonym_group(index: str, group_id: str, *terms: str) -> None:
    await client.ft(index).synupdate(group_id, False, *terms)

async def spellcheck(index: str, query_str: str, distance: int = 1) -> dict:
    return await client.ft(index).spellcheck(query_str, distance=distance)

async def add_suggestion(dict_key: str, suggestion: str, weight: float = 1.0) -> None:
    await client.ft().sugadd(dict_key, suggestion, weight)

async def get_suggestions(dict_key: str, prefix: str, fuzzy: bool = False) -> list[str]:
    return await client.ft().sugget(dict_key, prefix, fuzzy=fuzzy)
```
