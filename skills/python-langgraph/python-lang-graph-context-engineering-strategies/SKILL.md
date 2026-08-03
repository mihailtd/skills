---
name: python-lang-graph-context-engineering-strategies
description: Guides teams to design an agent's context pipeline using the five context engineering strategies — Generation (LLM-produced plans/reflections fed back into context), Retrieval (pulling in external information), Write (persisting context to external storage), Reduce (summarizing/filtering to fight Context Rot), and Isolate (separating tasks into distinct contexts/environments) — as a shared vocabulary for reviewing and designing agent context management.
---

# Python LangGraph: Context Engineering Strategies

This skill helps AI apply the five-strategy taxonomy for context
engineering — Generation, Retrieval, Write, Reduce, Isolate — as a concrete
design and review vocabulary for how an agent's context is built, persisted,
trimmed, and partitioned. It's the "how" companion to
python-lang-graph-context-engineering-fundamentals, which covers the "why"
(diagnosing failures, avoiding Context Rot).

---

## When to use this skill

Use this skill when you need to:

- design how an agent should build, maintain, and trim its context across a
  multi-step run,
- review an existing agent's context pipeline and classify what it's
  currently doing (and not doing) across the five strategies,
- decide whether a context problem needs more retrieval, less content, or
  a structural separation into isolated sub-contexts,
- connect context-engineering concerns to the concrete LangGraph
  subsystems that implement them (memory, tools, multi-agent architecture,
  planning/reflection nodes),
- explain to a team why memory design specifically touches three of the
  five strategies at once.

---

## Outcome

Produce guidance that:

- classifies context-management work into one or more of the five named
  strategies, so design discussions have a shared, specific vocabulary
  instead of a vague "manage context better,"
- recommends Generation for context the agent should build for itself
  (plans, reflections) rather than only ever pulling in external content,
- recommends Retrieval for pulling necessary external information into
  context (search, database queries, file reads, vector-store lookups),
  scoped to what's actually needed for the current task,
- recommends Write for persisting information out of the context window
  into external storage, so it survives beyond the window's limits and can
  be reloaded later,
- recommends Reduce specifically as the countermeasure to Context Rot
  (see python-lang-graph-context-engineering-fundamentals) — summarizing,
  deleting, or filtering by importance to keep the model's attention on
  what matters,
- recommends Isolate for partitioning tasks or tools into separate
  contexts/environments (sandboxes, specialized sub-agents) so no single
  context has to hold everything at once,
- identifies memory design as the point where Retrieval, Write, and Reduce
  converge, and treats it accordingly as a higher-complexity design surface.

---

## Instructions for the AI

1. **Generation — build context from the LLM's own output**
   - Recommend this strategy when the agent should structure or evolve its
     own context rather than only ingesting external material — e.g.,
     generating an explicit plan before tackling a complex, multi-step task,
     or reflecting on completed work to revise its own strategy mid-run.
   - Frame Generation as giving the agent a way to actively shape its own
     working context (turning intermediate reasoning into structured
     context content), which is distinct from simply retrieving more
     external information.
   - Connect this to planning/reflection-oriented LangGraph nodes (a node
     that emits a plan before execution, or a node that critiques and
     revises a prior answer).

2. **Retrieval — bring external information into context**
   - Recommend this strategy whenever the task needs information the model
     doesn't already have from training or its current context — web
     search, database queries, file reads, or document retrieval from a
     vector database.
   - Treat this as the core mechanism that lets an agent access up-to-date
     or domain-specific knowledge, but scope what's retrieved to what the
     current task actually needs (see step 4 on Reduce) rather than
     retrieving broadly and hoping relevance sorts itself out.
   - Cross-reference python-lang-graph-agentic-rag for the deeper treatment
     of retrieval design (dynamic retrieval, query expansion, chunk
     reasoning, iterative adaptation).

3. **Write — persist context out to external storage**
   - Recommend this strategy for information that needs to survive beyond
     the current context window or the current run — saving important
     conversation content to long-term memory, recording intermediate
     calculation results to a scratchpad, or saving generated code to
     files.
   - Frame Write as the mechanism that overcomes the context window's
     inherent limits: information doesn't have to stay in the active
     context to remain available — it can be written out and reloaded via
     Retrieval later, decoupling "what the agent currently has in view"
     from "what the agent has ever learned or produced."
   - Cross-reference python-lang-graph-agent-memory and
     python-lang-graph-checkpoints for concrete persistence mechanisms.

4. **Reduce — shrink the context to fight Context Rot**
   - Recommend this strategy specifically as the countermeasure to Context
     Rot and the Lost in the Middle effect (see
     python-lang-graph-context-engineering-fundamentals): summarizing older
     conversation turns, deleting information that's no longer relevant,
     or filtering content by importance before it's allowed into the active
     context.
   - Treat Reduce as essential, not optional cleanup — its purpose is
     keeping the model's attention concentrated on what actually matters
     for the current step, not just saving token budget.
   - Apply this at multiple points: when loading retrieved documents (only
     keep the genuinely relevant excerpts), when carrying conversation
     history forward (summarize rather than replaying verbatim), and when
     incorporating tool results (extract the relevant finding rather than
     dumping raw output).

5. **Isolate — separate tasks or tools into distinct environments**
   - Recommend this strategy when a single context would otherwise become
     overloaded by handling too many concerns at once — executing complex
     or risky code in a sandboxed environment, or assigning different
     specialized domains to separate agents rather than one agent juggling
     everything.
   - Frame Isolate as a structural strategy: instead of managing one large,
     increasingly complex context, partition the problem so each isolated
     context can be smaller, more focused, and easier to keep relevant
     (directly supporting the Reduce goal within each partition).
   - Cross-reference python-lang-graph-multi-agent-architectures for the
     concrete multi-agent patterns that implement this strategy, and
     python-lang-graph-workflow-vs-agent-patterns for the related pattern
     of bounding agent scope within a larger workflow.

6. **Treat memory design as the convergence point of three strategies**
   - Recognize that memory systems inherently involve Retrieval (pulling
     past experience back into context), Write (storing new experience out
     of context), and Reduce (summarizing or deleting old memories) all at
     once — this is why memory design tends to be more complex than
     applying any single strategy in isolation.
   - When reviewing or designing a memory system, explicitly check all
     three angles rather than treating memory as a single undifferentiated
     concern: how is information written out, how is it retrieved back in,
     and how is old or low-value memory reduced over time?

---

## Decision points and guidance

- **Does this context problem need more information, or better-curated
  information?** If the agent is missing something, that's Retrieval; if
  it has too much irrelevant material, that's Reduce — don't default to
  "add more" without checking which is actually true.
- **Should this information live in the active context, or be written out
  for later?** Information needed only for the current step stays in
  context; information that needs to persist across runs or exceed the
  window's capacity should go through Write.
- **Is the agent generating its own useful context (plans, reflections), or
  only consuming external content?** If it's only consuming, consider
  whether a Generation step (planning, self-critique) would improve
  performance.
- **Is a single context trying to do too much at once?** If a task spans
  clearly separable concerns, consider Isolate — a sandboxed execution
  environment or a dedicated sub-agent — rather than growing one shared
  context further.
- **Does this design touch memory?** If so, explicitly check its Retrieval,
  Write, and Reduce behavior as three separate questions, not one.

---

## Quality criteria

A strong context-engineering-strategies design should ensure that:

- **every context-management decision maps to a named strategy:** design
  discussions use Generation/Retrieval/Write/Reduce/Isolate as shared
  vocabulary, not vague descriptions,
- **Retrieval is scoped, not broad:** external information is pulled in
  based on actual task need, with Reduce applied to trim it further,
- **Write is used for anything that must outlive the active context:**
  scratchpads, long-term memory, and generated artifacts are persisted
  rather than assumed to remain in context indefinitely,
- **Reduce is treated as a first-class, ongoing concern:** summarization
  and filtering happen throughout the pipeline (retrieval, history, tool
  results), not just as an occasional cleanup step,
- **Isolate is used to bound complexity:** tasks that would otherwise
  overload a single context are structurally separated into sandboxes or
  specialized sub-agents,
- **memory design is checked against all three converging strategies**
  (Retrieval, Write, Reduce), not treated as a single undifferentiated
  concern.

---

## Example prompts

- "Classify what our current context pipeline is doing across the five
  context engineering strategies, and tell me what's missing."
- "Our agent's context keeps growing every turn — should we be summarizing,
  writing things out, or isolating parts of the task?"
- "Help me design the memory system for this agent, and make sure we've
  thought through retrieval, writing, and reduction separately."

---

## Related guidance

Use this tool alongside:

- python-lang-graph-context-engineering-fundamentals
- python-lang-graph-agent-memory
- python-lang-graph-agentic-rag
- python-lang-graph-multi-agent-architectures
- python-lang-graph-checkpoints
- python-lang-graph-workflow-vs-agent-patterns
