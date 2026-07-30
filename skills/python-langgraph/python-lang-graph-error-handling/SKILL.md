---
name: python-lang-graph-error-handling
description: Guides teams to implement robust error handling and retries in Python LangGraph workflows, including exception capture, fallback chains, and parser recovery.
---

# Python LangGraph Error Handling and Retry Strategies

This skill helps AI explain how to build LangGraph workflows that tolerate failures. It covers exception handling in nodes, logging, retry policies, fallback chains, and parser recovery techniques.

---

## When to use this skill

Use this skill when you need to:

- explain how to catch and recover from workflow failures,
- build robust LangGraph nodes that log exceptions and continue safely,
- use LangChain retry and fallback mechanisms,
- recover from parser failures with error-aware retries,
- test workflows with fake LLMs and simulated failures.

---

## Outcome

Produce guidance that:

- encourages explicit exception handling inside LangGraph nodes,
- shows how to log and return safe default state values,
- demonstrates retry policies on nodes and runnables,
- explains fallback chains for alternative providers,
- includes example code for fake LLM testing and parser repair.

---

## Instructions for the AI

1. **Explain the need for error handling**
   - Emphasize that LLM workflows can fail due to API issues, bad output, or external services.
   - Explain the value of converting failures into structured state updates.

2. **Show node-level exception handling**
   - Provide a Python example wrapping node logic in `try/except`.

```python
import logging
from langchain_core.runnables.config import RunnableConfig

logger = logging.getLogger(__name__)

llms = {
    "fake": fake_llm,
    "Google": llm,
}

def analyze_job_description(state, config: RunnableConfig):
    try:
        model_provider = config["configurable"].get("model_provider", "Google")
        llm = llms[model_provider]
        analyze_chain = llm | parser
        prompt = prompt_template_enum.format(job_description=state["job_description"])
        result = analyze_chain.invoke(prompt)
        return {"is_suitable": result}
    except Exception as e:
        logger.error(f"Exception {e} occurred while executing analyze_job_description")
        return {"is_suitable": IsSuitableJobEnum.NO}
```

3. **Show fake LLM testing**
   - Use fake models to simulate failure and validate resilience.

```python
from langchain_core.language_models import GenericFakeChatModel
from langchain_core.messages import AIMessage

class MessagesIterator:
    def __init__(self):
        self._count = 0

    def __iter__(self):
        return self

    def __next__(self):
        self._count += 1
        if self._count % 2 == 1:
            raise ValueError("Something went wrong")
        return AIMessage(content="False")

fake_llm = GenericFakeChatModel(messages=MessagesIterator())
```

4. **Explain retries on runnables**
   - Show how to use `with_retry()` on a chain.

```python
fake_llm_retry = fake_llm.with_retry(
    retry_if_exception_type=(ValueError,),
    wait_exponential_jitter=True,
    stop_after_attempt=2,
)
analyze_chain_fake_retries = fake_llm_retry | parser
```

5. **Show node-specific retry policies**
   - Use `RetryPolicy` on LangGraph nodes.

```python
from langgraph.pregel import RetryPolicy

builder.add_node(
    "analyze_job_description",
    analyze_job_description,
    retry=RetryPolicy(retry_on=ValueError, max_attempts=2),
)
```

6. **Explain parser repair retries**
   - Show `RetryWithErrorOutputParser` for output fixing.

```python
from langchain.output_parsers import RetryWithErrorOutputParser

fix_parser = RetryWithErrorOutputParser.from_llm(
    llm=llm,
    parser=parser,
    prompt=retry_prompt,
)
fixed_output = fix_parser.parse_with_prompt(
    completion=original_response,
    prompt_value=original_prompt,
)
```

7. **Show fallback chains**
   - Demonstrate using `with_fallbacks()` to switch providers.

```python
from langchain_core.runnables import RunnableLambda

chain_fallback = RunnableLambda(lambda _: print("running fallback"))
chain = fake_llm | RunnableLambda(lambda _: print("running main chain"))
chain_with_fb = chain.with_fallbacks([chain_fallback])
chain_with_fb.invoke("test")
```

8. **Recommend logging and observability**
   - Encourage using structured logs and alerts for production.
   - Suggest converting exceptions to state values when workflows should continue.

9. **Advise on retry semantics**
   - Use retries for transient API or network failures.
   - Avoid naive retries for parser failures unless the prompt or parser is corrected.
   - Combine parser repair with semantic feedback when necessary.

---

## Decision points and guidance

- **Is failure transient?** Use retries.
- **Is failure due to bad output?** Use parser repair or alternative prompt strategies.
- **Should the workflow continue?** Return conservative default state and log the error.
- **Is a provider unavailable?** Use a fallback chain.
- **Do you need controlled retries?** Use node-specific `RetryPolicy` or runnable-level retries.

---

## Quality criteria

A strong response should ensure the guidance is:

- **robust:** catches and logs exceptions,
- **recoverable:** retries transient failures,
- **fallback-ready:** provides alternate execution paths,
- **parser-aware:** handles invalid output gracefully,
- **testable:** uses fake LLMs to simulate failures.

---

## Example prompts

- "How do I make LangGraph workflows resilient to LLM failures?"
- "What is the best way to retry a failed LangGraph node?"
- "How do I fallback to an alternate LLM provider in a chain?"
- "How can I recover from parsing errors in a LangGraph workflow?"

---

## Related guidance

Use this tool alongside:

- python-lang-graph-workflows
- python-lang-graph-state-management
- python-lang-graph-output-parsing
