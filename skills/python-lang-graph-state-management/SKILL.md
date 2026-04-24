---
name: python-lang-graph-state-management
description: Guides teams to manage LangGraph workflow state in Python, including TypedDict schemas, reducers, message history, and configurable state updates.
---

# Python LangGraph State Management

This skill helps AI explain how to design and manage workflow state in LangGraph. It covers typed state schemas, reducers, message accumulation, state immutability, and configurable graphs.

---

## When to use this skill

Use this skill when you need to:

- explain LangGraph state concepts and schema design,
- teach immutable state updates and reducer behavior,
- accumulate messages or custom actions across supersteps,
- make graphs configurable with runtime settings,
- leverage built-in message state for conversational workflows.

---

## Outcome

Produce guidance that:

- shows how to define state schemas with `TypedDict`, dataclasses, or Pydantic,
- explains how nodes update state safely with partial dictionaries,
- demonstrates built-in reducers and custom reducer functions,
- shows how to manage conversational message history,
- describes configurable graphs using `RunnableConfig`.

---

## Instructions for the AI

1. **Explain LangGraph state concepts**
   - Define state as workflow context passed to each node.
   - Emphasize the state is conceptually a Python dictionary and nodes return updates, not direct mutations.
   - Note that state persistence enables interactive and multi-step workflows.

2. **Show typed state schema examples**
   - Use `TypedDict` for clear state definitions.

```python
from typing_extensions import TypedDict

class JobApplicationState(TypedDict):
    job_description: str
    is_suitable: bool
    application: str
```

3. **Explain default reducer behavior**
   - Describe that LangGraph replaces the current key value with the node’s output by default.
   - Give an example where a node returns `{"actions": ["action1"]}` and later node returns `{"actions": ["action2"]}`.

4. **Show additive reducer usage**
   - Explain `Annotated[list[str], add]` for concatenating list values.

```python
from typing import Annotated
from operator import add

class JobApplicationState(TypedDict):
    job_description: str
    is_suitable: bool
    application: str
    actions: Annotated[list[str], add]
```

5. **Show custom reducer example**
   - Demonstrate a custom reducer that accepts strings or lists.

```python
from typing import Annotated, Optional, Union

def my_reducer(left: list[str], right: Optional[Union[str, list[str]]]) -> list[str]:
    if right:
        return left + [right] if isinstance(right, str) else left + right
    return left

class JobApplicationState(TypedDict):
    job_description: str
    is_suitable: bool
    application: str
    actions: Annotated[list[str], my_reducer]
```

6. **Explain message state and conversations**
   - Introduce `MessagesState` and `add_messages` reducer for LLM messages.

```python
from langgraph.graph import MessagesState
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages

class JobApplicationState(MessagesState):
    job_description: str
    is_suitable: bool
    application: str
```

7. **Show configurable graphs with RunnableConfig**
   - Explain how nodes can accept `config: RunnableConfig`.

```python
from langchain_core.runnables.config import RunnableConfig

def generate_application(state: JobApplicationState, config: RunnableConfig):
    provider = config["configurable"].get("model_provider", "Google")
    model_name = config["configurable"].get("model_name", "gemini-1.5-flash-002")
    print(f"...generating application with {provider} and {model_name} ...")
    return {"application": "some_fake_application", "actions": ["action2", "action3"]}
```

8. **Show graph invocation with custom config**

```python
res = graph.invoke(
    {"job_description": "fake_jd"},
    config={"configurable": {"model_provider": "OpenAI", "model_name": "gpt-4o"}},
)
print(res)
```

9. **Explain state immutability and merge semantics**
   - Emphasize that nodes receive immutable state copies and return only updates.
   - Note that the graph merges updates at the end of each superstep.

---

## Decision points and guidance

- **Is the workflow state simple?** Use `TypedDict` for clarity.
- **Do you need message accumulation?** Use `MessagesState` and `add_messages`.
- **Do you need additive state updates?** Use `Annotated[<type>, add]` or a custom reducer.
- **Should nodes vary by configuration?** Accept `RunnableConfig` and make graphs configurable.
- **Do you want explicit state changes?** Keep nodes returning only updated keys.

---

## Quality criteria

A strong response should ensure the guidance is:

- **typed:** clearly illustrates state schemas,
- **safe:** explains immutable node inputs and explicit updates,
- **extendable:** encourages reducers for aggregated state,
- **configurable:** shows runtime graph configuration,
- **conversational:** addresses message history for LLM workflows.

---

## Example prompts

- "How do I define state for a LangGraph workflow in Python?"
- "What is the difference between default and custom reducers in LangGraph?"
- "How can I accumulate chat messages in LangGraph state?"
- "How do I pass configuration into a LangGraph node?"

---

## Related guidance

Use this tool alongside:

- python-lang-graph-workflows
- python-lang-graph-output-parsing
- python-lang-graph-error-handling
