---
name: python-fastmcp-server-basics
description: Guides teams to build MCP (Model Context Protocol) servers in Python using FastMCP — converting a plain function into an MCP tool with the @mcp.tool() decorator, writing type hints and docstrings that become the tool's schema and description, choosing a transport (stdio for local dev, HTTP/WebSocket for remote/shared servers), and understanding the Host/Client/Server architecture the server fits into.
---

# Python FastMCP: Server Basics

This skill helps AI build MCP servers with the FastMCP library — the
standardized way to expose tools an LLM agent can discover and call, without
hand-rolling tool schemas or a custom execution dispatcher. It covers the
Host/Client/Server architecture MCP defines, the decorator-based approach
FastMCP uses to turn ordinary Python functions into MCP tools, how type
hints and docstrings become the tool's contract, and how to pick a transport.

---

## When to use this skill

Use this skill when you need to:

- expose a Python function as a tool that any MCP-compatible LLM agent or
  client can discover and call,
- decide whether to build a custom MCP server versus using an existing
  community server for a given capability,
- write function signatures and docstrings that produce a clear, reliable
  MCP tool schema,
- choose a transport mechanism (stdio, HTTP, WebSocket) for a new server,
- understand where a server fits in the broader Host/Client/Server
  architecture before writing code.

---

## Outcome

Produce a FastMCP server that:

- exposes one or more Python functions as MCP tools via the `@mcp.tool()`
  decorator, with minimal changes to the original function's core logic,
- derives a correct, LLM-usable tool schema automatically from the
  function's type-hinted parameters, and a clear tool description from its
  docstring,
- uses a transport appropriate to how the server will actually be
  deployed and consumed (stdio for local/dev, HTTP or WebSocket for
  remote/shared production use),
- fits cleanly into the Host/Client/Server architecture MCP defines, so it
  can be consumed by any compliant client without custom integration code,
- fails predictably and informatively when a tool call errors, rather than
  crashing the server or returning an opaque failure.

---

## Instructions for the AI

1. **Explain the architecture the server fits into**
   - **Host:** the application with the LLM that reasons and decides which
     tools to use (e.g., an agent application, Claude Desktop, an
     AI-enabled IDE) — the "brain."
   - **Client:** the bridge between Host and Server — one Client maintains
     a connection to one Server, requesting tool definitions and
     forwarding execution requests.
   - **Server:** what you're building here — it hosts and executes the
     actual tools, responding to two standardized request types: "what
     tools do you have?" (tool discovery) and "execute this tool with
     these arguments" (tool execution). A server can wrap external
     services (databases, APIs) behind its tools.
   - Note that the official MCP specification defines three core server features:
     - **Tools:** executable actions ("the hands") that manipulate state, invoke APIs, or perform external operations.
     - **Resources:** read-only data and context sources ("the knowledge base") that provide documents, dynamic URI templates, and computed analytics (see `python-fastmcp-resource-providers`).
     - **Prompts:** reusable, parameterized prompt templates ("the conversation starters") that guide LLM interactions (see `python-fastmcp-prompt-providers`).
   - Emphasize good tool design principles:
     - **Atomic operations:** Each tool does one specific job cleanly.
     - **Clear interfaces:** Schema and signatures are explicit and unambiguous.
     - **Error handling:** Exceptions are caught and returned as readable failure payloads rather than crashing the server.
     - **Idempotency:** Tools modifying state are designed to be safely callable multiple times when possible.
     - **Input validation:** Validate parameters before attempting side effects.
   - Emphasize the value proposition driving this architecture: tool
     implementation and tool usage are fully decoupled — a server built
     once works with any MCP-compatible client, the same way any HTTP
     client can call any REST API without custom per-service integration
     code.

2. **Convert a function into an MCP tool with the decorator**
   - Start from a plain, working Python function with clear type-hinted
     parameters and a return type.
   - Instantiate a server with `FastMCP("server-name")`, using a name that
     meaningfully identifies the server to clients.
   - Apply `@mcp.tool()` directly above the function definition — this is
     normally the only change needed; the function's core logic stays
     untouched.
   - Example pattern:
     ```python
     import os
     from dotenv import load_dotenv
     from mcp.server.fastmcp import FastMCP

     load_dotenv()
     mcp = FastMCP("custom-tavily-search")

     @mcp.tool()
     def search_web(query: str, max_results: int = 5) -> str:
         """
         Search the web using Tavily API.

         Args:
             query: Search query string
             max_results: Maximum number of results to return (default: 5)

         Returns:
             Search results as formatted string
         """
         try:
             response = tavily_client.search(query, max_results=max_results)
             results = response.get("results", [])
             return "\n\n".join(
                 f"Title: {r['title']}\nURL: {r['url']}\nContent: {r['content']}"
                 for r in results
             )
         except Exception as e:
             return f"Error searching web: {str(e)}"

     if __name__ == "__main__":
         mcp.run(transport="stdio")
     ```
   - Recommend catching and returning errors as a formatted string result
     (rather than letting exceptions propagate) so a failed tool call
     produces a usable, LLM-readable message instead of crashing the
     server or surfacing an opaque protocol-level error.

3. **Write type hints and docstrings as the tool's real contract**
   - Treat function parameter type hints as the source of truth for the
     generated schema — FastMCP extracts parameter names, types, and
     defaults directly from the signature, so incorrect or overly loose
     typing (e.g., untyped `**kwargs`, bare `Any`) produces a weak or
     misleading schema for the calling LLM.
   - Treat the docstring as the tool's description — this is what an LLM
     reads to decide *when* and *how* to use the tool, so it should
     clearly state what the tool does, describe each parameter's meaning
     (not just its type), and describe the return value's shape. A vague
     or missing docstring degrades tool selection quality even if the
     underlying function works correctly.
   - Recommend giving default values for optional parameters directly in
     the signature (as in `max_results: int = 5` above) so the generated
     schema correctly marks them as optional with a sensible default.

4. **Enforce Streamable HTTP as the Production Transport Standard**
   - **Streamable HTTP (Mandatory for Production):** Production MCP servers MUST use Streamable HTTP (`transport="sse"` or HTTP/SSE streaming endpoints). Streamable HTTP leverages standard web infrastructure (load balancers, reverse proxies, TLS termination, API gateways) and enables streaming responses and partial updates via Server-Sent Events (SSE).
   - **stdio (Local Debugging Only):** stdio communicates over process pipes (`stdin`/`stdout`). Use stdio strictly for quick local CLI experimentation or Inspector debugging (`npx @modelcontextprotocol/inspector`). Do NOT use stdio for deployed agent infrastructure.
   - Example production server launch:
     ```python
     if __name__ == "__main__":
         # Production deployment enforcing Streamable HTTP
         mcp.run(transport="sse", host="0.0.0.0", port=8000)
     ```

5. **Protocol Primitives, Long-Running Operations & Metadata**
   - **Standard Primitives:** All interactions use JSON-RPC 2.0 primitives (`tools/list`, `tools/call`, `resources/list`, `resources/read`, `prompts/list`, `prompts/get`).
   - **Output Schema Validation:** Tool and resource responses must validate against declared schemas. Schemas default to **JSON Schema 2020-12** dialect (`$schema`).
   - **Standard JSON-RPC Error Codes:** Return standard error codes to help clients distinguish protocol errors from tool execution failures:
     - `-32700` Parse error (invalid JSON)
     - `-32600` Invalid request (malformed RPC structure)
     - `-32601` Method not found (requested tool/resource primitive missing)
     - `-32602` Invalid params (parameter validation failure)
     - `-32603` Internal error (unhandled server crash)
     - `-32000` to `-32099` Server-specific errors (rate limits, auth failures)
   - **Progress Tracking:** For long-running tools, clients pass a `progressToken` inside params. Servers issue intermediate updates via `notifications/progress` messages before returning the final JSON-RPC response.
   - **Cancellation Handling:** Either side can issue a `notifications/canceled` message with the `requestId` to stop in-flight operations when timeouts trigger or turns are aborted.
   - **Reserved Metadata (`_meta`):** Use the reserved `_meta` dictionary on requests and responses for context propagation (progress tokens, correlation IDs, session tracing, confidence scores) without breaking JSON-RPC 2.0 specification compliance.

6. **Scalability & Concurrency Design**
   - **Non-Blocking Async Event Loop:** Use Python's `asyncio` event loop for network and I/O-bound tool/resource calls.
   - **Offload Heavy Computation:** Spawning process workers or dispatching to task queues (e.g. Celery) for CPU-bound tasks (model inference, heavy document rendering) to avoid blocking the single-threaded event loop.
   - **Context Switch Efficiency:** Minimize unnecessary thread switching on Linux systems (~1–3 µs cost per switch) by leveraging async prefetching and batching operations.
   - **JSON-RPC 2.0 Batching:** Support processing arrays of JSON-RPC requests in a single HTTP payload to eliminate network handshake overhead across multiple tool calls.
   - **HTTP Payload Compression:** Enable `gzip` or `brotli` compression at the HTTP transport layer for large resource payloads to reduce network transfer time.

7. **Protocol-Level Structured Logging**
   - Declare logging capabilities to accept client `logging/setLevel` requests.
   - Emit `notifications/message` events containing severity levels, logger names, and structured JSON data for server diagnostic monitoring without polluting tool return values.

8. **Code Execution & Code Interpreter Tools**
   - Provide a specialized `@mcp.tool()` pattern for sandboxed code execution (Python interpreters) when tasks require high-precision deterministic results (e.g., high-precision pi computation, Fibonacci loops, complex logic) to bypass probabilistic LLM generation and prevent hallucinations.
   - Example code execution tool:
     ```python
     from decimal import Decimal, getcontext
     from mcp.server.fastmcp import FastMCP

     mcp = FastMCP("code-executor-server")

     @mcp.tool()
     def compute_high_precision_pi(precision: int = 50) -> str:
         """
         Execute Chudnovsky algorithm to compute high-precision Pi deterministically.

         Args:
             precision: Number of decimal digits of precision (default: 50)
         """
         getcontext().prec = precision + 10
         C = 426880 * Decimal(10005).sqrt()
         K, M, X, L, S = 0, 1, 1, 13591409, Decimal(13591409)
         for i in range(1, 100):
             M = M * (K**3 - 16 * K) // (i**3)
             K += 12
             L += 545140134
             X *= -262537412640768000
             S += Decimal(M * L) / X
         pi_val = C / S
         return str(pi_val)[: precision + 2]
     ```

9. **Constrained Decoding & Schema Conformance**
   - FastMCP tool signatures generate JSON Schema 2020-12 specifications. Enforce strict type annotations and Pydantic models to support constrained decoding (or "strict mode") on closed/open LLM inference engines (vLLM, OpenAI strict mode), reducing retries caused by malformed arguments.

10. **Tool Output Context Consumption & Hygiene**
    - Tool outputs (especially retrieval, web search, or database queries) can rapidly consume an LLM's context window.
    - Inside FastMCP tools, truncate large outputs, summarize extensive data blobs, or use cursor-based pagination (`nextCursor`) before returning results to client orchestrators.

11. **Keep the server focused and the tool boundary clean**
    - Recommend one server per coherent capability area (e.g., one server per external service or domain), keeping discovery output meaningful and dependencies scoped.
    - Load secrets (API keys, credentials) via environment variables (`os.getenv(...)`) rather than hardcoding them.

---

## Decision points and guidance

- **Is this capability better built as a custom server, or consumed from an
  existing community MCP server?** If a well-maintained server already
  exists for the target service, prefer using it directly (see
  python-fastmcp-client-integration) — build a custom server when the
  needed capability doesn't exist yet or must wrap internal/proprietary
  logic.
- **Does the function's docstring actually explain when to use the tool?**
  If it only restates the function name, strengthen it before shipping —
  this is the primary signal the LLM uses for tool selection.
- **Will this server run locally, remotely, or need to push updates?** Pick
  stdio, HTTP, or WebSocket accordingly rather than defaulting to stdio for
  every case.
- **Are secrets loaded from environment variables, not hardcoded?** Verify
  this before treating the server as ready to share or deploy.

---

## Quality criteria

A strong FastMCP server should ensure that:

- **tools are decorator-registered with minimal disruption** to the
  original function's logic,
- **schemas are accurate:** type hints correctly describe each parameter,
  including which are optional and their defaults,
- **descriptions are genuinely useful:** docstrings explain purpose,
  parameters, and return shape well enough for an LLM to select and call
  the tool correctly,
- **errors are handled gracefully:** failures return a readable message
  rather than crashing the server or the calling agent's turn,
- **the transport matches the deployment context:** stdio for local/dev,
  HTTP or WebSocket for remote/shared/production use,
- **secrets are externalized:** credentials come from environment
  variables, not hardcoded values.

---

## Example prompts

- "Turn this existing Python function into an MCP tool using FastMCP."
- "Review this MCP tool's docstring and type hints — will an LLM be able
  to tell when and how to use it?"
- "Should this MCP server use stdio or HTTP, given that we want to share it
  across multiple teammates' machines?"

---

## Related guidance

Use this tool alongside:

- python-fastmcp-manual-testing
- python-fastmcp-client-integration
- python-fastmcp-resource-providers
- python-fastmcp-prompt-providers
- python-fastmcp-security-discovery
- python-lang-graph-agent-suitability
- python-lang-graph-context-engineering-strategies
