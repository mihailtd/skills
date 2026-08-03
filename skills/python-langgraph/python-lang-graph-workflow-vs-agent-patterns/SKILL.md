---
name: python-lang-graph-workflow-vs-agent-patterns
description: Guides teams to choose the right level of LLM autonomy for a given step — from a single LLM call, through a predefined chain or router, up to a tool-using multi-step agent loop or dynamic tool creation — and to combine workflows and agents deliberately (agents as bounded nodes inside a larger workflow) rather than defaulting to full agency everywhere.
---

# Python LangGraph: Workflow vs. Agent Patterns

This skill helps AI explain the spectrum of control between fully
developer-defined workflows and fully LLM-directed agents, and how to design
systems that use each pattern where it actually earns its cost. It covers
the three workflow patterns (LLM call, chain, router), the two agent
patterns (tool-use loop, tool creation), and the common production pattern
of embedding an agent as one bounded stage inside an otherwise predictable
workflow.

---

## When to use this skill

Use this skill when you need to:

- decide how much control to hand to an LLM for a given piece of
  functionality, before writing any LangGraph-specific code,
- explain the difference between a chain, a router, and a true agent to a
  team that's using the terms loosely or interchangeably,
- design a production system that mixes predictable structure with
  LLM-directed flexibility, rather than committing entirely to one or the
  other,
- review an existing design and determine whether a component currently
  built as a full agent could be simplified to a chain or router (or vice
  versa),
- explain to stakeholders why "agent" doesn't mean "maximally autonomous
  everywhere" — it's a spectrum, and most production systems live in the
  middle of it.

---

## Outcome

Produce guidance that:

- frames the workflow/agent choice along three concrete dimensions: who
  produces the current output, who decides the next step, and who defines
  the set of available next steps — increasing LLM control across each
  dimension moves a design from workflow toward agent,
- correctly places a proposed design into one of the five named patterns
  (LLM call, chain, router, tool-use loop, tool creation), and explains what
  distinguishes each from its neighbors,
- treats a router as a workflow pattern, not an agent pattern — the LLM
  picks among developer-defined options, but doesn't control what happens
  along the chosen path,
- identifies the multi-step loop (assess, decide, observe, repeat until the
  LLM itself determines completion) as the defining mechanism that turns
  tool use into a true agent,
- recommends the hybrid pattern — agents as bounded nodes inside a larger
  workflow — as the default production architecture, and explains the
  concrete reasons this containment pays off.

---

## Instructions for the AI

1. **Explain the three dimensions of control**
   - **Current step:** who produces the actual output right now — the LLM,
     or predefined logic?
   - **Next step:** who decides what happens after this step — the LLM, or
     the developer's predefined flow?
   - **Available steps:** who defines the set of options that could happen
     next — a fixed, developer-specified set, or something the LLM can
     expand on its own (including creating entirely new tools)?
   - Frame every pattern below in terms of how much control each dimension
     hands to the LLM — this is what makes the workflow-to-agent
     progression a spectrum rather than a binary choice.

2. **Explain the workflow patterns (developer-defined flow)**
   - **LLM call:** the simplest pattern — one prompt in, one response out,
     with everything else under developer control. Appropriate for
     well-scoped tasks like classification, summarization, or answering a
     direct question. Note that a well-crafted single call can handle more
     complex tasks than it might seem.
   - **Chain:** multiple LLM calls connected in a developer-defined
     sequence, where each step's output feeds the next step's input.
     Recommend this specifically because LLMs perform better on focused,
     well-defined subtasks than on one large, compound instruction — e.g.,
     breaking "analyze this document and create a presentation" into
     extract-key-points, organize-by-theme, generate-slide-content,
     refine-for-clarity as separate steps, each more likely to succeed on
     its own.
   - **Router:** the LLM makes a decision — classifying input and selecting
     one path from a fixed, developer-defined set (e.g., billing questions
     go to one workflow, technical support to another). Emphasize that a
     router is still a workflow pattern: the LLM influences *which*
     predefined path is taken, but the developer retains control over what
     each path actually does and what the full set of paths is.

3. **Explain the agent patterns (LLM-directed flow)**
   - **Tool use with a multi-step loop:** this is the defining agent
     pattern. Since LLMs can only generate text, tools expose external
     capabilities (search, code execution, database/API access) as
     callable functions the LLM can choose to invoke. What makes this an
     *agent* rather than a single tool call is the loop: assess the
     current state, decide on an action, observe the result, and repeat —
     with the LLM itself deciding when the task is actually complete,
     rather than stopping after one call. Note that this loop is what lets
     an agent handle work of unpredictable length: a simple query might
     resolve in one iteration, a complex research task might take dozens.
   - **Tool creation:** the highest level of autonomy — the agent doesn't
     just select from a predefined toolkit, it generates new tools (e.g.,
     writing and executing custom code) to handle needs the original
     design didn't anticipate. Frame this as expanding the "available
     steps" dimension dynamically, at runtime, rather than just picking
     among options the developer already enumerated.

4. **Recommend the hybrid pattern as the default for production systems**
   - Describe the common, effective pattern: an overall workflow with
     well-defined stages, where one or more specific stages are
     implemented as an agent rather than as fixed logic — e.g., a document
     pipeline where intake and delivery are workflow stages, but content
     analysis (research, synthesis) is handled by an agent operating
     freely within that one stage, followed by a workflow-based quality
     review stage that validates the agent's output before it proceeds.
   - Recommend this hybrid approach as the default rather than an
     exception, and explain the three concrete reasons it pays off:
     - **Contains complexity:** agent behavior is inherently less
       predictable, so bounding where it's allowed to operate keeps the
       overall system easier to reason about and debug.
     - **Optimizes cost:** agent loops multiply LLM calls, so restricting
       agent behavior to the stages that actually need its flexibility
       keeps operational cost manageable (see
       python-lang-graph-agent-suitability for the concrete cost math).
     - **Enables graceful degradation:** wrapping an agent stage in a
       workflow means a failing or unexpected agent result can be caught
       at a defined checkpoint (e.g., a quality-review stage), with
       fallback behavior or human escalation available, rather than the
       failure propagating unchecked through the rest of the system.
   - Reframe the design question explicitly as "where in this system does
     agent behavior provide enough value to justify its cost?" rather than
     a global "workflow or agent" choice for the whole system.

---

## Decision points and guidance

- **Does the LLM decide what happens next, or just produce output within a
  fixed step?** If it's the latter, the pattern is a workflow (LLM call or
  chain), not an agent.
- **Does the LLM choose among developer-defined paths, or does it operate a
  loop it controls the exit condition for?** The former is a router
  (workflow); the latter is an agent.
- **Can the LLM only select from a fixed toolkit, or can it create new
  tools at runtime?** The latter is the highest-autonomy agent pattern
  (tool creation) and should be reserved for cases that genuinely need it.
- **Does this component need full agent autonomy, or just one bounded
  agentic stage inside a larger predictable flow?** Default to the bounded
  hybrid pattern unless there's a specific reason the whole system needs to
  be agent-directed.
- **If an agent stage fails or produces a bad result, is there a workflow
  checkpoint that catches it?** If not, recommend adding one (a
  quality-review stage, a validation step, a human-escalation path) rather
  than letting agent output flow through unchecked.

---

## Quality criteria

A strong workflow-vs-agent design should ensure that:

- **the control level matches the task:** simple, well-scoped work uses an
  LLM call or chain; only genuinely unpredictable, multi-step work uses a
  true agent loop,
- **routers aren't mistaken for agents:** LLM-driven path selection among
  fixed, developer-defined options is correctly classified as a workflow
  pattern,
- **agent scope is bounded:** agent behavior is contained to the specific
  stages that need it, wrapped in workflow structure elsewhere,
- **failure is caught, not propagated:** agent stages have a defined
  workflow checkpoint (validation, review, escalation) downstream,
- **the design question is asked per-component, not globally:** "where does
  agent behavior earn its cost here?" rather than committing the whole
  system to one paradigm.

---

## Example prompts

- "Is this feature a chain, a router, or does it actually need to be a full
  agent?"
- "Help me design a document-processing pipeline that uses an agent for the
  research stage but keeps the rest of the pipeline predictable."
- "We built this as a fully autonomous agent — could part of it be
  simplified to a router instead?"

---

## Related guidance

Use this tool alongside:

- python-lang-graph-agent-suitability
- python-lang-graph-agentic-architectures
- python-lang-graph-workflows
- python-lang-graph-multi-agent-architectures
