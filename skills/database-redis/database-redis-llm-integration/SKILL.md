---
name: database-redis-llm-integration

description: Guides the agent on Redis's three main roles in LLM-based applications — vector context retrieval for retrieval-augmented generation (RAG), persisting conversation memory as embeddings, and semantic caching of LLM completions by prompt similarity — all built on VSS, plus pointers to the frameworks that already integrate Redis for these patterns.
---

# Redis for LLM / Generative AI Integration

You are an expert in using Redis as the data layer for LLM-based applications. When a user is building a feature involving an LLM — a chatbot, a Q&A system, a RAG pipeline, anything that calls out to a model like GPT — guide them to the specific role(s) Redis plays, all of which build on `database-redis-vector-similarity-search`, rather than treating Redis as just an incidental cache.

- **The core constraint Redis addresses: an LLM's knowledge is frozen at training time, and training is too expensive to redo often.** A model trained months or years ago has no knowledge of anything newer, and can't be casually retrained to catch up. Applications that need current, private, or domain-specific knowledge in their responses need a way to feed that knowledge to the model *at request time*, without retraining it — this is what all three patterns below actually solve, from different angles.

## Retrieval-Augmented Generation (RAG)

- **RAG's job is to find the small number of documents/chunks actually relevant to a given question, and hand them to the LLM as context** — instead of relying on the model's frozen training data, or (impossibly) feeding it your entire knowledge base on every request.
- **This is a direct application of VSS**: embed your knowledge base (documents, support articles, product data — chunked to a reasonable size per the embedding model's input limit, see `database-redis-vector-similarity-search`) once, ahead of time; embed the user's question the same way at query time; retrieve the nearest-neighbor chunks; include them in the prompt sent to the LLM.
- **The quality of the retrieved context directly bounds the quality of the answer** — if VSS returns irrelevant chunks (wrong distance metric, embeddings from an inconsistent model, chunks that are too large/small to be meaningfully specific), the LLM either answers from its own possibly-outdated training data anyway, or fabricates ("hallucinates") an answer to fill the gap. Treat retrieval quality as the primary lever for answer quality, not just a preprocessing detail.
- **Use a hybrid query (see `database-redis-vector-similarity-search`) when retrieval needs to respect a real constraint** — e.g. "only search documents this user has access to," "only search the current product line's docs" — as a pre-filter alongside the KNN clause, not as a separate access-control pass applied after retrieval.

## LLM Conversation Memory

- **Persisting conversation history as embeddings lets a conversational agent recall relevant *past* exchanges, not just the current session's immediate context window.** An LLM's context window is finite and doesn't persist between sessions on its own — without an external memory store, a returning user's earlier conversations are simply gone.
- **The retrieval pattern is the same VSS pipeline as RAG, applied to conversation turns instead of documents**: embed each meaningful exchange as it happens, store it (with metadata like user ID, timestamp, topic), and on a new turn, retrieve the most relevant past exchanges by similarity to the current message — not necessarily the most *recent* ones, which is the key difference from a simple chat log.
- **This enables cross-session continuity and personalization**: a user picking up a topic from a prior session, or the agent adapting behavior based on established preferences, without the application needing to hand-manage what to remember and what to forget.

## Semantic Caching

- **LLM completions are expensive (latency and cost) to compute, and caching by exact prompt string misses most of the actual savings opportunity** — users rarely phrase the same question identically twice, so a naive string-keyed cache (see `database-redis-caching-strategy`) has a very low hit rate for this workload even when the same underlying question is being asked repeatedly.
- **Semantic caching keys the cache by prompt *similarity*, not exact match**: embed the incoming prompt, run a VSS query against previously-cached prompts, and treat a close-enough match (within a chosen distance threshold — see `database-redis-vector-similarity-search`'s range-query guidance for picking one) as a cache hit, returning the previously-computed completion instead of calling the LLM again.
- **This is a genuine cost/latency optimization specific to LLM workloads** — apply the general caching discipline from `database-redis-caching-strategy` (TTLs, since cached answers can go stale as underlying knowledge changes; stampede protection for popular prompts) on top of the similarity-based lookup, don't treat semantic caching as a replacement for those concerns.
- **Set the similarity threshold deliberately, not loosely** — too tight and the cache rarely hits (defeating the purpose); too loose and meaningfully different questions get served the same cached answer, which is a correctness problem, not just an efficiency one. Validate the threshold against real query pairs the application actually expects to see.

## Ecosystem Context

- **Don't assume every RAG/agent project needs to be built from these primitives by hand** — established frameworks (LangChain, LlamaIndex) and cloud services (Azure OpenAI/Microsoft Semantic Kernel, Amazon Bedrock) already integrate Redis as a vector store/memory backend for exactly these patterns. Check whether the project's existing framework already has a Redis integration before hand-rolling the embedding/storage/retrieval plumbing — reach for the raw VSS primitives in this package when building a custom pipeline outside those frameworks, or when framework-provided integration doesn't fit a specific requirement.

## Code Example

```python
from redis import asyncio as aioredis
from redis.commands.search.query import Query
import numpy as np

client = aioredis.from_url("redis://localhost")

async def semantic_cache_lookup(index_name: str, prompt_embedding: np.ndarray, max_distance: float = 0.1) -> str | None:
    """Return a cached completion if a sufficiently similar prompt was already answered."""
    query = (
        Query("@embedding:[VECTOR_RANGE $radius $vec]=>{$YIELD_DISTANCE_AS: score}")
        .sort_by("score").return_field("completion").dialect(2)
    )
    params = {"radius": max_distance, "vec": prompt_embedding.astype(np.float32).tobytes()}
    result = await client.ft(index_name).search(query, query_params=params)
    return result.docs[0].completion if result.docs else None

async def cache_completion(index_name: str, prompt_id: str, prompt_embedding: np.ndarray, completion: str, ttl_seconds: int = 3600) -> None:
    key = f"promptcache:{prompt_id}"
    await client.hset(key, mapping={
        "embedding": prompt_embedding.astype(np.float32).tobytes(), "completion": completion,
    })
    await client.expire(key, ttl_seconds)
```
