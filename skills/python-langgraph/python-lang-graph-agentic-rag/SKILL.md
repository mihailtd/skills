---
name: python-lang-graph-agentic-rag
description: Guides teams to build agentic RAG workflows with LangGraph and LangChain, including dynamic retrieval, query expansion, chunk reasoning, reflection, and iterative adaptation.
---

# Python LangGraph Agentic RAG

This skill helps AI explain how to build Retrieval-Augmented Generation systems that share control with an LLM. It covers agentic RAG design patterns, dynamic retrieval tools, chunk-level reasoning, query expansion, reflection loops, and LangGraph orchestration for advanced RAG applications.

---

## When to use this skill

Use this skill when you need to:

- explain how RAG evolves into agentic RAG,
- design retrieval workflows controlled by an LLM,
- incorporate query expansion and chunk-level reasoning,
- build reflective and adaptive retrieval pipelines,
- compare traditional RAG to LangGraph-based agentic RAG.

---

## Outcome

Produce guidance that:

- defines agentic RAG as retrieval + LLM control over workflow,
- shows tool-based retrieval as part of LangGraph workflows,
- explains query expansion, chunk filtering, and hybrid retrieval,
- includes examples of chunk reasoning and fusion,
- demonstrates reflection and iteration for grounding and answer polishing.

---

## Instructions for the AI

1. **Explain agentic RAG conceptually**
   - Define classic RAG: retrieve, combine context, ask LLM.
   - Explain agentic RAG as handing partial flow control to the LLM for retrieval, filtering, and adaptation.
   - Emphasize that agentic RAG may use LangGraph when a workflow needs dynamic branching and stateful orchestration.

2. **Describe dynamic retrieval as a tool**
   - Explain that the LLM can decide whether to use embeddings, keyword search, hybrid search, or web search.
   - Show retrieval wrapped as a tool with a descriptive interface.

```python
from langchain.agents import Tool

search_tool = Tool(
    name="document_search",
    func=lambda query: vector_search(query),
    description="Search documents using the best strategy for the provided query.",
)
```

3. **Show query expansion**
   - Explain generating multiple search queries from the initial question.
   - Use a small LangGraph node or chain to expand queries.

```python

def expand_queries(state):
    prompt = (
        f"Given the user question, generate three related search queries:\n" \
        f"Question: {state['question']}\n"
    )
    result = llm.invoke(prompt)
    return {"queries": result.split("\n")}
```

4. **Explain chunk-level reasoning**
   - Describe evaluating or summarizing each retrieved chunk before final composing.
   - Show a node that scores or filters chunks with the LLM.

```python

def evaluate_chunk(state):
    chunk = state['chunk']
    prompt = (
        "Does this chunk help answer the question? Answer YES or NO and summarize the relevant part.\n"
        f"Question: {state['question']}\nChunk: {chunk}"
    )
    result = llm.invoke(prompt)
    return {"chunk_assessment": result}
```

5. **Show fusion of multiple chunk outputs**
   - Explain reducing smaller reasoning results into a final answer.
   - Mention reciprocal fusion or weighted aggregation.

```python

def fuse_results(state):
    summaries = state['chunk_summaries']
    prompt = (
        "Combine these chunk summaries into a single answer to the question.\n"
        f"Question: {state['question']}\nSummaries:\n{summaries}"
    )
    answer = llm.invoke(prompt)
    return {"answer": answer}
```

6. **Explain reflection and iteration**
   - Show a node that evaluates the answer and decides whether to continue.

```python

def reflect(state):
    prompt = (
        "Review the current answer and determine whether additional retrieval or refinement is needed."
        f"\nQuestion: {state['question']}\nAnswer: {state['answer']}"
    )
    decision = llm.invoke(prompt)
    return {"continue_search": "yes" in decision.lower()}
```
```

7. **Describe LangGraph orchestration**
   - Explain that LangGraph manages state, branching, and retries across retrieval and reasoning nodes.
   - Mention `StateGraph` can execute retrieval, evaluation, fusion, and reflection steps in sequence or with conditional edges.

8. **Mention practical tradeoffs**
   - Warn that chunk reasoning and query expansion improve quality but increase cost and latency.
   - Recommend balancing retrieval threshold, chunk count, and iteration depth.

9. **Discuss when to use LangGraph vs LangChain**
   - Explain that LangChain is fine for simpler pipelines.
   - Recommend LangGraph when the workflow needs dynamic branching, stateful intermediate steps, or checkpointing.

10. **Cover grounding and attribution**
    - Suggest using grounding steps to verify sources and add citations.
    - Explain that an extra tool can call a grounded attribution module or web search.

---

## Decision points and guidance

- **Is the retrieval strategy static?** Classic RAG is sufficient.
- **Does the LLM need to choose retrieval methods?** Use agentic retrieval tools.
- **Are retrieved chunks noisy?** Add chunk-level filtering or summarization.
- **Should the workflow adapt during execution?** Use reflection and iterative refinement.
- **Is the task complex or open-ended?** LangGraph offers better orchestration for agentic RAG.

---

## Quality criteria

A strong response should ensure the guidance is:

- **agentic:** shows how the LLM controls retrieval and workflow decisions,
- **modular:** separates retrieval, expansion, chunk evaluation, and synthesis,
- **reflective:** encourages dynamic iteration and answer validation,
- **practical:** includes tool and node examples for real systems,
- **balanced:** notes cost/latency tradeoffs of advanced RAG steps.

---

## Example prompts

- "How do I build an agentic RAG workflow with LangGraph?"
- "When should I use query expansion and chunk reasoning in RAG?"
- "What does dynamic retrieval mean in an agentic RAG system?"
- "How can I add reflection to my LangGraph RAG pipeline?"

---

## Related guidance

Use this tool alongside:

- python-lang-graph-workflows
- python-lang-graph-output-parsing
- python-lang-graph-checkpoints
- python-lang-graph-agentic-architectures
