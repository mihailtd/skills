---
name: python-lang-graph-workflows
description: Guides teams to build LangGraph workflows in Python, explaining graph fundamentals, stateful execution, conditional edges, and reusable workflow patterns.
---

# Python LangGraph Workflow Design

This skill helps AI explain how to build LangGraph workflows with Python. It covers LangGraph fundamentals, graph construction, state-aware nodes, conditional branching, supersteps, and visualization.

---

## When to use this skill

Use this skill when you need to:

- explain LangGraph as a workflow orchestration framework,
- define Python-based LangGraph nodes and state schemas,
- chain multiple steps and conditionally branch execution,
- teach LangGraph graph construction and visualization,
- compare LangGraph to simpler sequential LangChain flows.

---

## Outcome

Produce guidance that:

- defines LangGraph workflows as directed graphs with nodes and edges,
- shows how to build a workflow with `StateGraph`, `START`, and `END`,
- explains immutable state updates and how nodes return partial state,
- demonstrates conditional edges and branching logic,
- includes code examples for building and invoking a graph.

---

## Instructions for the AI

1. **Explain LangGraph fundamentals**
   - Describe LangGraph as a graph-based orchestration framework for generative AI workflows.
   - Explain that a graph consists of nodes (tasks) and edges (dependencies), and that LangGraph supports cycles, streaming, and complex control flow.
   - Contrast DAG-style workflows with LangGraph’s richer graph semantics.

2. **Show state schema definition**
   - Explain that LangGraph state is often a typed Python dictionary, but Pydantic models or dataclasses also work.
   - Provide a `TypedDict` example.

```python
from typing_extensions import TypedDict

class JobApplicationState(TypedDict):
    job_description: str
    is_suitable: bool
    application: str
```

3. **Show a simple graph build example**
   - Use `StateGraph`, add nodes, and connect edges.

```python
from langgraph.graph import StateGraph, START, END


def analyze_job_description(state):
    print("...Analyzing a provided job description ...")
    return {"is_suitable": len(state["job_description"]) > 100}


def generate_application(state):
    print("...generating application...")
    return {"application": "some_fake_application"}

builder = StateGraph(JobApplicationState)
builder.add_node("analyze_job_description", analyze_job_description)
builder.add_node("generate_application", generate_application)
builder.add_edge(START, "analyze_job_description")
builder.add_edge("analyze_job_description", "generate_application")
builder.add_edge("generate_application", END)
graph = builder.compile()

res = graph.invoke({"job_description": "fake_jd"})
print(res)
```

4. **Explain state handling**
   - Stress that nodes receive an immutable copy of state.
   - Explain that nodes return only updated keys, and LangGraph merges them into the master state.
   - Note that this design avoids side effects and makes workflows traceable.

5. **Show workflow visualization**
   - Demonstrate Mermaid visualization with `graph.get_graph().draw_mermaid_png()`.
   - Explain that visualization helps inspect graph structure and conditional flow.

6. **Introduce conditional edges**
   - Explain `add_conditional_edges` and how state determines the next path.
   - Show a branching example using `Literal` for type safety.

```python
from typing import Literal


def is_suitable_condition(state: JobApplicationState) -> Literal["generate_application", END]:
    return "generate_application" if state.get("is_suitable") else END

builder.add_conditional_edges("analyze_job_description", is_suitable_condition)
```

7. **Explain execution semantics**
   - Describe LangGraph supersteps as iterations over nodes and merged state updates.
   - Mention that LangGraph can execute multiple nodes concurrently and supports cycles with controlled recursion.

8. **Mention reusable workflows**
   - Encourage separating node logic from graph topology.
   - Suggest parameterizing graphs for different behaviors and reusing nodes across graphs.

---

## Decision points and guidance

- **Is the workflow sequential or complex?** Use LangGraph for conditional and parallel graph execution.
- **Should node behavior depend on shared context?** Define a typed state schema and state-aware nodes.
- **Do you need branching?** Use `add_conditional_edges` with `Literal` or mapping.
- **Do you want observable workflows?** Use built-in graph visualization.

---

## Quality criteria

A strong response should ensure the guidance is:

- **graph-aware:** explains nodes, edges, and state clearly,
- **typed:** encourages typed state schemas or models,
- **modular:** separates node logic from graph topology,
- **conditional:** shows branching examples,
- **observable:** uses visualization and explicit invocation.

---

## Example prompts

- "How do I build a LangGraph workflow in Python?"
- "What is the difference between a LangGraph graph and a LangChain chain?"
- "How can I add conditional branches to a LangGraph workflow?"
- "How does LangGraph manage state across nodes?"

---

## Related guidance

Use this tool alongside:

- python-lang-graph-state-management
- python-lang-graph-output-parsing
- python-lang-graph-error-handling
