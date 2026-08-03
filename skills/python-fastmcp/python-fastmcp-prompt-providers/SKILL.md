---
name: python-fastmcp-prompt-providers
description: Guides teams to build MCP Prompt Providers in Python using FastMCP — creating reusable, parameterized prompt templates that structure system and user roles, inject dynamic context, and guide model behavior toward tool execution instead of hallucinated answers.
---

# Python FastMCP: Prompt Providers

This skill helps AI build MCP Prompt Providers — precomposed message templates that guide an LLM agent's role, rules, boundaries, and workflow. Prompt providers allow server authors to standardize system instructions (e.g. policies for interacting with internal domain services) and user interaction patterns (e.g. daily summaries, standup reports) across agent sessions.

---

## When to use this skill

Use this skill when you need to:

- expose reusable system prompts or multi-message templates to MCP clients,
- parameterize prompts with optional arguments (`userName`, `date`, `timeZone`),
- separate system-level guidelines (`role: system`) from task-level user instructions (`role: user`),
- ensure an LLM invokes specific tools (`search_flights`, `send_email`) rather than hallucinating responses,
- version prompt templates for backward compatibility as instructions evolve.

---

## Outcome

Produce a FastMCP prompt provider that:

- advertises available prompt templates via catalog metadata (`id`, `displayName`, `description`),
- builds parameterized prompt messages containing structured role assignments (`system`, `user`, `assistant`),
- dynamically injects context from underlying data stores or incoming argument dictionaries,
- explicitly instructs models to call server tools when domain actions or facts are required.

---

## Instructions for the AI

1. **Understand the Structure of MCP Prompts**
   - Prompts expose two primary operations:
     - `list_prompts()`: Returns a catalog of prompt metadata dictionaries (`id`, `displayName`, `description`).
     - `get_prompt(prompt_id, args)`: Returns an array of prompt messages, where each message contains a `role` (`system`, `user`, `assistant`) and a `content` field.
   - Use `@mcp.prompt()` in FastMCP or prompt provider class wrappers.

2. **Design Modular, Parameterized Prompts**
   - **Role Separation:** Use `system` role messages for high-level rules and domain guardrails. Use `user` role messages for specific task execution templates.
   - **Parameterization:** Allow clients to supply custom arguments (e.g., `date`, `user_id`) with fallback default values.

3. **Implement Task & Standup Prompt Patterns**
   - Example pattern (custom provider class or FastMCP decorators):
     ```python
     from typing import Any, Optional
     from mcp.server.fastmcp import FastMCP

     mcp = FastMCP("task-prompt-provider")

     # Sample context (in production, fetch from PostgreSQL/MongoDB repositories)
     TASKS = {
         "1": {"title": "Deploy auth service", "status": "open", "assignedTo": "u1"},
         "2": {"title": "Fix SQL query leak", "status": "in_progress", "assignedTo": "u2"},
     }
     USERS = {
         "u1": {"name": "Alice"},
         "u2": {"name": "Bob"},
     }

     @mcp.prompt()
     def daily_summary() -> str:
         """Summarize open tasks into a system message."""
         lines = [f"• {t['title']} ({t['status']})" for t in TASKS.values()]
         return "Here is today's task summary:\n" + "\n".join(lines)

     @mcp.prompt()
     def standup_report(user_id: Optional[str] = None) -> str:
         """Build a standup report grouped by user or filtered for one user."""
         report_lines = []
         target_users = (
             {user_id: USERS[user_id]}
             if user_id and user_id in USERS
             else USERS
         )
         for uid, user in target_users.items():
             user_tasks = [
                 t["title"] for t in TASKS.values() if t.get("assignedTo") == uid
             ]
             summary = ", ".join(user_tasks) if user_tasks else "no tasks"
             report_lines.append(f"{user['name']}: {summary}")
         return "Stand up report:\n" + "\n".join(report_lines)
     ```

4. **Follow Prompt Authoring Guidelines**
   - **Provide Contextual Rationale:** Explain *why* certain parameter choices or tool chains are recommended, helping the LLM understand trade-offs rather than rigidly following a brittle script.
   - **Make Prompts Adaptive:** Dynamically adapt template suggestions based on user role, domain context, or incoming dataset characteristics (e.g. suggesting financial analysis strategies for financial documents vs debugging strategies for log files).
   - **Clarity over Ambiguity:** Use explicit language. Direct the model to call tools rather than guess facts (e.g., *"If the user asks for current metrics, call the `get_task_metrics` tool rather than guessing numbers."*).
   - **Conciseness:** Avoid bloated instructions that waste context window tokens.
   - **Versioning:** Include version identifiers in metadata if prompt semantics change over time.

---

## Decision points and guidance

- **System Prompt vs User Prompt?** Use `system` for persistent policy, capabilities summary, and tool usage rules. Use `user` for formatted task inputs, standups, or reporting templates.
- **Should dynamic data be baked into the prompt or fetched by tools?** Small context summaries (like today's open task counts) can be rendered directly into the prompt. Large datasets or active searches should be fetched by the model invoking a Tool.

---

## Quality criteria

- **Explicit Metadata:** Prompts have clear IDs, human-readable display names, and concise descriptions.
- **Clean Role Assignment:** Message roles are correctly marked (`system` for rules, `user` for execution context).
- **Tool Directives:** Prompts explicitly direct the agent to call available tools for domain actions.

---

## Example prompts

- "Create an MCP prompt provider that exposes a code review prompt template."
- "Build a parameterized standup report prompt in FastMCP that accepts user IDs."
- "Add system instructions to our FastMCP prompt that mandate tool usage over hallucinated API responses."

---

## Related guidance

- python-fastmcp-server-basics
- python-fastmcp-resource-providers
- python-lang-graph-context-engineering-strategies
