---
name: python-fastmcp-manual-testing
description: Guides teams to manually verify an MCP server before wiring it into any client or agent, using the official MCP Inspector (npx @modelcontextprotocol/inspector) to connect, list tools, inspect generated schemas, and run tools interactively in the browser. This is the required first verification step for any new or changed FastMCP server, catching bad schemas and descriptions before they reach an agent.
---

# Python FastMCP: Manual Testing with MCP Inspector

This skill helps AI guide a developer through manually verifying an MCP
server using the MCP Inspector — a browser-based tool for connecting to a
server, listing its tools, inspecting the auto-generated schema for each
one, and running tools interactively with real arguments. Every new or
changed FastMCP server should go through this check before it's wired into
a client or agent, since it's the fastest way to catch a bad schema,
misleading description, or broken tool before those problems surface as a
confusing agent failure downstream.

---

## When to use this skill

Use this skill when you need to:

- verify a newly built or modified FastMCP server actually works before
  writing any client integration code,
- confirm the schema and description FastMCP generated from a tool's type
  hints and docstring look correct and LLM-usable,
- debug a tool that an agent is calling incorrectly, by first checking
  whether the problem is visible directly in the Inspector,
- explore an unfamiliar third-party MCP server (e.g., a community-provided
  one) before integrating it,
- onboard a teammate to the team's MCP server development workflow.

---

## Outcome

Produce a verification pass that:

- launches the target MCP server through MCP Inspector without requiring
  any custom client code to be written first,
- confirms, via the Tools tab, that every expected tool appears, with
  accurate names, descriptions, and parameter schemas,
- exercises at least one real call per tool with realistic arguments,
  confirming the tool executes and returns a sensible result,
- catches schema or description problems (missing parameters, unclear
  descriptions, wrong types) while they're cheap to fix — before any
  client or agent integration work depends on them,
- gives the team a repeatable, low-friction habit: every new or changed
  server gets an Inspector pass before it's considered done.

---

## Instructions for the AI

1. **Launch the server through MCP Inspector**
   - For a stdio-based FastMCP server, launch it with:
     ```bash
     npx @modelcontextprotocol/inspector uv run <server_file>.py
     ```
     (substitute the actual run command for the server — e.g., `uv run
     tavily_mcp_server.py` for a `uv`-managed project, or `python
     server.py` if that's how the project runs it).
   - For an existing third-party server distributed as an npm package
     (useful for exploring or comparing against a custom server), launch
     it the same way:
     ```bash
     npx @modelcontextprotocol/inspector npx -y <package-name>@latest
     ```
   - Explain what's happening: `npx @modelcontextprotocol/inspector`
     starts a local web interface for interacting with MCP servers; the
     command after it is what actually launches the target server as a
     subprocess, exactly as a real MCP Client would.
   - Note that any environment variables the server needs (API keys,
     credentials) must be set in the shell before running this command
     (e.g., `export TAVILY_API_KEY=<key>`), since the Inspector launches
     the server as a subprocess of the current shell environment.

2. **Connect and confirm the server is reachable**
   - Open the URL printed in the terminal output (it typically includes a
     session token, allowing connection without further configuration).
   - In the Inspector UI, click **Connect**. A successful connection
     reveals the server's capability tabs — Resources, Prompts, and Tools
     — reflecting whatever the server actually exposes.
   - If the connection fails, treat this as the first and most basic
     signal something is wrong — check that the launch command is correct,
     required environment variables are set, and the server starts
     without raising an exception on its own before assuming the problem
     is protocol-related.

3. **Inspect every tool's schema and description**
   - Open the **Tools** tab and click **List Tools** — confirm every tool
     the server is expected to expose actually appears.
   - Click into each tool individually and review its generated
     description and parameter schema. Check specifically:
     - Does the description clearly explain what the tool does and when
       it should be used (not just restate the function name)?
     - Are all parameters present, with the correct types and optional/
       required status?
     - Do default values for optional parameters look correct?
   - Treat any mismatch here as a signal to go back and fix the
     originating function's type hints or docstring (see
     python-fastmcp-server-basics) — Inspector is showing exactly what an
     LLM client would see, so problems visible here will directly degrade
     an agent's ability to select and call the tool correctly.

4. **Run each tool with realistic arguments**
   - For each tool, enter representative input values and click **Run
     Tool** — confirm the server executes and returns a sensible,
     well-formatted result directly in the browser.
   - Test at least one deliberately invalid or edge-case input per tool
     (e.g., an empty query, an out-of-range parameter) to confirm the
     server's error handling returns a readable message rather than
     crashing or hanging — this is verifying the same error-handling
     behavior recommended in python-fastmcp-server-basics.
   - Treat this step as non-optional even for simple tools — the Inspector
     surfaces execution problems (exceptions, malformed output, hanging
     calls) far faster than discovering them through a client integration
     or, worse, in front of an actual agent run.

5. **Make this a standard step, not a one-off debugging tool**
   - Recommend running an Inspector pass on every new server before
     writing any client integration code, and on every meaningfully
     changed server (new tool, changed signature, changed docstring)
     before considering the change done.
   - When debugging an agent that's misusing a tool (wrong arguments,
     wrong tool selected, unexpected failure), recommend starting with an
     Inspector pass on the tool in question — this isolates whether the
     problem is in the tool/schema itself or in the agent's reasoning
     around it, before spending time debugging the agent side.
   - Frame this as a fast, code-free verification loop: no client code,
     no agent wiring, no LLM calls required — just the server and a
     browser — making it the cheapest possible place to catch a problem.

---

## Decision points and guidance

- **Does the server connect and list all expected tools?** If not, debug
  the server launch itself before assuming anything about MCP client
  integration.
- **Does each tool's Inspector-rendered description explain when to use
  it, not just what it's called?** If not, go fix the docstring, not the
  client code.
- **Does every tool return a sensible result for realistic input, and a
  readable error for bad input?** Confirm both, not just the happy path.
- **Is an agent misbehaving with a tool?** Check the tool directly in
  Inspector first — this quickly rules in or out the tool/schema as the
  cause before debugging agent reasoning.
- **Has this server or tool changed since its last Inspector pass?** If
  so, treat it as unverified until it's been re-checked.

---

## Quality criteria

A strong manual-testing pass should confirm that:

- **the server launches and connects cleanly** through Inspector with no
  custom code required,
- **every expected tool is discoverable** and correctly listed,
- **every tool's schema and description are accurate and LLM-usable** as
  rendered in the Inspector UI,
- **every tool has been executed at least once** with realistic input and
  produces a sensible result,
- **error handling has been exercised**, not just the happy path,
- **this check happens routinely** — for every new server and every
  meaningful change to an existing one, not only when something is already
  suspected to be broken.

---

## Example prompts

- "Walk me through testing this new FastMCP server with MCP Inspector
  before I write any client code for it."
- "Our agent keeps calling this tool with the wrong arguments — help me
  check the tool's schema in Inspector to see if that's the actual
  problem."
- "I changed this tool's docstring — how do I verify the new description
  looks right before shipping it?"

---

## Related guidance

Use this tool alongside:

- python-fastmcp-server-basics
- python-fastmcp-client-integration
