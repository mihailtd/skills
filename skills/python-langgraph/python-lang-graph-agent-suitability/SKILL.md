---
name: python-lang-graph-agent-suitability
description: Guides teams through the two-stage decision that should precede any agent build — does this task even require an LLM (unstructured data, unpredictable input diversity), and if so, does it require an agent or will a workflow suffice (task complexity, task value vs. added cost/latency, error cost and detectability) — countering the tendency to reach for agents by default because they're new and powerful.
---

# Python LangGraph: Agent Suitability

This skill helps AI apply a disciplined, two-stage filter before recommending
an agent architecture: first, whether the task needs an LLM at all, and
second — only if it does — whether it needs the multi-step autonomy of a
true agent rather than a simpler workflow. It exists to counter the common
trap of applying agents to every task simply because they're new and
powerful, using concrete criteria and cost math rather than intuition.

---

## When to use this skill

Use this skill when you need to:

- decide, before any implementation, whether a task genuinely requires an
  LLM or whether traditional programming/ML would be faster, cheaper, and
  more reliable,
- decide, once an LLM is warranted, whether the task needs full agent
  autonomy or whether a single call, chain, or router pattern would suffice
  (see python-lang-graph-workflow-vs-agent-patterns for the pattern
  taxonomy itself),
- push back on a proposal to build an agent for a task that doesn't clearly
  need one, with concrete criteria rather than a vague "seems
  over-engineered" objection,
- estimate the real cost and latency implications of an agent design before
  committing to it,
- explain to stakeholders why "we could build this as an agent" isn't the
  same question as "we should."

---

## Outcome

Produce suitability guidance that:

- checks whether a task needs an LLM at all before considering agent
  design, using two concrete tests: whether the task involves unstructured
  data, and whether input/task diversity is high enough that hard-coded or
  specialized-model logic can't reasonably cover it,
- checks, only once an LLM is warranted, whether the task's step count is
  genuinely unpredictable — the core signal that favors an agent over a
  simpler workflow pattern,
- weighs task value against the real, multiplied cost and latency of an
  agent loop, using concrete unit economics rather than treating "it's more
  capable" as sufficient justification,
- weighs error cost and detectability specifically — how bad a mistake
  would be, and how likely anyone is to notice it — as a distinct criterion
  from complexity and value,
- actively counters reflexive agent adoption, naming it as a real, common
  failure mode rather than a hypothetical one.

---

## Instructions for the AI

1. **Stage one: does this task need an LLM at all?**
   - **Unstructured data test:** if the task requires analyzing
     unstructured input — text, images, audio — that's a strong signal an
     LLM (or multimodal LLM) is warranted, since traditional programming
     handles structured, well-schemed data well but struggles with the
     ambiguity inherent in natural language or visual content.
   - **Input diversity test:** if the range of possible inputs or requested
     tasks is small and predictable, recommend a specialized ML model or
     rule-based/hard-coded system instead — it will be more cost-effective
     in both compute and latency. Reserve LLM usage for situations where
     inputs are genuinely unpredictable and varied, requiring flexible
     interpretation that can't be fully anticipated in advance.
   - If neither test is met, recommend against using an LLM at all,
     regardless of how appealing an agent-based design might otherwise
     seem — this decision should be made before any agent-specific
     reasoning begins.

2. **Stage two: does this task need an agent, or will a workflow suffice?**
   - Only proceed to this stage once stage one has established that an LLM
     is genuinely needed.
   - **Task complexity:** ask whether it's possible to predict in advance
     how many steps the task will require. A task like "find the
     population of region A" is a single, predictable lookup — a workflow
     pattern suffices. A task like "analyze how LLM agents will shape the
     future" requires gathering diverse information and synthesizing
     relationships in a way that's genuinely hard to predefine with static
     logic — this favors an agent. Use a concrete multi-step research
     example to illustrate: a question requiring several dependent lookups
     and a calculation (e.g., "how long would it take to run from Earth to
     the Moon at a given pace") is exactly the shape of task where an agent's
     ability to search, gather, and compute across an unpredictable number
     of steps earns its cost.
   - **Task value:** since agent loops involve multiple LLM calls and
     dynamic decision-making, they cost more and respond more slowly than
     simpler patterns — recommend proceeding only when the value of
     completing the task clearly outweighs that added cost and latency.
     Make the cost multiplication concrete: a query handled by a single
     $0.01 LLM call versus the same query requiring a 10-step agent loop
     at roughly $0.10 is a 10x cost difference — at production volume
     (thousands of requests daily), frame this explicitly as a business
     decision, not just a technical one.
   - **Error cost and detectability:** weigh how costly a mistake would be
     if the agent gets something wrong, and — separately — how likely
     anyone would be to actually notice. Note that error risk compounds
     with more LLM calls, so longer agent loops carry more accumulated
     risk than a single call. Be especially cautious in domains requiring
     specialized knowledge, where neither users nor developers may
     recognize a faulty result when they see one. Recommend against an
     agent architecture when errors would be high-consequence and hard to
     detect, even if the task otherwise looks agent-shaped by complexity
     and value.

3. **Actively counter reflexive agent adoption**
   - Name the failure mode directly when it appears: teams reaching for
     agents because they're novel and capable, not because the task's
     criteria actually call for one — the same caution as reaching for a
     complex web framework to build a simple static site.
   - When a proposed agent design doesn't clearly pass the task-complexity,
     task-value, and error-cost/detectability criteria, recommend stepping
     back to python-lang-graph-workflow-vs-agent-patterns and considering
     whether a chain or router pattern would accomplish the same goal at
     lower cost, latency, and risk.

---

## Decision points and guidance

- **Does this task involve unstructured data, or is input diversity low
  enough for hard-coded/specialized-model logic?** If neither condition is
  met, don't recommend an LLM at all.
- **Can the number of steps this task requires be predicted in advance?**
  If yes, a workflow pattern likely suffices; if genuinely unpredictable,
  that favors an agent.
- **Does the task's value clearly exceed the multiplied cost and latency of
  an agent loop?** Use concrete unit economics (per-call cost × expected
  steps × request volume), not intuition, to answer this.
- **How bad would an error be, and how likely is it to be caught?** High
  cost combined with low detectability is a strong argument against an
  agent architecture regardless of the other criteria.
- **Is "agent" being proposed because it's the best fit, or because it's
  the exciting option?** If the criteria above don't clearly support it,
  say so directly and propose the simpler alternative.

---

## Quality criteria

A strong agent-suitability assessment should ensure that:

- **the LLM-necessity question is asked first:** no agent design discussion
  begins before confirming an LLM is actually warranted,
- **task complexity is judged by predictability, not perceived
  sophistication:** unpredictable step count is the real signal, not how
  impressive the task sounds,
- **cost is quantified, not assumed:** a concrete cost/latency comparison
  is made between the simplest sufficient pattern and the proposed agent
  design,
- **error cost and detectability are assessed as a distinct criterion:**
  not folded into or assumed to be covered by the complexity or value
  checks,
- **reflexive agent adoption is named and resisted:** proposals that don't
  clearly meet the criteria are redirected to simpler workflow patterns.

---

## Example prompts

- "Do we actually need an LLM for this task, or would a rule-based system
  work just as well?"
- "This task could be a single LLM call or a full agent — help me figure
  out which one actually makes sense given the cost and value involved."
- "We're about to build an agent for this — talk me out of it if the
  criteria don't support it."

---

## Related guidance

Use this tool alongside:

- python-lang-graph-workflow-vs-agent-patterns
- python-lang-graph-agentic-architectures
- python-lang-graph-error-handling
