---
name: python-fastmcp-langgraph-integration
description: Guides teams to integrate FastMCP servers into LangGraph applications — binding FastMCP tools to LangGraph StateGraphs via langchain-mcp-adapters (MultiServerMCPClient), loading resources into vector memory, handling listChanged notifications, and enforcing correlation ID tracing across graph nodes.
---

# Python FastMCP: LangGraph Integration

This skill helps AI integrate FastMCP servers into **LangGraph** agent applications. Rather than writing brittle custom loaders, tools, and memory backends for every database or API, developers leverage standardized FastMCP JSON-RPC endpoints to expose document collections, agent tools, and persistent memory to LangGraph state graphs (`StateGraph`, `create_react_agent`) with dynamic runtime discovery (`MultiServerMCPClient`).

---

## When to use this skill

Use this skill when you need to:

- connect LangGraph agents or state graphs to external data and tools via standardized FastMCP interfaces,
- convert FastMCP resource providers (`@mcp.resource()`) into document loaders for vector store indexing inside LangGraph nodes,
- dynamically expose FastMCP tools as native LangGraph tools using `langchain-mcp-adapters` (`MultiServerMCPClient`),
- handle runtime tool updates via `notifications/tools/listChanged` notifications to refresh LangGraph node toolkits without restarting processes,
- use FastMCP resources as persistent memory backends across LangGraph thread executions,
- instrument production LangGraph + FastMCP graph nodes with correlation ID tracing and schema contract tests.

---

## Outcome

Produce a LangGraph application that:

- connects to one or more Streamable HTTP (SSE) FastMCP servers using `MultiServerMCPClient`,
- binds discovered FastMCP tools directly to LangGraph `ChatOpenAI`/`ChatAnthropic` model invocations within graph nodes,
- reads FastMCP resources into vector stores or graph state,
- dynamically adapts node tool bindings when FastMCP servers emit `listChanged` events.

---

## Instructions for the AI

1. **Bind FastMCP Tools to LangGraph Model Invocations (`langchain-mcp-adapters`)**
   - Connect to FastMCP servers over Streamable HTTP (`SSE`) using `MultiServerMCPClient`.
   - Pass `client.get_tools()` to `.bind_tools()` when defining LangGraph node functions.
   - Example implementation:
     ```python
     import asyncio
     from typing import Annotated, TypedDict
     from langchain_mcp_adapters.client import MultiServerMCPClient
     from langchain_openai import ChatOpenAI
     from langgraph.graph import StateGraph, START, END
     from langgraph.graph.message import add_messages

     class State(TypedDict):
         messages: Annotated[list, add_messages]

     async def run_langgraph_mcp_agent():
         # Connect to production FastMCP servers via SSE transport
         async with MultiServerMCPClient({
             "documents": {"url": "http://docs-mcp.prod.internal:8000/sse", "transport": "sse"},
             "analytics": {"url": "http://analytics-mcp.prod.internal:8000/sse", "transport": "sse"},
         }) as client:
             tools = client.get_tools()
             model = ChatOpenAI(model="gpt-4o").bind_tools(tools)

             async def agent_node(state: State):
                 response = await model.ainvoke(state["messages"])
                 return {"messages": [response]}

             # Build LangGraph StateGraph
             workflow = StateGraph(State)
             workflow.add_node("agent", agent_node)
             workflow.add_edge(START, "agent")
             workflow.add_edge("agent", END)

             app = workflow.compile()
             result = await app.ainvoke({"messages": [{"role": "user", "content": "Fetch Q4 revenue metrics"}]})
             print(result)
     ```

2. **Load FastMCP Resources into Vector Memory for LangGraph RAG**
   - Wrap `@mcp.resource()` calls in document loading utilities to populate vector stores (Chroma, Qdrant, FAISS) for RAG nodes in LangGraph.
   - Example resource loader:
     ```python
     import json
     from typing import List
     from mcp import ClientSession
     from langchain_core.documents import Document

     async def load_mcp_resources_for_langgraph(session: ClientSession, resource_uris: List[str]) -> List[Document]:
         """Load documents from connected FastMCP servers for LangGraph vector indexing."""
         documents = []
         for uri in resource_uris:
             result = await session.read_resource(uri)
             for content in result.contents:
                 if hasattr(content, "text"):
                     documents.append(Document(
                         page_content=content.text,
                         metadata={"source": uri, "mime_type": content.mimeType}
                     ))
         return documents
     ```

3. **Handle Dynamic Tool Refresh (`listChanged`) in LangGraph**
   - Listen for `notifications/tools/listChanged` events emitted by FastMCP servers.
   - Refresh `client.get_tools()` and re-bind to the LLM model instance when tool schemas update at runtime.

4. **Propagate Correlation IDs across Graph Nodes (`_meta`)**
   - Include a unique `correlation_id` in request metadata (`_meta={"correlation_id": "tx-9012"}`) when calling tools from LangGraph nodes.

---

## Decision points and guidance

- **`MultiServerMCPClient` vs Raw SDK?** Always use `MultiServerMCPClient` from `langchain-mcp-adapters` for LangGraph agent integrations.
- **How to manage multiple document servers in LangGraph?** Maintain server configurations in `MultiServerMCPClient` dict and merge loaded `Document` lists into a single vector store node in the StateGraph.

---

## Quality criteria

- **LangGraph Native:** Graph nodes use `MultiServerMCPClient` tools bound directly to model invocations.
- **Production SSE Transport:** Server connections default to Streamable HTTP (`sse`).
- **Dynamic Refresh:** Tool bindings update on `listChanged` notifications without restarting the agent graph process.
- **Correlation Traced:** Tool call metadata includes `correlation_id` tracking across graph state transitions.

---

## Example prompts

- "Connect a LangGraph StateGraph to a FastMCP server using MultiServerMCPClient."
- "Load FastMCP resources into a vector store node for a LangGraph RAG workflow."
- "Handle listChanged notifications to dynamically refresh tool bindings in a LangGraph agent."

---

## Related guidance

- python-fastmcp-server-basics
- python-fastmcp-client-integration
- python-fastmcp-multi-agent-orchestration
- python-langgraph
