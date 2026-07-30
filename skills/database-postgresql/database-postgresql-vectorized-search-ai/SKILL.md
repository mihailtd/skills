---
name: database-postgresql-vectorized-search-ai

description: Guides the implementation of Vectorized Search using pgvector in PostgreSQL for emerging AI, machine-learning, and RAG (Retrieval-Augmented Generation) integrations.
---

# PostgreSQL Vectorized Search for AI and RAG

This skill helps developers use PostgreSQL with `pgvector` to store embeddings,
perform semantic similarity search, and power RAG pipelines without a separate
vector database.

---

## 1. Unifying Workloads with `pgvector`

- **The unified platform:** PostgreSQL can handle transactional, analytical, and
  vector workloads together, reducing data movement and eliminating silos.
- **Vector similarity search:** AI models encode text semantics into vectors, and
  similarity search finds the closest vectors by measuring high-dimensional
  distance.
- **Enable `pgvector`:** Install and enable the extension with
  `CREATE EXTENSION pgvector;`.

---

## 2. Powering RAG (Retrieval-Augmented Generation)

- **RAG architecture:** Use PostgreSQL vector search as the retrieval layer for
  RAG. A semantic query retrieves relevant private data and appends it to the
  prompt instead of fine-tuning the model.
- **Chunking:** Split documents into smaller segments (e.g., a few hundred
  tokens) before generating embeddings to preserve semantic matching and avoid
  noise.

Example:

```sql
CREATE TABLE documents (
    id serial PRIMARY KEY,
    content text NOT NULL,
    embedding vector(1536)
);
```

---

## 3. Optimizing Vector Indexes (HNSW)

- **HNSW indexes:** Use HNSW for fast approximate nearest neighbor search.
- **Tuning:** Calibrate graph density (connections per node) and search depth to
  balance retrieval quality, latency, and CPU usage.
- **Resource management:** Indexing millions of rows can be CPU- and memory-
  intensive. Tune `maintenance_work_mem` and use parallel index builds when
  available.

Example:

```sql
CREATE INDEX idx_documents_embedding ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

---

## 4. Managing High-Dimensional Embeddings

- **Dimensionality cost:** High-dimensional embeddings carry more semantic
  information but increase storage and query time.
- **`half_vec` solution:** When performance or storage is an issue, use
  reduced dimensionality or the `half_vec` data type to save space and improve
  speed with minimal quality loss.

Example:

```sql
CREATE TABLE documents (
    id serial PRIMARY KEY,
    content text NOT NULL,
    embedding half_vec(1536)
);
```

---

## 5. Hybrid Search and Avoiding Over-Filtering

- **Over-filtering trap:** Applying strict metadata filters after vector search
  can shrink the result set too much and discard good semantic matches.
- **Iterative scanning:** Use pgvector’s iterative scanning to keep probing until
  enough valid candidates are found.
- **Hybrid scoring:** Blend vector similarity with metadata weights like recency or
  author to rank results rather than strictly excluding them.

Example hybrid strategy:

```sql
SELECT id, content,
       1 - (embedding <=> query_embedding) * 0.8 +
       (CASE WHEN created_at > now() - interval '30 days' THEN 0.2 ELSE 0 END)
       AS score
FROM documents
ORDER BY score DESC
LIMIT 10;
```

---

## Example prompt

- "Show me how to store embeddings in PostgreSQL with `pgvector` and use it for
  semantic search in a RAG pipeline."
- "Explain how to tune HNSW indexes and avoid over-filtering when combining vector
  similarity with metadata constraints."
- "Describe when to use `half_vec` for large embedding tables and how it helps
  with performance."
