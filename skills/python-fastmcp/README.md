# Python — FastMCP

MCP (Model Context Protocol) server development with FastMCP: building tools (`@mcp.tool()`), resource providers (`@mcp.resource()`), prompt providers (`@mcp.prompt()`), security middleware & discovery registries, manual verification with MCP Inspector, and programmatic client integration for agents.

For projects that build or consume MCP servers. Pair with `python-langgraph` (or another agent framework) for the agent side that consumes these tools.

## Install

```bash
npx skills add mihailtd/skills/skills/python-fastmcp --all
```

Add `-g` to install globally instead of per-project. Use `--skill <name>` (repeatable) instead of `--all` to cherry-pick individual skills from this package, e.g.:

```bash
npx skills add mihailtd/skills/skills/python-fastmcp --skill python-fastmcp-resource-providers
```

## Skills (14)

- **[python-fastmcp-server-basics](python-fastmcp-server-basics/SKILL.md)** — Building MCP servers with FastMCP: the `@mcp.tool()` decorator, core primitives (Tools, Resources, Prompts), writing type hints and docstrings that become the tool's schema and description, choosing a transport (stdio/HTTP/WebSocket), and the Host/Client/Server architecture.
- **[python-fastmcp-resource-providers](python-fastmcp-resource-providers/SKILL.md)** — Building read-only MCP Resource Providers: exposing data, static documents (`doc://`), dynamic collections (`docs://recent`), computed query URIs (`docs://search`), MIME types, and transitioning from in-memory dictionary stores to persistent databases (PostgreSQL/MongoDB).
- **[python-fastmcp-prompt-providers](python-fastmcp-prompt-providers/SKILL.md)** — Building MCP Prompt Providers with `@mcp.prompt()`: creating reusable, parameterized prompt templates, structuring `system` and `user` roles, dynamic context injection, and guiding LLM behavior to invoke tools over guessing.
- **[python-fastmcp-security-discovery](python-fastmcp-security-discovery/SKILL.md)** — Implementing MCP security controls and service discovery: API key authentication middleware, sliding-window rate limiting, request/response audit logging, and discovery registries (`register_server`, `discover_servers` by capability/tag).
- **[python-fastmcp-multi-agent-orchestration](python-fastmcp-multi-agent-orchestration/SKILL.md)** — Designing multi-agent AI systems with FastMCP and LangGraph: Orchestrator, Marketplace, and Specialized Team patterns, wiring agents via LangGraph StateGraphs to shared FastMCP servers, dynamic capability discovery (`listChanged`), and multi-agent trust models.
- **[python-fastmcp-rag-retrieval](python-fastmcp-rag-retrieval/SKILL.md)** — Building Retrieval-Augmented Generation (RAG) pipelines with FastMCP: exposing document stores via resources/tools, multi-stage retrieval across servers, leveraging metadata annotations (`audience`, `priority`, `lastModified`), dynamic query expansion, source attribution, and graceful fallback degradation.
- **[python-fastmcp-langgraph-integration](python-fastmcp-langgraph-integration/SKILL.md)** — Integrating FastMCP servers into LangGraph applications: binding FastMCP tools to LangGraph StateGraphs via `langchain-mcp-adapters` (`MultiServerMCPClient`), reading resources into vector memory, handling `listChanged` notifications, and enforcing correlation ID tracing across graph nodes.
- **[python-fastmcp-enterprise-knowledge-management](python-fastmcp-enterprise-knowledge-management/SKILL.md)** — Building Enterprise Knowledge Management (EKM) systems with FastMCP: federated search across SharePoint/Confluence/CRMs, cursor-based pagination (`nextCursor`), document classification (`confidential`/`internal`/`public`), field redaction, and Knowledge Graph traversal.
- **[python-fastmcp-personalization-recommendations](python-fastmcp-personalization-recommendations/SKILL.md)** — Building Personalization and Recommendation engines with FastMCP: aggregating unified user profiles (`profile://`, `behavior://`, `content://`), real-time context boosts, multi-factor scoring, explainability metadata, and cold-start fallback strategies.
- **[python-fastmcp-multimodal-applications](python-fastmcp-multimodal-applications/SKILL.md)** — Building Multimodal AI Applications (vision, text, audio, tabular data) with FastMCP: Adaptive Routing, Orchestrated Pipelines, Collaborative Processing, cross-modal context sharing, confidence propagation, and multi-session synthesis.
- **[python-fastmcp-evaluation-benchmarking](python-fastmcp-evaluation-benchmarking/SKILL.md)** — Building Evaluation Frameworks and Automated Benchmarks for FastMCP systems: testing capability discovery, P95 response distributions, concurrent throughput (`asyncio.gather`), fault recovery, LLM-as-a-Judge quality, and persistent SQLite benchmark logging.
- **[python-fastmcp-performance-optimization](python-fastmcp-performance-optimization/SKILL.md)** — Optimizing FastMCP performance across server, client, and protocol layers: Context-Aware LRU Caching, Capability-Aware Load Balancing, DAG Request Orchestration, HTTP Compression/Connection Pooling, and Sliding-Window Latency Monitors.
- **[python-fastmcp-manual-testing](python-fastmcp-manual-testing/SKILL.md)** — Manually verifying an MCP server with MCP Inspector (`npx @modelcontextprotocol/inspector`) before wiring it into any client or agent — connecting, listing tools, inspecting generated schemas, and running tools interactively in the browser. Required first step for any new or changed server.
- **[python-fastmcp-client-integration](python-fastmcp-client-integration/SKILL.md)** — Connecting to any MCP server programmatically with the MCP Python SDK (`StdioServerParameters`, `stdio_client`, `ClientSession`) for tool discovery and execution, and converting MCP tool schemas into an LLM provider's tool-calling format.

## License

MIT
