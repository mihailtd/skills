---
name: python-lang-graph-agentic-architectures
description: Guides teams to design advanced agentic systems with LangGraph and LangChain, covering multi-agent architectures, tool calling, reflection, ToT, handoff, and long-term memory.
---

# Python LangGraph Agentic Architectures

This skill helps AI explain how to build advanced agentic systems with LangGraph and LangChain. It covers architecture patterns for tools, task decomposition, multi-agent communication, self-reflection, Tree-of-Thoughts, handoff flows, streaming, human-in-the-loop, and long-term memory.

---

## When to use this skill

Use this skill when you need to:

- explain the architecture of high-performing agents,
- design multi-agent workflows and agent communication,
- build reflection and retry loops in LangGraph,
- implement Tree-of-Thoughts reasoning or LATS patterns,
- add human-in-the-loop and checkpoint-backed adaptation,
- integrate long-term memory and cache/store patterns.

---

## Outcome

Produce guidance that:

- defines agentic architecture patterns for LangGraph,
- shows tool-calling and task decomposition best practices,
- explains multi-agent collaboration and coordination,
- describes reflection, retry, and structured reasoning loops,
- provides code snippets for LangGraph nodes, agent orchestration, and fallback handling,
- discusses long-term memory, stores, and adaptive systems.

---

## Instructions for the AI

1. **Introduce agentic architecture patterns**
   - Explain that building agents is more than prompt engineering: it is a software architecture problem.
   - Emphasize modularity, configuration, and production-readiness.
   - Present key patterns: tool calling, task decomposition, cooperation/diversity, reflection/adaptation, and code-centric problem framing.

2. **Explain tool calling as a core pattern**
   - Describe tools as structured APIs the agent can call instead of producing free-form text.
   - Recommend clear tool descriptions, typed inputs, and explicit output formats.

```python
from langchain.agents import Tool

search_tool = Tool(
    name="web_search",
    func=lambda query: web_search(query),
    description="Search the web for current information relevant to the question.",
)
```

3. **Show task decomposition and planning**
   - Explain splitting complex tasks into smaller steps with explicit planning.
   - Show a LangGraph node that emits a plan before execution.

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class ExamState(TypedDict):
    question: str
    plan: list[str]
    answer: str


def plan_question(state: ExamState):
    prompt = f"Create a step-by-step plan to answer: {state['question']}"
    plan = llm.invoke(prompt)
    return {"plan": plan.split("\n")}

builder = StateGraph(ExamState)
builder.add_node("plan_question", plan_question)
# ... add subsequent solving nodes ...
```

4. **Describe multi-agent architectures**
   - Explain using multiple agents with different roles, tools, or system prompts.
   - Mention natural language as a shared medium for agent communication.

```python
agent_a_prompt = "You are a fact-checker agent."
agent_b_prompt = "You are a reasoning agent that chains thoughts."
```

5. **Describe self-reflection and adaptation**
   - Show a node that evaluates previous reasoning and revises the answer.

```python

def reflect_and_improve(state: ExamState):
    critique = llm.invoke(
        f"Review this answer: {state['answer']} and improve it."
    )
    return {"answer": critique}
```

6. **Explain handoff and advanced control flow**
   - Describe how LangGraph supports handoff between agents or tools.
   - Mention streaming control flows and external signals for advanced orchestration.

7. **Introduce Tree-of-Thoughts and reasoning paths**
   - Explain that ToT expands multiple reasoning candidates instead of one answer.
   - Mention advanced trimming/pruning to keep search tractable.

```python
thoughts = ["Consider definition first.", "Check edge cases."]
for thought in thoughts:
    result = llm.invoke(f"Next reasoning step: {thought}")
```

8. **Show long-term memory and stores**
   - Explain memory as caches or persistent stores used across runs.
   - Mention LangChain stores, vector databases, and custom caches.

9. **Discuss adaptive systems and human-in-the-loop**
   - Explain how checkpoints and human feedback can pause and resume workflows.
   - Recommend using thread IDs, checkpoints, and external approval steps.

10. **Warn about non-determinism and experimentation**
    - Advise against trusting a single output.
    - Encourage evaluating multiple candidates and experimenting with patterns.
    - Emphasize measurement and iteration.

---

## Decision points and guidance

- **Is the task complex?** Decompose it and use tool-calling instead of one giant prompt.
- **Do you need better reliability?** Add reflection, retry, and structured parsers.
- **Should agents collaborate?** Use multiple agent roles and shared messages.
- **Is the system production-bound?** Make workflows modular, configurable, and checkpointable.
- **Do you need memory across runs?** Add stores or caches and persist state.

---

## Quality criteria

A strong response should ensure the guidance is:

- **architectural:** treats agents as composable software components,
- **modular:** separates tools, planning, reasoning, and execution,
- **adaptive:** includes reflection, retries, and fallbacks,
- **multi-agent:** encourages diversity and cooperation,
- **evaluated:** recommends experimentation and measurement.

---

## Example prompts

- "How do I design a multi-agent system with LangGraph?"
- "What patterns should I use to build a reasoning agent with tools?"
- "How can I add reflection and retries to a LangGraph agent?"
- "What is the Tree-of-Thoughts pattern and how do I use it?"

---

## Related guidance

Use this tool alongside:

- python-lang-graph-workflows
- python-lang-graph-error-handling
- python-lang-graph-output-parsing
- python-lang-graph-checkpoints
