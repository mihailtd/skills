---
name: python-lang-graph-output-parsing
description: Guides teams to parse and validate LLM outputs in Python LangGraph workflows, including prompt engineering, output parsers, and controlled generation patterns.
---

# Python LangGraph Output Parsing and Prompt Engineering

This skill helps AI explain how to build reliable LangGraph workflows by designing prompts, parsing LLM output, and integrating parsing into graph nodes. It includes parser examples, enum parsing, and structured output patterns.

---

## When to use this skill

Use this skill when you need to:

- explain prompt design for controlled generation,
- parse LLM text output into structured values,
- integrate LangChain output parsers into LangGraph nodes,
- handle enum, JSON, and schema-constrained outputs,
- minimize downstream workflow failures from unstructured output.

---

## Outcome

Produce guidance that:

- explains why output parsing is essential for workflow reliability,
- shows how to use LangChain `OutputParser` classes,
- demonstrates enum parsing and prompt constraints,
- integrates parsing into LangGraph nodes and conditional logic,
- highlights controlled generation and programmatic chaining.

---

## Instructions for the AI

1. **Explain controlled output generation**
   - Define controlled generation as forcing an LLM to emit output in a predictable shape.
   - Explain that unstructured text can break downstream workflow steps.
   - Emphasize prompt engineering and parser validation.

2. **Show a prompt that constrains output**

```python
prompt_template_enum = (
    "Given a job description, decide whether it suits a junior Java developer."
    "\nJOB DESCRIPTION:\n{job_description}\n\nAnswer only YES or NO."
)
```

3. **Demonstrate enum parsing**
   - Use `EnumOutputParser` and `Enum` for discrete decisions.

```python
from enum import Enum
from langchain.output_parsers import EnumOutputParser

class IsSuitableJobEnum(Enum):
    YES = "YES"
    NO = "NO"

parser = EnumOutputParser(enum=IsSuitableJobEnum)
assert parser.invoke("NO") == IsSuitableJobEnum.NO
assert parser.invoke(" YES \n") == IsSuitableJobEnum.YES
```

4. **Show a LangChain chain with a parser**

```python
chain = llm | parser
result = chain.invoke(prompt_template_enum.format(job_description=job_description))
```

5. **Integrate parsing into a LangGraph node**
   - Show how to use the parser inside a node and store structured output in state.

```python
class JobApplicationState(TypedDict):
    job_description: str
    is_suitable: IsSuitableJobEnum
    application: str

analyze_chain = llm | parser

def analyze_job_description(state: JobApplicationState):
    prompt = prompt_template_enum.format(job_description=state["job_description"])
    result = analyze_chain.invoke(prompt)
    return {"is_suitable": result}
```

6. **Explain conditional edge selection using parsed output**
   - Show mapping returned values to destination nodes.

```python
builder.add_conditional_edges(
    "analyze_job_description",
    is_suitable_condition,
    {True: "generate_application", False: END},
)
```

7. **Discuss structured parsing patterns**
   - Mention JSON, CSV, Pydantic models, and other parser types.
   - Explain that structured generation is key for downstream programmatic consumption.

8. **Warn about common parser failure modes**
   - Explain that the LLM may still return invalid text.
   - Suggest validating parser output and using retries or fallback strategies.

---

## Decision points and guidance

- **Does the next step require structured data?** Use an output parser.
- **Is the desired result a simple enum?** Use `EnumOutputParser`.
- **Should the LLM output be programmatically consumed?** Use controlled generation and schema validation.
- **Can the parser fail?** Build error handling around parser invocation.
- **Do you need robust prompts?** Add explicit instructions and examples.

---

## Quality criteria

A strong response should ensure the guidance is:

- **structured:** promotes reliable parsing over free text,
- **prompt-aware:** shows how prompt constraints support parsing,
- **parser-based:** uses LangChain output parser classes,
- **workflow-ready:** integrates parsing into LangGraph nodes,
- **resilient:** anticipates parser failures and invalid output.

---

## Example prompts

- "How can I parse LLM output into an enum in LangGraph?"
- "What is controlled generation and why does it matter for workflows?"
- "How do I integrate LangChain output parsers into LangGraph nodes?"
- "How should I handle structured output in a LangGraph workflow?"

---

## Related guidance

Use this tool alongside:

- python-lang-graph-workflows
- python-lang-graph-state-management
- python-lang-graph-error-handling
