---
name: python-lang-graph-checkpoints
description: Guides teams to implement LangGraph checkpoints in Python for persistence, debugging, human-in-the-loop workflows, and fault-tolerant execution.
---

# Python LangGraph Checkpoints

This skill helps AI explain how to use LangGraph checkpoints to snapshot workflow state, restore execution, and build resilient stateful workflows. It includes code samples for local memory checkpointing and database-backed checkpoint persistence.

---

## When to use this skill

Use this skill when you need to:

- explain the purpose of LangGraph checkpoints,
- persist workflow state across executions,
- enable debugging, time travel, and branching experiments,
- implement human-in-the-loop workflows,
- build production-ready fault-tolerant LangGraph systems.

---

## Outcome

Produce guidance that:

- defines LangGraph checkpoints as a snapshot of graph state, pending tasks, and metadata,
- shows how to compile a graph with a checkpoint saver,
- demonstrates invoking graphs with thread IDs and checkpoint IDs,
- explains how to list and restore checkpoints,
- compares in-memory checkpointing with SQLite/Postgres persistence.

---

## Instructions for the AI

1. **Explain what a checkpoint is**
   - Define a checkpoint as a snapshot of graph execution state, including full state, pending nodes, and failed tasks.
   - Explain how checkpoints differ from chat history by allowing the workflow to resume from any point in time.
   - Emphasize benefits: debugging, experimentation, human intervention, and fault tolerance.

2. **Show a simple checkpoint example**
   - Use `MessageGraph` and `MemorySaver`.

```python
from langgraph.graph import MessageGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_core.messages import AIMessage, HumanMessage


def test_node(state):
    print(f"History length = {len(state[:-1])}")
    return [AIMessage(content="Hello!")]

builder = MessageGraph()
builder.add_node("test_node", test_node)
builder.add_edge(START, "test_node")
builder.add_edge("test_node", END)

memory = MemorySaver()
graph = builder.compile(checkpointer=memory)
```

3. **Explain thread IDs and checkpoint invocation**
   - Show how each run uses a unique `thread_id`.
   - Demonstrate separate execution contexts for independent runs.

```python
_ = graph.invoke(
    [HumanMessage(content="test")],
    config={"configurable": {"thread_id": "thread-a"}},
)
_ = graph.invoke(
    [HumanMessage(content="test")],
    config={"configurable": {"thread_id": "thread-b"}},
)
_ = graph.invoke(
    [HumanMessage(content="test")],
    config={"configurable": {"thread_id": "thread-a"}},
)
```

4. **Show checkpoint inspection**
   - Use `memory.list()` to enumerate checkpoints for a thread.

```python
checkpoints = list(memory.list(config={"configurable": {"thread_id": "thread-a"}}))
for checkpoint in checkpoints:
    print(checkpoint.config["configurable"]["checkpoint_id"])
```

5. **Show checkpoint restoration**
   - Restore from an initial or intermediate checkpoint.

```python
checkpoint_id = checkpoints[-1].config["configurable"]["checkpoint_id"]
_ = graph.invoke(
    [HumanMessage(content="test")],
    config={
        "configurable": {
            "thread_id": "thread-a",
            "checkpoint_id": checkpoint_id,
        }
    },
)
```

6. **Explain human-in-the-loop and stateless services**
   - Describe how checkpoints let a stateless web service resume stateful graphs across requests.
   - Explain that local memory checkpointing is fine for demos, but production needs durable storage.

7. **Introduce persistent checkpoint storage**
   - Mention `SqliteSaver` and `PostgresSaver`.
   - Explain that persistence stores checkpoint dictionaries externally so any instance can restore them.

```python
from langgraph.checkpoint.sqlite import SqliteSaver

saver = SqliteSaver(database_file="./checkpoints.db")
graph = builder.compile(checkpointer=saver)
```

8. **Explain custom checkpoint storage**
   - State that any checkpoint store can be implemented by storing and retrieving dictionaries representing checkpoints.
   - Mention that custom savers can target Redis, MongoDB, or another database.

9. **Discuss practical use cases**
   - Deep debugging and time travel through checkpoint history.
   - Running experiments from different decision points without rerunning upstream work.
   - Human intervention workflows where a user action resumes the graph later.
   - Fault-tolerant production systems that resume after crashes or retries.

10. **Warn about checkpoint lifecycle**
    - Recommend pruning or expiring old checkpoints.
    - Note that checkpoints may consume storage and should be managed.
    - Mention that checkpoint data may contain sensitive state and should be protected accordingly.

---

## Decision points and guidance

- **Do you need fast experimentation or debugging?** Use checkpoints to time travel and resume workflows.
- **Do you need multi-instance production workflows?** Persist checkpoints to SQLite, Postgres, or a shared database.
- **Is your workflow human-in-the-loop?** Use checkpoints to pause and resume execution.
- **Are you running a demo or prototype?** `MemorySaver` is fine.
- **Do you need durability and resilience?** Use a persistent saver and protect checkpoint storage.

---

## Quality criteria

A strong response should ensure the guidance is:

- **checkpoint-aware:** explains snapshot semantics and use cases,
- **practical:** includes code examples for invoking and restoring checkpoints,
- **resilient:** highlights persistence for production workflows,
- **interactive:** covers human-in-the-loop and multi-instance scenarios,
- **secure:** reminds users to manage checkpoint lifecycle and sensitive state.

---

## Example prompts

- "How do LangGraph checkpoints work in Python?"
- "How can I restore LangGraph execution from a checkpoint?"
- "What is the difference between local and persistent LangGraph checkpoints?"
- "How do checkpoints help build human-in-the-loop workflows?"

---

## Related guidance

Use this tool alongside:

- python-lang-graph-workflows
- python-lang-graph-state-management
- python-lang-graph-error-handling
