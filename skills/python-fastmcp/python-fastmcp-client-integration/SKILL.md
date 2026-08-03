---
name: python-fastmcp-client-integration
description: Guides teams to connect to any MCP server programmatically using the MCP Python SDK — StdioServerParameters, stdio_client, and ClientSession for tool discovery (list_tools) and execution (call_tool), client feature management (dynamic sampling, sandboxed roots, human-in-the-loop elicitation), and converting MCP tool schemas into LLM provider formats.
---

# Python FastMCP: Client Integration

This skill helps AI write the programmatic MCP client code an agent uses to
discover and call tools from any MCP-compatible server — whether that's an
official server, a community-provided one, or a custom FastMCP server built
in-house. It covers the MCP Python SDK's connection and session pattern,
and how to convert MCP's tool schema format into whatever format the
target LLM provider's tool-calling API expects.

---

## When to use this skill

Use this skill when you need to:

- write the client-side code an agent uses to connect to an MCP server and
  discover its tools programmatically (as opposed to manual verification —
  see python-fastmcp-manual-testing for that),
- convert MCP tool definitions into the tool-calling format a specific LLM
  provider (e.g., OpenAI) expects,
- point the same client code at either an official/community MCP server or
  a custom FastMCP server, and understand why the integration code doesn't
  need to change between them,
- wire MCP-discovered tools into a LangGraph-based agent (see
  python-langgraph for the surrounding agent architecture).

---

## Outcome

Produce client integration code that:

- connects to an MCP server via the appropriate transport (stdio for local
  subprocess servers) using the MCP Python SDK,
- discovers available tools via `list_tools()` and executes them via
  `call_tool()`, handling the async lifecycle correctly,
- converts each MCP tool's `name`, `description`, and `inputSchema` into the
  target LLM provider's expected tool-definition format, without manually
  re-specifying anything the server already provided,
- works identically regardless of whether the server behind it is an
  official package, a community server, or an in-house FastMCP server —
  demonstrating MCP's actual value: decoupled tool implementation and tool
  usage.

---

## Instructions for the AI

1. **Install the MCP Python SDK**
   ```bash
   uv add mcp
   ```

2. **Connect to a server using Streamable HTTP (Production) or stdio (Local)**
   - **Streamable HTTP (Production):** Connect to remote or containerized production servers via SSE using `sse_client("http://<server-host>:<port>/sse")`. This leverages web infrastructure, OAuth 2.1, load balancing, and streaming responses.
     ```python
     import asyncio
     import os
     from mcp import ClientSession
     from mcp.client.sse import sse_client

     async def run_production_client():
         # Production connection over Streamable HTTP (SSE)
         async with sse_client("http://mcp-server.prod.internal:8000/sse") as (read_stream, write_stream):
             async with ClientSession(read_stream, write_stream) as session:
                 await session.initialize()
                 tools_result = await session.list_tools()
                 for tool in tools_result.tools:
                     print(f"  - {tool.name}: {tool.description[:60]}...")

                 result = await session.call_tool(
                     "search_documents",
                     arguments={"query": "financial results"},
                 )
                 print(result.content)
     ```
   - **stdio (Local Development / Inspector):** Use `StdioServerParameters` and `stdio_client(server_params)` when running local process servers during development or debugging.
     ```python
     from mcp import ClientSession, StdioServerParameters
     from mcp.client.stdio import stdio_client

     server_params = StdioServerParameters(
         command="uv",
         args=["run", "mcp_server.py"],
         env={"API_KEY": os.getenv("API_KEY")},
     )

     async with stdio_client(server_params) as (read_stream, write_stream):
         async with ClientSession(read_stream, write_stream) as session:
             await session.initialize()
             # Interaction proceeds identically via ClientSession
     ```
   - Note that regardless of whether `sse_client` or `stdio_client` is used, downstream session calls (`session.list_tools()`, `session.call_tool()`, `session.read_resource()`) operate identically — demonstrating transport-decoupled client design.

3. **Discover tools with `list_tools()`**
   - Call `await session.list_tools()` to retrieve all tool definitions the
     server currently exposes — each result includes `name`, `description`,
     and `inputSchema`.
   - Recommend doing this dynamically at agent startup (or per-session)
     rather than hardcoding a fixed tool list, so the agent automatically
     picks up new or changed tools the server adds later without requiring
     a code change on the client side.

4. **Execute tools with `call_tool()`**
   - Call `await session.call_tool(tool_name, arguments={...})`, passing
     the tool's name and a dictionary of arguments matching its schema.
   - Handle the returned `result.content` as the tool's output to feed back
     into the agent's context — this is the same execution-result content
     that context engineering strategies (see
     python-lang-graph-context-engineering-strategies) need to process
     before adding it to the model's context.

5. **Convert MCP tool schemas to the target LLM provider's format**
   - Recognize that `list_tools()` returns tool definitions in MCP's own
     schema format, which isn't necessarily what a given LLM provider's
     tool-calling API expects directly — a conversion step is needed.
   - Because MCP tool objects already expose exactly `name`, `description`,
     and `inputSchema`, this conversion is typically a direct mapping, not
     a rebuild — reuse whatever tool-definition formatting helper the
     project already has for hand-built tools:
     ```python
     def mcp_tools_to_openai_format(mcp_tools) -> list[dict]:
         """Convert MCP tool definitions to OpenAI tool format."""
         return [
             format_tool_definition(
                 name=tool.name,
                 description=tool.description,
                 parameters=tool.inputSchema,
             )
             for tool in mcp_tools.tools
         ]
     ```
   - Recommend keeping this conversion function provider-specific and
     isolated (one small function per target format), so switching or
     adding LLM providers doesn't require touching the MCP connection code
     itself.

6. **Enforce RFC 8707 Resource Indicators for OAuth 2.1**
   - In production HTTP environments where MCP servers act as OAuth 2.1 resource servers, client token requests MUST include a resource indicator (RFC 8707) specifying the target server's canonical URI (`resource=https://mcp.example.com`).
   - This binds the issued access token's audience (`aud`) claim strictly to that specific server, preventing token replay attacks against un-intended third-party MCP servers.

7. **Integrate with LangGraph via Adapters (`langchain-mcp-adapters`)**
   - Connect LangGraph state graphs to multi-server FastMCP setups cleanly using `MultiServerMCPClient`.
   - Pattern for LangGraph integration:
     ```python
     from langchain_mcp_adapters.client import MultiServerMCPClient
     from langgraph.prebuilt import create_react_agent
     from langchain_openai import ChatOpenAI

     async def build_mcp_agent():
         # Configure multi-server client across transports
         async with MultiServerMCPClient({
             "math": {"command": "uv", "args": ["run", "math_server.py"], "transport": "stdio"},
             "weather": {"url": "http://weather.prod.internal:8000/sse", "transport": "sse"},
         }) as mcp_client:
             # Dynamically discover and convert tools
             tools = mcp_client.get_tools()
             model = ChatOpenAI(model="gpt-4o", temperature=0.0)
             
             # Wire into LangGraph React Agent
             agent = create_react_agent(model, tools)
             response = await agent.ainvoke({"messages": [("user", "What's the weather in Tokyo?")]})
             return response
     ```

8. **Maintain Context Isolation and Privacy Boundaries**
   - Each client instance maintains an isolated session with a single server.
   - **Snippet Hygiene:** Never send the full conversation history to an MCP server. Send only minimal, relevant contextual snippets required for tool execution or resource querying to prevent information leakage across servers.

9. **Implement Intelligent Context Retrieval & Caching**
   - **Similarity Thresholds:** Use vector embedding similarity filtering (cosine threshold between 0.3 and 0.8 depending on domain/embedding model) for candidate context retrieval.
   - **Progressive Loading:** Load high-level summaries first; fetch detailed document contents (`doc://{id}`) on demand.
   - **Context Caching:** Cache processed insights and entity relationships across turns to conserve context window tokens and reduce retrieval latency.

10. **Implement Client-Side Sampling Control**
    - Configure sampling parameters (`temperature`, `top_p`, `stop_sequences`) dynamically per turn.
    - **Dynamic Sampling Policy:** Use higher temperature (e.g. `temperature=0.9`, `top_p=0.95`) during creative brainstorming or drafting turns. Reduce temperature to `0.0` when preparing to invoke a tool call to ensure generated arguments strictly conform to JSON schema definitions and behave deterministically.

11. **Manage Sandboxed Filesystem Roots**
    - Roots expose sandboxed filesystem directories accessible across model turns for reading/writing intermediate files, code snippets, or session state.
    - Define allowed directory boundaries when initializing session roots. Ensure roots are sandboxed so the model cannot access arbitrary host files outside specified session boundaries.
    - Configure session persistence policies (e.g., retaining roots across turns or discarding them when the session terminates).

12. **Handle Human-in-the-Loop Elicitation**
    - Elicitation allows an MCP server to request information or explicit authorization directly from the end user when the model cannot proceed autonomously (e.g. payment processing, credit card charges, or deleting resources).
    - Client execution flow:
      1. Server returns an elicitation request during a tool call specifying required fields/approvals.
      2. Client pauses the model turn and renders an interactive prompt to the user.
      3. Client captures the user's response or consent.
      4. Client resumes the tool call passing the user's input payload.

13. **Autoregressive Tool Orchestration Loop & Message Interleaving**
    - The standard client orchestration loop receives tool calls from the model, executes them via FastMCP `ClientSession.call_tool()`, and appends tool output messages (`{"role": "tool", "tool_call_id": call.id, "content": result}`) back into the conversation history:
      ```python
      async def autoregressive_tool_loop(model_client, session, messages, tools):
          """Standard autoregressive orchestration loop for multi-turn FastMCP tool calls."""
          while True:
              response = await model_client.generate(messages=messages, tools=tools)
              if not response.tool_calls:
                  return response.text

              # Execute discovered FastMCP tools
              for call in response.tool_calls:
                  result = await session.call_tool(call.name, call.args)
                  messages.append({
                      "role": "tool",
                      "tool_call_id": call.id,
                      "content": result.content[0].text if result.content else ""
                  })
      ```

14. **Reasoning Token Continuity & Interleaving**
    - **Reasoning/Thinking Tokens (`<thinking>`, OpenAI `o1`/`o3`):** When interacting with reasoning models, preserve reasoning token streams between tool calling steps within a single turn to maintain multi-step logical context across tool calls.
    - Erase or summarize past turn reasoning tokens between separate user turns to minimize serving costs while preserving task history.

15. **Client-Side Context Window Hygiene**
    - Tool outputs (especially retrieval search results or database dumps) can rapidly consume context window budgets.
    - Client orchestrators MUST inspect tool return payloads and apply truncation, summarization, or pagination before appending them as `role: "tool"` messages to prevent context exhaustion and token cost spikes.

16. **Mitigate Latency Cascades via Parallel Orchestration & Preloading**
    - **Parallel Execution (`asyncio.gather`):** Eliminate the "latency cascade problem" by executing independent tool calls and resource reads concurrently rather than sequentially.
      ```python
      # Execute independent tool calls concurrently to eliminate cumulative network latency
      results = await asyncio.gather(
          session.call_tool("get_user_profile", {"user_id": "u123"}),
          session.call_tool("get_account_balance", {"user_id": "u123"}),
          session.read_resource("analytics://summary"),
      )
      ```
    - **Predictive Context Preloading:** Pre-load likely downstream resource URIs (`doc://{id}`) based on past interaction patterns before the model explicitly requests them.
    - **Speculative Execution & Cancellation:** Issue speculative tool requests for high-probability downstream paths, using `notifications/canceled` to abort speculative requests if the agent selects an alternate turn path.

---

## Decision points and guidance

- **Is this server local/dev-only, or does it need to run remotely or be
  shared?** stdio via `StdioServerParameters`/`stdio_client` covers the
  local subprocess case; a remote/shared server needs the corresponding
  HTTP or WebSocket client setup instead (see python-fastmcp-server-basics
  for the transport tradeoffs).
- **Is the tool list being hardcoded, or discovered dynamically?** Prefer
  `list_tools()` at runtime so server-side tool changes don't require
  client-side updates.
- **Has this server already been verified manually?** Confirm it's passed
  an Inspector check (python-fastmcp-manual-testing) before debugging
  programmatic integration issues — this rules out schema/server problems
  before assuming the client code is at fault.
- **Does the conversion function need to change per LLM provider?** Keep
  provider-specific formatting isolated in its own small function rather
  than baked into the connection/session code.

---

## Quality criteria

A strong MCP client integration should ensure that:

- **connection lifecycle is handled correctly:** `stdio_client` and
  `ClientSession` are both used within `async with`, and `initialize()` is
  called before any other request,
- **tools are discovered dynamically:** `list_tools()` drives the available
  tool set rather than a hardcoded list,
- **the same client code works against any compliant server:** switching
  between an official, community, or custom FastMCP server only requires
  changing `StdioServerParameters`, not the session logic,
- **schema conversion is accurate and isolated:** MCP's `name`/
  `description`/`inputSchema` map cleanly to the target provider's format,
  in a small function separate from connection logic,
- **execution results are handled appropriately** for how they'll be used
  downstream (e.g., fed into an agent's context).

---

## Example prompts

- "Write the MCP client code to connect to this FastMCP server and list
  its available tools."
- "Convert these MCP tool definitions into the format our OpenAI-based
  agent expects."
- "We're switching from the official Tavily MCP server to our own custom
  one — what actually needs to change in our client code?"

---

## Related guidance

Use this tool alongside:

- python-fastmcp-server-basics
- python-fastmcp-manual-testing
- python-lang-graph-context-engineering-strategies
- python-lang-graph-agentic-architectures
