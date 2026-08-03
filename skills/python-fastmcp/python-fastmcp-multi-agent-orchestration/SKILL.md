---
name: python-fastmcp-multi-agent-orchestration
description: Guides teams to build multi-agent AI systems with FastMCP and LangGraph — implementing Orchestrator, Marketplace, and Specialized Team coordination patterns, wiring LangGraph StateGraphs to shared FastMCP servers, handling dynamic capability discovery (listChanged), parallel task execution, and multi-agent security/reputation models.
---

# Python FastMCP: Multi-Agent Orchestration with LangGraph

This skill helps AI design and build multi-agent systems where specialized autonomous agents (implemented in **LangGraph**) collaborate using FastMCP servers. By standardizing tools, resources, and prompt interfaces, FastMCP eliminates fragile glue code and enables dynamic capability discovery, task allocation, and concurrent execution across LangGraph graphs (`StateGraph`, `create_react_agent`).

---

## When to use this skill

Use this skill when you need to:

- build a multi-agent system where specialized LangGraph agents (e.g. Research, Analysis, Synthesis, Writing) collaborate to achieve complex goals,
- implement multi-agent coordination patterns (**Orchestrator**, **Marketplace**, **Collaborative Filtering**, **Specialized Team**),
- connect LangGraph nodes to FastMCP servers via `MultiServerMCPClient` (`langchain-mcp-adapters`),
- handle dynamic server capability changes via `notifications/tools/listChanged` notifications,
- coordinate parallel tool execution across agents (`asyncio.gather`),
- enforce context isolation and privacy boundaries so agents only receive minimal, relevant context snippets,
- establish reputation-based trust and federated authorization across multi-agent toolchains.

---

## Outcome

Produce multi-agent LangGraph orchestration code that:

- delegates domain tasks to specialized LangGraph nodes connected to dedicated or shared FastMCP servers,
- coordinates workflow execution using standard patterns (Orchestrator for hierarchical workflows, Marketplace for capability bidding),
- executes independent agent tool calls concurrently using `asyncio.gather` to mitigate latency cascades,
- listens for `listChanged` notifications to adapt tool availability dynamically at runtime without restarting agents,
- maintains context isolation so agents never leak unneeded conversation history across server boundaries.

---

## Instructions for the AI

1. **Select the Right Multi-Agent Coordination Pattern**
   - **Orchestrator Pattern:** A central coordinator node parses user intent into discrete workflow steps, delegates subtasks to specialized worker agent nodes via FastMCP tools, and synthesizes output. Best for structured, hierarchical workflows.
   - **Marketplace Pattern:** Worker agents publish capabilities to a FastMCP registry; client nodes query capabilities (`tools/list`) and select the best provider based on reputation or cost. Best for dynamic, open ecosystems.
   - **Specialized Team Pattern:** Agents form functional units (e.g., Ideation -> Composition -> Critic) collaborating internally via shared FastMCP resources and externally via higher-level tools.
   - **Collaborative Filtering Pattern:** Agents attach metadata ratings and usage feedback to resources, recommending effective tools and papers to peer agents.

2. **Orchestrate Multi-Agent Tool Execution in LangGraph (`langchain-mcp-adapters`)**
   - Connect LangGraph agent graphs to FastMCP servers via `MultiServerMCPClient`.
   - Example pattern:
     ```python
     import asyncio
     from typing import Annotated, TypedDict
     from langchain_mcp_adapters.client import MultiServerMCPClient
     from langchain_openai import ChatOpenAI
     from langgraph.graph import StateGraph, START, END
     from langgraph.graph.message import add_messages

     class AgentState(TypedDict):
         messages: Annotated[list, add_messages]
         research_output: str
         analysis_output: str

     async def run_langgraph_mcp_workflow():
         # 1. Connect to FastMCP servers via Streamable HTTP (SSE)
         async with MultiServerMCPClient({
             "research_server": {"url": "http://research-mcp.prod.internal:8000/sse", "transport": "sse"},
             "analytics_server": {"url": "http://analytics-mcp.prod.internal:8000/sse", "transport": "sse"},
         }) as client:
             tools = client.get_tools()
             model = ChatOpenAI(model="gpt-4o").bind_tools(tools)

             # 2. Define LangGraph nodes
             async def research_node(state: AgentState):
                 response = await model.ainvoke([
                     {"role": "system", "content": "You are a Research Specialist. Search and summarize documents."},
                     *state["messages"]
                 ])
                 return {"messages": [response], "research_output": str(response.content)}

             async def analysis_node(state: AgentState):
                 response = await model.ainvoke([
                     {"role": "system", "content": "You are an Analytics Specialist. Analyze dataset metrics."},
                     *state["messages"]
                 ])
                 return {"messages": [response], "analysis_output": str(response.content)}

             # 3. Build LangGraph StateGraph
             builder = StateGraph(AgentState)
             builder.add_node("research", research_node)
             builder.add_node("analysis", analysis_node)
             builder.add_edge(START, "research")
             builder.add_edge("research", "analysis")
             builder.add_edge("analysis", END)

             graph = builder.compile()
             result = await graph.ainvoke({"messages": [{"role": "user", "content": "Analyze quantum computing trends."}]})
             print(result)
     ```

3. **Handle Dynamic Discovery & `listChanged` Events**
   - Subscribe to server capability change events (`notifications/tools/listChanged`).
   - Re-query `client.get_tools()` when notifications trigger so LangGraph node tool bindings update dynamically without process restarts.

4. **Enforce Context Isolation & Privacy Boundaries**
   - Enforce single stateful session isolation per client-server pair.
   - Pass only minimal, purpose-specific context snippets to each server rather than full conversation histories to prevent cross-agent data leakage.

5. **Implement Multi-Agent Trust & Federated Security**
   - Require OAuth 2.1 RFC 8707 Resource Indicators for all HTTP server token requests (`resource=https://server-uri`).
   - Track agent reputation scores based on task completion and peer ratings.
   - Intercept high-risk tool calls with explicit human-in-the-loop consent before execution.

6. **Capability Composition & Composite Solutions**
   - Implement dynamic capability composition (`discover_capability_compositions`) where complex user requirements are satisfied by combining complementary capability types (`LANGUAGE_MODEL`, `MULTIMODAL_PROCESSOR`, `REASONING_ENGINE`, `KNOWLEDGE_BASE`).

7. **Track Emergent Behaviors & Enforce Governance Guardrails**
   - Log emergent collaborative behaviors (`emergent_insight`) and novelty metrics when multi-agent interaction loops produce unprompted workflow improvements.
   - Enforce AI Governance compliance (EU AI Act, NIST AI Risk Management Framework, ISO 42001) by maintaining audit logs of all inter-agent tool calls and data lineage.

8. **Edge AI & Peer-to-Peer Agent Networks**
   - Support peer-to-peer agent routing and offline execution for edge deployments (mobile, IoT, local workstations).
   - Require zero-trust access controls, local payload encryption, and secure boot verification for edge-deployed agents.

---

## Decision points and guidance

- **Orchestrator vs Marketplace?** Use Orchestrator when the sequence of tasks is predictable and hierarchical in LangGraph. Use Marketplace when capabilities change dynamically or agents are provided by different organizations.
- **Shared vs Dedicated FastMCP Servers?** Specialized agents can share a single FastMCP server instance if role-based access control is enforced, or connect to separate domain servers for physical isolation.
- **How to prevent agent deadlocks?** Enforce timeouts on agent tool calls and implement fallback retries or human-in-the-loop intervention if a LangGraph node stalls.

---

## Quality criteria

- **Clean Decoupling:** Agents communicate through standard FastMCP tool schemas without custom per-agent glue code.
- **Concurrent Efficiency:** Independent agent subtasks execute concurrently via `asyncio.gather` or parallel LangGraph branches.
- **Context Hygiene:** Context window payloads are strictly scoped per agent role.
- **Resilient Discovery:** LangGraph node registries handle runtime tool updates via `listChanged` notifications.

---

## Example prompts

- "Build a multi-agent research team in LangGraph where a Research node and an Analysis node share a FastMCP server."
- "Implement an Orchestrator pattern in LangGraph that delegates subtasks to FastMCP tools concurrently."
- "Set up dynamic capability discovery listening for `listChanged` notifications in our LangGraph agent client."

---

## Related guidance

- python-fastmcp-server-basics
- python-fastmcp-client-integration
- python-fastmcp-security-discovery
- python-langgraph
