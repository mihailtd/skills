# Python — LangGraph

LangGraph agentic-architecture skills: multi-agent design, state management, checkpoints, memory, output parsing, and agentic RAG workflows.

For projects building LLM agents with LangGraph.

## Install

```bash
npx skills add mihailtd/skills/skills/python-langgraph --all
```

Add `-g` to install globally instead of per-project. Use `--skill <name>` (repeatable) instead of `--all` to cherry-pick individual skills from this package, e.g.:

```bash
npx skills add mihailtd/skills/skills/python-langgraph --skill python-lang-graph-agent-memory
```

## Skills (13)

- **[python-lang-graph-agent-memory](python-lang-graph-agent-memory/SKILL.md)** — Guides teams to design LangGraph agent memory patterns, including per-thread checkpoints, long-term stores, LLM caching, retrieval, and compaction.
- **[python-lang-graph-agentic-architectures](python-lang-graph-agentic-architectures/SKILL.md)** — Guides teams to design advanced agentic systems with LangGraph and LangChain, covering multi-agent architectures, tool calling, reflection, ToT, handoff, and long-term memory.
- **[python-lang-graph-workflow-vs-agent-patterns](python-lang-graph-workflow-vs-agent-patterns/SKILL.md)** — Choosing the right level of LLM autonomy for a given step: single LLM call, chain, or router (workflow patterns) vs. tool-use loop or tool creation (agent patterns) — and combining them by embedding an agent as a bounded node inside a larger predictable workflow.
- **[python-lang-graph-agent-suitability](python-lang-graph-agent-suitability/SKILL.md)** — The two-stage decision that should precede any agent build: does this task need an LLM at all (unstructured data, input diversity), and if so does it need an agent or will a workflow suffice (task complexity, value vs. cost/latency, error cost and detectability).
- **[python-lang-graph-context-engineering-fundamentals](python-lang-graph-context-engineering-fundamentals/SKILL.md)** — Diagnosing agent failures correctly (insufficient model intelligence vs. the far more common missing-information cause), the shift from prompt engineering to context engineering, and why bigger context windows aren't automatically better (Context Rot, Lost in the Middle).
- **[python-lang-graph-context-engineering-strategies](python-lang-graph-context-engineering-strategies/SKILL.md)** — Designing an agent's context pipeline using the five context engineering strategies: Generation, Retrieval, Write, Reduce, and Isolate — with memory design as the point where Retrieval, Write, and Reduce converge.
- **[python-lang-graph-agentic-rag](python-lang-graph-agentic-rag/SKILL.md)** — Guides teams to build agentic RAG workflows with LangGraph and LangChain, including dynamic retrieval, query expansion, chunk reasoning, reflection, and iterative adaptation.
- **[python-lang-graph-checkpoints](python-lang-graph-checkpoints/SKILL.md)** — Guides teams to implement LangGraph checkpoints in Python for persistence, debugging, human-in-the-loop workflows, and fault-tolerant execution.
- **[python-lang-graph-error-handling](python-lang-graph-error-handling/SKILL.md)** — Guides teams to implement robust error handling and retries in Python LangGraph workflows, including exception capture, fallback chains, and parser recovery.
- **[python-lang-graph-multi-agent-architectures](python-lang-graph-multi-agent-architectures/SKILL.md)** — Guides teams to design and implement multi-agent architectures with LangGraph, including specialization, subgraphs, consensus, semantic routing, and agent communication patterns.
- **[python-lang-graph-output-parsing](python-lang-graph-output-parsing/SKILL.md)** — Guides teams to parse and validate LLM outputs in Python LangGraph workflows, including prompt engineering, output parsers, and controlled generation patterns.
- **[python-lang-graph-state-management](python-lang-graph-state-management/SKILL.md)** — Guides teams to manage LangGraph workflow state in Python, including TypedDict schemas, reducers, message history, and configurable state updates.
- **[python-lang-graph-workflows](python-lang-graph-workflows/SKILL.md)** — Guides teams to build LangGraph workflows in Python, explaining graph fundamentals, stateful execution, conditional edges, and reusable workflow patterns.
