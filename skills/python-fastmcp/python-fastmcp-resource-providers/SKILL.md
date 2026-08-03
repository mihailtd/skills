---
name: python-fastmcp-resource-providers
description: Guides teams to build MCP Resource Providers in Python using FastMCP — exposing read-only data, dynamic URIs (doc://{id}), computed context collections (docs://recent, docs://search), structured MIME types, and handling persistent datastore integration vs volatile in-memory caching.
---

# Python FastMCP: Resource Providers

This skill helps AI implement MCP Resource Providers — the read-only "knowledge base" components of Model Context Protocol servers. Unlike Tools (which execute side-effects and modify state), Resource Providers deliver structured context, documents, database entity views, dynamic parameters, and on-demand calculated analytics to AI clients and LLM agents.

---

## When to use this skill

Use this skill when you need to:

- expose read-only data, documentation, logs, or metrics to an AI system as structured MCP resources,
- design URI schemes for resources (`doc://{id}`, `users://all`, `analytics://task_metrics`),
- serve computed or filtered views dynamically based on URI path or parameters (`docs://recent`, `docs://search`),
- return resource content with accurate MIME types (`text/markdown`, `application/json`, `text/plain`),
- transition an in-memory resource provider mock into a durable, scalable implementation backed by PostgreSQL or MongoDB.

---

## Outcome

Produce a FastMCP resource provider that:

- advertises individual document URIs and derived resource collections via standard listing interfaces,
- resolves resource reads for exact URIs as well as pattern-based or computed queries,
- formats context clearly using `TextContent` payloads and appropriate MIME types,
- avoids volatile data loss by delegating persistent storage to proper database engines (PostgreSQL/MongoDB) rather than in-memory dictionaries.

---

## Instructions for the AI

1. **Understand Resource Providers vs Tool Providers**
   - **Resource Providers:** Read-only data sources that supply context, documents, user data, and calculated metrics. They do not alter system state.
   - **Tool Providers:** Action handlers ("hands") that modify state or trigger external side-effects (see `python-fastmcp-server-basics`).
   - Use resources whenever an AI agent needs background information, dynamic file views, or real-time status without performing an operational change.

2. **Define URI Schemes and Resource Catalog**
   - Create clean, predictable URI schemes:
     - Direct item access: `doc://project_charter`, `tasks://101`, `users://all`
     - Computed collections: `docs://recent`, `analytics://task_metrics`
     - Search & query URIs: `docs://search?query=api`
   - Use `@mcp.resource("uri://...")` in FastMCP or `list_resources` / `read_resource` handlers.
   - Specify human-friendly names, descriptions, and accurate MIME types (`text/markdown`, `application/json`).

3. **Implement Dynamic Retrieval and Derived Views**
   - Handle exact resource identifiers (e.g. stripping prefixes from `doc://{id}`).
   - Implement computed filtering logic for derived resources (e.g., filtering items modified within a cutoff period for `docs://recent`).
   - Example pattern (custom provider class or low-level/FastMCP handlers):
     ```python
     from datetime import datetime, timedelta
     import json
     from mcp.server.fastmcp import FastMCP, Resource
     from mcp.types import TextContent

     mcp = FastMCP("document-resource-provider")

     # Sample document store (for production, query PostgreSQL / MongoDB)
     DOCUMENTS = {
         "project_charter": {
             "title": "Project Charter",
             "content": "# Project Charter\nThis project aims to...",
             "format": "markdown",
             "last_modified": datetime.now() - timedelta(days=5),
         },
         "api_spec": {
             "title": "API Specification",
             "content": "# REST API Documentation\n\n## Endpoints\n...",
             "format": "markdown",
             "last_modified": datetime.now() - timedelta(days=2),
         },
     }

     @mcp.resource("doc://{doc_id}")
     def get_document(doc_id: str) -> str:
         """Retrieve a specific document by its ID."""
         if doc_id in DOCUMENTS:
             return DOCUMENTS[doc_id]["content"]
         raise ValueError(f"Document '{doc_id}' not found")

     @mcp.resource("docs://recent")
     def get_recent_documents() -> str:
         """Retrieve JSON collection of documents modified in the last 7 days."""
         cutoff = datetime.now() - timedelta(days=7)
         recent_docs = {
             doc_id: doc
             for doc_id, doc in DOCUMENTS.items()
             if doc["last_modified"] > cutoff
         }
         return json.dumps(recent_docs, indent=2, default=str)
     ```

4. **Ensure Durable Datastore Backing for Production**
   - **In-memory storage warning:** Storing resources in plain dictionaries is fast for dev/testing, but volatile and non-persistent. Process restarts lose data, and scaling across multiple worker processes will cause state drift.
   - **Production pattern:** Query relational databases (SQLAlchemy / PostgreSQL) or document stores (Beanie / MongoDB) inside resource read handlers.

5. **Implement Context-Driven Persona Views**
   - Format raw data for specific LLM intent and role contexts.
   - Example: A single customer entity in MongoDB or PostgreSQL can be exposed via different resource URIs:
     - `customer://{id}/support`: Formatted for support agents (recent tickets, escalation history, contact preferences).
     - `customer://{id}/sales`: Formatted for sales agents (purchase history, upsell opportunities, relationship timeline).

6. **Follow Progressive Disclosure of Complexity**
   - Provide summary endpoints first for low token footprint (e.g. `docs://titles` returning titles and summaries), while allowing models to drill down into specific detail URIs (`doc://{id}`) or raw logs when needed.

7. **Attach Data Context in `_meta`**
   - Use the reserved `_meta` dictionary on resource payloads to supply non-content metadata:
     - Data freshness / last modified timestamps (`last_modified`).
     - Source confidence scores or data quality indicators.
     - Usage hints for downstream tools.

8. **Format Payloads and Handle Missing Resources**
   - Return structured content cleanly formatted as markdown or JSON strings.
   - Raise explicit `ValueError` or return clear error signals when a requested URI does not exist or parameters are invalid.

---

## Decision points and guidance

- **Should this data be exposed as a Resource or a Tool?** If the operation is purely informational and has zero side-effects, expose it as a Resource. If it changes state, sends messages, or executes code, expose it as a Tool.
- **Static vs Computed Resources?** Static resources mirror fixed documents or records. Computed resources calculate metrics on demand (e.g. `analytics://task_metrics`) rather than storing pre-computed values.
- **Is the data persistent?** Do not rely on process memory for production resources — connect the resource provider to PostgreSQL or MongoDB.

---

## Quality criteria

- **Predictable URIs:** URI schemes follow clear namespace patterns (`doc://`, `users://`, `analytics://`).
- **Correct MIME types:** MIME types match output content (`text/markdown` for markdown text, `application/json` for dynamic queries/collections).
- **Graceful Error Handling:** Requesting unknown URIs raises informative exceptions rather than returning corrupted or blank data.
- **Durable Architecture:** Production resource providers read from persistent databases or external services.

---

## Example prompts

- "Create an MCP resource provider that exposes system log files and recent audit entries as resources."
- "Implement dynamic search resources in FastMCP over our project documentation store."
- "Refactor this in-memory MCP resource dictionary to read directly from a PostgreSQL database using async SQLAlchemy."

---

## Related guidance

- python-fastmcp-server-basics
- python-fastmcp-client-integration
- python-postgresql
- python-mongodb
