---
name: python-lang-graph-agent-memory
description: Guides teams to design LangGraph agent memory patterns, including per-thread checkpoints, long-term stores, LLM caching, retrieval, and compaction.
---

# Python LangGraph Agent Memory

This skill explains how to design memory for LangGraph agents. It covers short-term memory through checkpoints, long-term memory with stores and caches, retrieval for context-aware execution, and practical patterns for persistence, compaction, and privacy-aware memory management.

---

## When to use this skill

Use this skill when you need to:

- explain how agent memory differs from prompt context,
- preserve state across LangGraph runs with checkpoints,
- implement long-term knowledge retention beyond a single conversation,
- cache LLM outputs and reuse previously generated answers,
- build retrieval-driven memory for dynamic agent adaptation,
- manage memory lifecycle, compaction, and privacy.

---

## Outcome

Produce guidance that:

- distinguishes per-thread checkpoint persistence from long-term memory,
- shows code snippets for LangChain caches and LangGraph stores,
- explains how to extract and retrieve relevant memory,
- describes how to compact memory over time,
- encourages using a custom store when you need advanced read/write patterns,
- highlights security and privacy considerations.

---

## Instructions for the AI

1. **Explain the three memory layers**
   - Short-term memory: LangGraph checkpoints store per-thread state and execution progress.
   - Long-term memory: persistent stores keep facts, user history, and reusable knowledge across sessions.
   - Response caching: caches save LLM outputs to avoid recomputing repeated prompts.

2. **Describe per-thread persistence with checkpoints**
   - Explain that a LangGraph thread is similar to a conversation.
   - Show how `checkpointer` preserves state so the agent resumes where it left off.

```python
from langgraph.graph import StateGraph, START
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import Command

class SessionState(TypedDict):
    user_id: str
    question: str
    answer: Optional[str]

builder = StateGraph(SessionState)

# ... add nodes and edges ...

checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "conversation-123"}}
response = graph.invoke({"user_id": "user1", "question": "What is the weather?"}, config=config)
```

3. **Explain why checkpoints are not long-term memory**
   - Checkpoints remember the current session state and execution flow.
   - They are efficient for resume and HIL workflows, but they do not automatically store reusable knowledge across unrelated sessions.

4. **Show LLM response caching**
   - Explain the global LLM cache in LangChain.
   - Show how prompt text and model settings become part of the cache key.

```python
from langchain_core.caches import InMemoryCache
from langchain_core.globals import set_llm_cache
from langchain.chat_models import ChatOpenAI

cache = InMemoryCache()
set_llm_cache(cache)
llm = ChatOpenAI(temperature=0.5)

result = llm.invoke("What is the capital of the UK?")
```

5. **Explain built-in store patterns**
   - Describe `BaseStore` as a persistent key-value store with namespaces.
   - Mention built-in implementations: `InMemoryStore` and `PostgresStore`.

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()
store.put(namespace=("users", "user1"), key="name", value={"text": "John"})
item = store.get(namespace=("users", "user1"), key="name")
print(item.value)
```

6. **Show retrieval using namespaces and search**
   - Explain partial namespace search and semantic query support.
   - Show how to retrieve memory relevant to a user or a task.

```python
facts = store.search(("users", "user1"), query="address")
for fact in facts:
    print(fact.key, fact.value)
```

7. **Show a LangGraph memory-aware node**
   - Demonstrate a node that fetches context from a store and stores new memories.
   - Emphasize that memory retrieval belongs in the workflow state, not hidden inside prompt engineering.

```python
from langgraph.types import Command

class MemoryState(TypedDict):
    user_id: str
    task: str
    memory_context: Optional[str]
    action: Optional[str]


def retrieve_memory(state: MemoryState):
    items = store.search(("users", state["user_id"]), query=state["task"])
    memory_context = "\n".join(item.value["text"] for item in items)
    return {"memory_context": memory_context}


def persist_memory(state: MemoryState):
    if state.get("action"):
        key = f"memory-{uuid4()}"
        store.put(
            namespace=("users", state["user_id"]),
            key=key,
            value={"text": state["action"], "task": state["task"]},
        )
    return {}
```

8. **Present a custom cache/store node pattern**
   - Explain when to use Redis, MongoDB, or another database.
   - Show a simple custom cache node using Redis.

```python
import redis
from langchain_core.schema import BaseCache

class RedisCache(BaseCache):
    def __init__(self, redis_client):
        self.redis = redis_client

    def lookup(self, key: str):
        return self.redis.get(key)

    def update(self, key: str, value: str):
        self.redis.set(key, value)

redis_client = redis.Redis(host="localhost", port=6379, db=0)
llm_cache = RedisCache(redis_client)
set_llm_cache(llm_cache)
```

9. **Show memory compaction and summarization**
   - Explain periodic summarization and forgetting.
   - Show a memory compaction node that turns many facts into a shorter summary.

```python
from langchain_core.prompts import PromptTemplate

summary_prompt = PromptTemplate.from_template(
    "Summarize the following user facts into a short profile:\n\n{facts}"
)


def compact_memory(state: MemoryState):
    items = store.search(("users", state["user_id"]), query=state["task"])
    if not items:
        return {}

    facts = "\n".join(item.value["text"] for item in items)
    summary = llm.invoke(summary_prompt.format(facts=facts))
    store.put(
        namespace=("users", state["user_id"]),
        key="profile_summary",
        value={"text": summary},
    )
    return {"memory_context": summary}
```

10. **Highlight design tradeoffs and guardrails**
    - Recommend extracting only useful, non-sensitive facts.
    - Advise against caching personally identifiable information unless explicitly consented.
    - Recommend using namespace hierarchies like `("users", user_id, "session")` and `("users", user_id, "profile")`.
    - Recommend pruning old memories and using a refresh/compaction policy.

---

## Decision points and guidance

- **Need session resume?** Use LangGraph checkpoints and thread IDs.
- **Need reuse across sessions?** Use a long-term store, not just checkpoints.
- **Need repeated prompt answers?** Use an LLM cache.
- **Need semantic retrieval?** Use a store with search over embeddings.
- **Need distributed memory across workers?** Use a custom external store (Redis, Postgres, MongoDB).
- **Need privacy controls?** Separate sensitive data into dedicated namespaces and avoid caching raw user-provided secrets.

---

## Quality criteria

A strong response should ensure the guidance is:

- **clear:** separates checkpoint memory, caching, and store-based memory,
- **practical:** includes runnable code snippets,
- **retrieval-aware:** explains how to search and reuse stored memory,
- **adaptive:** encourages using memory to improve future runs,
- **secure:** calls out privacy, namespace hygiene, and pruning.

---

## Example prompts

- "How do I add long-term memory to a LangGraph agent?"
- "What is the difference between LangGraph checkpoints and a memory store?"
- "How can I retrieve user memory for a follow-up task?"
- "How should I summarize and compact agent memory?"

---

## Related guidance

Use this tool alongside:

- python-lang-graph-checkpoints
- python-lang-graph-state-management
- python-lang-graph-agentic-architectures
- python-lang-graph-output-parsing
