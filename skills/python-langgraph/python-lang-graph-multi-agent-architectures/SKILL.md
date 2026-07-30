---
name: python-lang-graph-multi-agent-architectures
description: Guides teams to design and implement multi-agent architectures with LangGraph, including specialization, subgraphs, consensus, semantic routing, and agent communication patterns.
---

# Python LangGraph Multi-Agent Architectures

This skill helps AI explain how to build multi-agent systems with LangGraph. It covers specialization, subgraph composition, consensus mechanisms, semantic routing, communication protocols, and advanced multi-agent orchestration patterns.

---

## When to use this skill

Use this skill when you need to:

- explain why multi-agent architectures improve complex task performance,
- design specialized agents with limited toolkits,
- compose child graphs into larger workflows,
- implement consensus mechanisms to choose the best answer,
- route tasks semantically between specialized agents,
- organize agent communication with structured protocols or message buses.

---

## Outcome

Produce guidance that:

- defines agent roles and specialization,
- shows how to keep agents modular and configurable,
- demonstrates LangGraph subgraph integration patterns,
- explains consensus strategies and voting mechanisms,
- provides semantic routing architectures,
- describes structured and message-based inter-agent communication.

---

## Instructions for the AI

1. **Explain specialization and modularity**
   - Emphasize that specialized agents are more effective on narrow tasks.
   - Highlight benefits: tailored prompts, smaller tool sets, and easier testing.
   - Recommend limiting each agent to 5–15 tools and grouping them into coherent toolkits.

2. **Discuss agent composition patterns**
   - Explain how LangGraph allows subgraphs to be used as nodes in larger graphs.
   - Show two composition styles: direct subgraph nodes and wrapper functions.

```python
builder.add_node("pay", payments_agent)
```

```python

def _run_payment(state):
    result = payments_agent.invoke({"client_id": state["client_id"]})
    return {"payment_status": result["status"]}

builder.add_node("pay", _run_payment)
```

3. **Show subgraph schema considerations**
   - Explain that child and parent agents may have different schemas.
   - Note that direct subgraph invocation uses matching keys, while wrapper functions give full control over state mapping.

4. **Explain consensus mechanisms**
   - Describe parallel agent execution followed by consensus.
   - Show simple majority voting for classification.

```python

def majority_vote(state):
    answers = state["agent_answers"]
    best = max(set(answers), key=answers.count)
    return {"final_answer": best}
```

5. **Describe alternative consensus strategies**
   - Score solutions on 0–1 scale and choose max,
   - use an external oracle or verifier,
   - use a judge LLM with few-shot scoring,
   - build a dedicated selection agent.

6. **Show semantic routing**
   - Explain routing tasks to agents based on question classification.
   - Show a router node that chooses the appropriate agent.

```python

def route_question(state):
    prompt = (
        "Classify this question into general, company, or user-specific data.\n"
        f"Question: {state['question']}"
    )
    category = router_llm.invoke(prompt)
    return {"route": category}
```

builder.add_conditional_edges(
    "route_question",
    lambda state: state["route"],
    {"general": "general_agent", "company": "company_agent", "user": "user_agent"},
)
```

7. **Explain communication protocols**
   - Compare structured communication with controlled output vs. shared message lists.
   - Show a structured output approach using Pydantic or typed messages.
   - Mention that message-based communication is natural but requires trimming and careful provider support.

8. **Show a shared message pattern**
   - Explain using a common message history for agent interactions.
   - Warn about growing context and memory trimming.

9. **Discuss reflection and critique workflows**
   - Explain self-reflection, cross-reflection, and supervisor agents.
   - Show a reflection node that criticizes another agent’s answer.

```python

def critique_answer(state):
    prompt = (
        "Evaluate this answer and provide critique if it is flawed.\n"
        f"Answer: {state['candidate_answer']}\n"
    )
    critique = critic_llm.invoke(prompt)
    return {"critique": critique}
```

10. **Mention evaluation and experimentation**
    - Encourage trying different agent roles, tool sets, and consensus mechanisms.
    - Recommend measuring performance and collecting labeled examples.

---

## Decision points and guidance

- **Can the task be split into specialized roles?** Yes — build modular agents.
- **Does the task need more than one perspective?** Use parallel agents and consensus.
- **Does the application need dynamic routing?** Use a semantic router.
- **Should agents communicate in natural language or structured data?** Prefer structured state for control, messages for flexibility.
- **Do you need production-strength orchestration?** Use LangGraph subgraphs and checkpoints.

---

## Quality criteria

A strong response should ensure the guidance is:

- **architectural:** focuses on agent roles and composition,
- **modular:** encourages reusable specialized agents,
- **consensus-driven:** explains how to choose among multiple outputs,
- **routed:** includes semantic routing or supervision patterns,
- **evaluative:** emphasizes experimentation and evaluation.

---

## Example prompts

- "How do I build a multi-agent architecture with LangGraph?"
- "What is a semantic router in a multi-agent system?"
- "How do I combine multiple agent answers using consensus?"
- "When should agents communicate via structured data versus shared messages?"

---

## Related guidance

Use this tool alongside:

- python-lang-graph-workflows
- python-lang-graph-agentic-architectures
- python-lang-graph-checkpoints
- python-lang-graph-error-handling
