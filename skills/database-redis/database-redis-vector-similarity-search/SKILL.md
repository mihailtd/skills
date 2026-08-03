---
name: database-redis-vector-similarity-search

description: Guides the agent on Redis Stack's vector similarity search (VSS) — storing embeddings in Hash (binary blob) or JSON (array), indexing with the VECTOR field type (FLAT vs HNSW, TYPE/DIM/DISTANCE_METRIC), KNN similarity queries, hybrid pre-filtered queries, and VSS range queries — for recommendation, fraud-pattern, and similarity-matching use cases.
---

# Redis Stack Vector Similarity Search (VSS)

You are an expert in Redis Stack's vector similarity search capability. When a user needs to recommend, match, or classify unstructured data (images, audio, text, time series, transaction patterns) by similarity rather than exact/range/text matching, guide them to the VSS pipeline instead of trying to force the problem into `TEXT`/`TAG`/`NUMERIC` search. This is the similarity half of a recommendation feature; for the content/text-relevance half (ranking full-text matches by TF-IDF, a rating field, or a custom scorer), see `database-redis-search-relevance-scoring` — a real recommendation feature often combines both.

- **Recognize when a requirement is actually a similarity-search problem, not a keyword-search one.** "Find items like this one," "recommend based on visual/audio resemblance," "detect this transaction as similar to known fraud patterns," "match this voice recording to a known speaker" — none of these can be expressed as exact-match, range, or full-text queries (see `database-redis-search-indexing`), because there's no discrete field to filter on. The underlying data needs to be represented as a vector so that "similar" becomes a well-defined, computable distance between two vectors.
- **The pipeline has three stages, and Redis Stack is only responsible for the last two:**
  1. **Generate the embedding.** An external model converts the unstructured input — text, an image, audio — into a fixed-length vector (a pre-trained model from Hugging Face/Sentence-Transformers, PyTorch, OpenAI, or similar; Redis Stack does not generate embeddings itself).
  2. **Store and index the vector alongside its metadata** in Redis Stack, building a searchable database of known items.
  3. **Embed a new query input the same way, then ask Redis Stack for the nearest matches** by a chosen distance metric.
- **Typical use cases**: product recommendations by visual similarity, correlating time series patterns, flagging transactions similar to known fraud (see `database-redis-fraud-detection`), content/music recommendation, connecting profiles with similar interests, and the LLM-application patterns in `database-redis-llm-integration` (RAG context retrieval, conversation memory, semantic caching). If a requirement matches one of these shapes, VSS is very likely the right tool even if the user hasn't named it that way.
- **The embedding model must be consistent between indexing and querying** — vectors are only comparable if produced by the same model (or a model producing vectors in the same space). Mixing embeddings from different models/versions makes distances meaningless. Pin and track which model/version generated a given index's vectors — this is a real operational dependency, not incidental metadata.
- **Every model has a fixed output size and an input size limit** — e.g. a given Sentence-Transformers model might emit 384-dimensional vectors and silently truncate input text beyond ~512 words (the truncated portion simply isn't considered, with no error). Check both the vector dimensionality (it must match the index's `DIM`) and the input length limit for whatever model is in use; for content that can exceed the limit, split it into chunks and index one embedding per chunk rather than assuming truncation is harmless.

## Storing Embeddings: Hash vs. JSON

The two document types (see `database-redis-data-modeling-fundamentals`/`database-redis-json-documents`) store a vector in genuinely different formats — this isn't cosmetic, get it wrong and the index either rejects the data or silently misinterprets it.

- **Hash stores the vector as a packed binary blob**, not as text — convert with NumPy before writing:

  ```python
  blob = embedding.astype(np.float32).tobytes()
  r.hset("doc:1", mapping={"embedding": blob, "genre": "technical", "content": text})
  ```

  The byte layout must match what the index expects (`FLOAT32` vs `FLOAT64`, and the right count of values for `DIM`) — get either wrong and the index either rejects the write or reads garbage.
- **JSON stores the vector as a plain array of numbers**, not a blob — convert to a Python list instead:

  ```python
  doc = {"embedding": embedding.tolist(), "genre": "technical", "content": text}
  r.json().set("doc:1", "$", doc)
  ```

- **JSON additionally supports multiple embeddings per document via multi-value indexing** (see `database-redis-json-documents`) — when a JSONPath resolves to multiple vectors (e.g. one embedding per paragraph, or per product photo), all of them become independently searchable under one indexed field, letting one logical item match on *any* of its several embeddings.

## Indexing: the VECTOR Field Type

```
FT.CREATE doc_idx ON HASH PREFIX 1 doc:
    SCHEMA content AS content TEXT
           genre AS genre TAG
           embedding VECTOR HNSW 6 TYPE FLOAT32 DIM 384 DISTANCE_METRIC COSINE
```

(Swap `ON HASH`/flat field names for `ON JSON`/JSONPath — `$.embedding VECTOR ...` — exactly as with any other field type; see `database-redis-search-indexing`.)

- **Choose the algorithm based on the accuracy/scale trade-off actually needed, not by default:**
  - **`FLAT`** — brute-force: computes the distance to *every* indexed vector for each query. Exact (never approximate), but its cost grows linearly with the number of vectors — expensive once the collection is large. Scales well across shards, since the brute-force computation parallelizes cleanly.
  - **`HNSW`** (Hierarchical Navigable Small World) — a graph-based approximate-nearest-neighbor algorithm. Much faster at scale, but trades away some accuracy — it can occasionally miss the true nearest neighbor in exchange for speed.
  - Default to `HNSW` once the vector count is large enough that brute-force search is measurably slow; use `FLAT` when exactness matters more than raw speed, or the dataset is small enough that the difference doesn't matter.
- **Three parameters are mandatory for either algorithm, and all three must match the embeddings actually being stored:**
  - **`TYPE`** — `FLOAT32` or `FLOAT64`, matching the precision the embedding model actually outputs (most models: `FLOAT32`).
  - **`DIM`** — the exact vector length the model produces (384 in the example above). A mismatch here isn't a subtle bug — writes/queries with the wrong dimensionality fail outright.
  - **`DISTANCE_METRIC`** — see below; this determines what "similar" actually means for the data.

### Choosing a Distance Metric

- **`L2` (Euclidean distance)** — straight-line distance in vector space. Use when the vector's *magnitude* is meaningful, not just its direction — e.g. vectors of physical/continuous measurements or coordinates, where "how far apart" has a literal physical meaning.
- **`COSINE`** — the angle between vectors, ignoring magnitude entirely. Use for most text/document/image embedding comparisons, and generally for high-dimensional spaces — this is the default choice for the vast majority of "find similar content" use cases (it's what the worked examples above use).
- **`IP` (inner product)** — accounts for both angle *and* magnitude. Use specifically when magnitude itself carries meaning alongside direction — e.g. comparing rating vectors where both which items a user likes (direction) and how strongly (magnitude) both matter for the recommendation. Note `IP` becomes equivalent to `COSINE` once vectors are normalized to unit length — if embeddings are already normalized, the choice between the two stops mattering.
- Getting this wrong doesn't error — it silently returns technically-valid but semantically-meaningless "closest" results, which is a much harder bug to catch than a rejected write. Choose deliberately based on what the underlying data actually represents, not by copying a metric from an unrelated example.

## Similarity (KNN) Search

```python
q = Query("*=>[KNN 2 @embedding $vec AS score]").return_field("score").dialect(2)
res = r.ft("doc_idx").search(q, query_params={"vec": query_embedding.astype(np.float32).tobytes()})
```

- **`*=>[KNN k @field $param AS alias]` means "no pre-filter, just the k nearest neighbors"** — `*` is the (absent) filter, `k` is how many results to return, `@field` is the indexed `VECTOR` field, `$param` is a query parameter bound at execution time (never inline the raw vector bytes into the query string itself).
- **`DIALECT 2` is required for any VSS query** — this isn't optional the way it's a tunable elsewhere (see `database-redis-search-query-syntax`); a VSS query without it won't work as intended. Set it per-query (`.dialect(2)`) or as the server default (`FT.CONFIG SET DEFAULT_DIALECT 2`).
- **The returned `score` is a distance, not a similarity — lower is better/closer.** Sort ascending by it (or by the implicit `__<field>_score` value) to get closest-first ordering; don't assume "score" means "higher is more relevant" the way it might for the TF-IDF scoring in `database-redis-search-relevance-scoring` — these are different, unrelated numbers.

## Hybrid Queries (Pre-Filtered VSS)

```python
q = Query("@genre:{technical}=>[KNN 2 @embedding $vec AS score]").return_field("score").sort_by("score").dialect(2)
```

- **A conventional filter (`TAG`/`TEXT`/`NUMERIC`/`GEO`) placed before `=>[KNN ...]` narrows the candidate set *before* the vector distance is computed**, rather than computing KNN over the whole index and filtering after. This is both a correctness tool (restrict results to a meaningful category, e.g. "similar documents, but only technical ones") and a performance one (fewer candidate vectors to compare against).
- Reach for a hybrid query whenever there's a real business constraint the recommendation must respect ("similar products, but only in stock," "similar support tickets, but only from this customer") — encoding that as a pre-filter is far simpler than post-filtering KNN results in application code and back-filling with more results when too many get filtered out.

## VSS Range Queries

```python
q = Query("@embedding:[VECTOR_RANGE $radius $vec]=>{$YIELD_DISTANCE_AS: score}").sort_by("score").return_field("score").dialect(2)
query_params = {"radius": 0.8, "vec": query_embedding.astype(np.float32).tobytes()}
```

- **Use a range query when the requirement is "everything within a similarity threshold," not "the top K closest items."** `KNN` always returns exactly `k` results, even if the k-th closest match is nowhere near actually similar; `VECTOR_RANGE` returns however many results fall within `$radius` of the query vector — zero, one, or many — which is the right semantics when "how similar is similar enough" matters more than a fixed result count.
- **Choosing the right radius depends entirely on the distance metric and the embedding model** — there's no universal "good" threshold; determine it empirically for the specific model/metric combination in use (e.g. by checking distances for known-similar and known-dissimilar pairs) rather than guessing a value.

## Code Examples

```python
from redis import asyncio as aioredis
from redis.commands.search.field import TextField, TagField, VectorField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType
from redis.commands.search.query import Query
import numpy as np

client = aioredis.from_url("redis://localhost")

async def create_vector_index(index_name: str, dim: int) -> None:
    schema = (
        TextField("content"),
        TagField("genre"),
        VectorField("embedding", "HNSW", {"TYPE": "FLOAT32", "DIM": dim, "DISTANCE_METRIC": "COSINE"}),
    )
    await client.ft(index_name).create_index(schema, definition=IndexDefinition(prefix=["doc:"], index_type=IndexType.HASH))

async def store_embedding(doc_id: str, embedding: np.ndarray, content: str, genre: str) -> None:
    await client.hset(f"doc:{doc_id}", mapping={
        "embedding": embedding.astype(np.float32).tobytes(), "content": content, "genre": genre,
    })

async def nearest_neighbors(index_name: str, query_embedding: np.ndarray, k: int = 2) -> list[tuple[str, float]]:
    query = Query(f"*=>[KNN {k} @embedding $vec AS score]").return_field("score").sort_by("score").dialect(2)
    result = await client.ft(index_name).search(query, query_params={"vec": query_embedding.astype(np.float32).tobytes()})
    return [(doc.id, float(doc.score)) for doc in result.docs]

async def nearest_neighbors_in_genre(index_name: str, query_embedding: np.ndarray, genre: str, k: int = 2) -> list[str]:
    """Hybrid query: pre-filter by genre before computing vector distance."""
    query = Query(f"@genre:{{{genre}}}=>[KNN {k} @embedding $vec AS score]").sort_by("score").dialect(2)
    result = await client.ft(index_name).search(query, query_params={"vec": query_embedding.astype(np.float32).tobytes()})
    return [doc.id for doc in result.docs]

async def similar_within_radius(index_name: str, query_embedding: np.ndarray, radius: float) -> list[str]:
    """Range query: however many results fall within the threshold, not a fixed count."""
    query = (
        Query("@embedding:[VECTOR_RANGE $radius $vec]=>{$YIELD_DISTANCE_AS: score}")
        .sort_by("score").dialect(2)
    )
    params = {"radius": radius, "vec": query_embedding.astype(np.float32).tobytes()}
    result = await client.ft(index_name).search(query, query_params=params)
    return [doc.id for doc in result.docs]
```
