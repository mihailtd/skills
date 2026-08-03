---
name: python-fastmcp-rag-retrieval
description: Guides teams to build Retrieval-Augmented Generation (RAG) pipelines in Python using FastMCP — exposing knowledge bases via MCP resources and tools, implementing multi-stage retrieval across servers, leveraging annotations (audience, priority, lastModified), dynamic query expansion, source attribution, and graceful degradation.
---

# Python FastMCP: RAG & Knowledge Retrieval

This skill helps AI implement Retrieval-Augmented Generation (RAG) pipelines using FastMCP. By wrapping vector databases, document repositories, SQL stores, and domain APIs as standardized MCP resource providers (`@mcp.resource()`) and search tools (`@mcp.tool()`), developers build flexible, self-discovering retrieval systems that eliminate static training cutoff limitations, mitigate hallucinations, and provide verifiable source citations.

---

## When to use this skill

Use this skill when you need to:

- build a RAG system that grounds LLM responses in real-time, domain-specific, or enterprise knowledge bases,
- expose document stores and search engines as MCP resources or tools,
- leverage MCP metadata annotations (`audience`, `priority`, `lastModified`) to filter and prioritize context,
- implement **Multi-Stage Retrieval** pipelines across multiple specialized MCP servers (e.g. Index Server -> Specialized Store Server -> Fact-Checker Server),
- perform **Dynamic Query Expansion** where MCP tools generate domain-specific search variations,
- track data provenance and generate source citations for retrieved snippets,
- handle source timeouts and partial retrieval gracefully without failing the entire generation turn.

---

## Outcome

Produce an MCP-based RAG pipeline that:

- dynamically discovers available document resources via `resources/list` and `tools/list`,
- queries domain stores through standardized JSON-RPC interfaces (`resources/read`, `tools/call`),
- filters and ranks context based on embedding similarity, recency (`lastModified`), and target persona (`audience`),
- grounds LLM generation in retrieved context snippets while returning verifiable source citations (`source`, `uri`, `last_modified`),
- degrades gracefully when individual data sources time out or fail.

---

## Instructions for the AI

1. **Model RAG Data Sources as MCP Primitives**
   - **Resources:** Use `@mcp.resource("doc://{id}")` for static or computed document reads where URI patterns identify specific records.
   - **Tools:** Use `@mcp.tool()` for parameterized semantic search queries (`search_documents(query: str, top_k: int = 5)`).
   - Use annotations to attach metadata context:
     - `audience`: `"user"` for end-user-facing summaries vs `"assistant"` for technical context.
     - `priority`: Numeric weight (0.0 to 1.0) indicating document importance.
     - `lastModified`: ISO timestamp indicating freshness.

2. **Implement Multi-Stage Retrieval Across MCP Servers**
   - **Stage 1 (General Index):** Query a high-level index server to identify relevant categories, jurisdictions, or domains.
   - **Stage 2 (Specialized Store):** Query specific domain MCP servers (e.g. legal statutes, medical guidelines, technical documentation) for detailed context snippets.
   - **Stage 3 (Verification / Fact-Checking):** Query a validation server to cross-check citations or verify factual consistency before generation.
   - Example multi-stage pipeline pattern:
     ```python
     import asyncio
     from mcp import ClientSession

     async def multi_stage_rag_pipeline(session: ClientSession, user_query: str):
         # Stage 1: Dynamic Query Expansion via MCP Tool
         expanded_queries_res = await session.call_tool(
             "expand_query", {"query": user_query}
         )
         queries = expanded_queries_res.content[0].text.split("\n")

         # Stage 2: Parallel Retrieval across queries
         search_tasks = [
             session.call_tool("semantic_search", {"query": q, "top_k": 3})
             for q in queries
         ]
         search_results = await asyncio.gather(*search_tasks, return_exceptions=True)

         # Consolidate & Rank retrieved snippets
         snippets = []
         for res in search_results:
             if not isinstance(res, Exception):
                 snippets.extend(res.content)

         # Stage 3: Grounded Generation Context
         context_str = "\n\n".join([s.text for s in snippets])
         return context_str
     ```

3. **Incorporate Metadata Annotations & Provenance Tracking**
   - Retain source URIs (`doc://HR-001`), document titles, and `lastModified` timestamps alongside every retrieved snippet.
   - Format prompt context so the LLM explicitly cites sources:
     ```python
     def build_augmented_prompt(user_query: str, retrieved_snippets: list[dict]) -> str:
         context_blocks = []
         for s in retrieved_snippets:
             context_blocks.append(
                 f"Source: [{s['title']}]({s['uri']}) (Last Updated: {s['last_modified']})\n"
                 f"Content: {s['content']}"
             )
         context_text = "\n\n---\n\n".join(context_blocks)
         return f"""Use ONLY the following retrieved context to answer the user query.
If the context does not contain the answer, state that information is unavailable.
Always cite your sources using the provided URIs.

RETRIEVED CONTEXT:
{context_text}

USER QUERY:
{user_query}"""
     ```

4. **Implement Graceful Degradation & Timeout Handling**
   - Set individual timeouts for retrieval calls (e.g. 2.0s per source).
   - If a source times out or returns an error (`return_exceptions=True`), log the warning and generate responses using the remaining successful sources.
   - Include source status warnings in response metadata when data sources are unreachable.

5. **Evaluate Retrieval & Generation Quality**
   - **Retrieval Quality:** Track precision, recall, and embedding cosine similarity scores (thresholds 0.3–0.8).
   - **Generation Consistency:** Validate that generated claims directly reference retrieved context blocks to prevent hallucination.

---

## Decision points and guidance

- **Resource vs Search Tool for RAG?** Use Resources (`@mcp.resource`) when documents are identified by known URIs or path schemes. Use Search Tools (`@mcp.tool`) when vector similarity or hybrid keyword searches require runtime query parameters.
- **How to handle large document corpora?** Do not dump full documents into the prompt context window. Return truncated, top-scoring context snippets with drill-down URIs (`doc://{id}`) for detailed inspection.
- **When to update data sources?** Server resource providers should emit `notifications/resources/listChanged` events when underlying documents are re-indexed or updated.

---

## Quality criteria

- **Factually Grounded:** Generation prompts restrict responses to retrieved context and mandate URI citations.
- **Freshness Aware:** Retrieval ordering evaluates `lastModified` annotations to prioritize up-to-date facts.
- **Resilient Execution:** Source timeouts or missing endpoints degrade gracefully without failing the user request.
- **Clean Metadata:** Provenance metadata (`source`, `uri`, `last_modified`) is preserved throughout the pipeline.

---

## Example prompts

- "Build an MCP RAG pipeline that exposes internal policy documents as FastMCP resources and provides source citations."
- "Implement a multi-stage retrieval pipeline using FastMCP that expands queries before searching vector stores."
- "Add fallback timeout logic to our FastMCP RAG system so missing data sources don't crash generation."

---

## Related guidance

- python-fastmcp-resource-providers
- python-fastmcp-server-basics
- python-fastmcp-client-integration
- python-langgraph
