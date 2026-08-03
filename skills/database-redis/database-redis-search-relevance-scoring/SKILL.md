---
name: database-redis-search-relevance-scoring

description: Guides the agent on how RediSearch ranks full-text results — the default TF-IDF scorer, boosting relevance with a document field via SCORE_FIELD, switching scoring algorithms with SCORER, debugging a ranking with EXPLAINSCORE, and boosting individual fields with WEIGHT — for building content-based recommendation/ranking features, not just filtered lookups.
---

# RediSearch Relevance Scoring

You are an expert in RediSearch's relevance scoring. When a user needs search results ranked by *relevance* rather than an arbitrary or purely chronological order — the content-based half of a recommendation feature — guide them to RediSearch's scoring mechanism instead of computing a ranking in application code after the fact.

- **A full-text (`TEXT` field) search is ranked, not just filtered — every match gets a numeric score, and results come back ordered by it by default.** This ranking is what separates a *search* from a *query* (see `database-redis-search-query-syntax`'s query-vs-search framing): a query returns a deterministic set with no notion of "how well" something matched, whereas a search result's order carries meaning.
- **The default scorer is TF-IDF**, scoring a match by how often the search term(s) appear in the matched document relative to how common they are across the whole index — a document mentioning the search term twice generally outranks one mentioning it once, all else equal. This happens automatically; no configuration is needed to get relevance-ordered full-text results.
- **Understand *why* a particular ranking happened with `EXPLAINSCORE`**, rather than guessing — it breaks down exactly how the final score was computed for each result:

  ```
  FT.SEARCH books_idx death NOCONTENT WITHSCORES EXPLAINSCORE
  ```

  This is the debugging tool of first resort whenever a ranking looks wrong or surprising — it shows term frequency, inverse document frequency, and any document-score/weight factors that went into the number, instead of leaving you to reverse-engineer the ranking from the results alone.
- **Fold in a "human" relevance signal (a rating, a popularity score) with `SCORE_FIELD` at index-creation time**, so ranking reflects more than just term frequency — this is what turns a plain full-text search into something closer to a recommendation:

  ```
  FT.CREATE books_idx SCORE_FIELD $.rating ON JSON PREFIX 1 book:
      SCHEMA $.synopsis AS synopsis TEXT
  ```

  The named field (expressed as a value between 0 and 1) multiplies into the final score, so a highly-rated document can outrank a document that merely mentions the search term more often. This requires rebuilding the index with the new setting — it's not a per-query option — so decide on this at schema-design time if relevance-plus-rating ranking is a known requirement, or use `database-redis-search-indexing`'s `FT.ALIAS` technique to swap in the change without downtime.
- **Switch the scoring algorithm entirely with `SCORER`** when TF-IDF's term-frequency approach isn't the right relevance model for the use case — options include `TFIDF.DOCNORM` (TF-IDF normalized by document length), `BM25` (a widely-used, generally stronger general-purpose alternative to plain TF-IDF), `DISMAX`, `DOCSCORE` (rank purely by the `SCORE_FIELD` value, ignoring term frequency entirely), and `HAMMING`. Reach for `DOCSCORE` specifically when the search should rank *entirely* by an external signal (rating, popularity) and term-frequency differences between matches shouldn't matter at all:

  ```
  FT.SEARCH books_idx death NOCONTENT WITHSCORES SCORER DOCSCORE EXPLAINSCORE
  ```

  A custom scoring function can also be registered for cases none of the built-ins fit — treat that as an escape hatch for genuinely unusual ranking requirements, not the default path.
- **`SORTBY` and scoring are mutually exclusive per query** — sorting by a specific field bypasses the relevance scorer entirely for that query. Don't combine `SORTBY` with an expectation that relevance still factors in; pick one: relevance-ranked (no `SORTBY`, let the scorer order results) or field-ordered (`SORTBY`, ignore relevance).
- **Boost specific fields' contribution to the score with `WEIGHT`** when a search spans multiple `TEXT` fields (e.g. title and body) and a match in one field should count for more than a match in another — e.g. a title match is usually a stronger relevance signal than a body match, and `WEIGHT` on the title field lets the scorer reflect that instead of treating every field's term frequency equally.
- **This is the content/text-relevance half of a recommendation engine — vector similarity search is the other half**, for ranking by resemblance to unstructured data (images, audio, "items like this one") rather than term frequency in text. See `database-redis-vector-similarity-search` for that complementary technique; a real recommendation feature often combines both (filter/rank by scoring here, then re-rank or blend with a similarity search pass).

## Code Examples

```python
from redis import asyncio as aioredis
from redis.commands.search.query import Query

client = aioredis.from_url("redis://localhost")

async def search_ranked_by_relevance(index: str, term: str) -> list[str]:
    """Default TF-IDF ranking — no extra configuration needed."""
    result = await client.ft(index).search(Query(term).no_content())
    return [doc.id for doc in result.docs]

async def explain_ranking(index: str, term: str) -> None:
    """Debug why results are ordered the way they are."""
    result = await client.execute_command(
        "FT.SEARCH", index, term, "NOCONTENT", "WITHSCORES", "EXPLAINSCORE",
    )
    return result

async def search_by_document_score_only(index: str, term: str) -> list[str]:
    """Rank purely by the field configured via SCORE_FIELD, ignoring term frequency."""
    result = await client.execute_command(
        "FT.SEARCH", index, term, "NOCONTENT", "SCORER", "DOCSCORE",
    )
    return result
```
