---
name: python-lang-graph-context-engineering-fundamentals
description: Guides teams to diagnose agent failures correctly — distinguishing insufficient model intelligence from the far more common cause, missing information in context — and to treat context engineering (what information reaches the model, when, and in what form) as the primary lever for agent quality, including the "bigger context isn't better" finding (Context Rot, Lost in the Middle) that argues for selective inclusion over maximal inclusion.
---

# Python LangGraph: Context Engineering Fundamentals

This skill helps AI correctly diagnose why an agent is failing and reframe
context design as a first-class engineering discipline rather than an
afterthought to prompt writing. It covers the prompt-vs-context distinction,
the two root causes of agent failure (and why information deficiency
dominates in practice), the historical shift from prompt engineering to
context engineering as agents started using tools, and the finding that
larger context windows don't straightforwardly improve results.

---

## When to use this skill

Use this skill when you need to:

- diagnose why an agent is producing wrong or incomplete answers, before
  assuming the model itself is the problem,
- explain to a team why "just write a better prompt" is an incomplete
  strategy once an agent is using tools and accumulating dynamic context,
- decide whether to dump more information into an agent's context or to
  actively curate what it receives,
- review an agent design for context quality, not just prompt wording,
- set expectations about what a larger context window can and can't fix.

---

## Outcome

Produce guidance that:

- distinguishes prompt (the input text sent to the LLM — system + user
  prompts) from context (everything the LLM references when generating a
  response, including prompts, history, tool results, and retrieved
  documents) — and treats context as the broader, more consequential
  concept,
- diagnoses agent failures against two distinct root causes — insufficient
  model intelligence versus missing information in context — and correctly
  weights the second as the dominant real-world cause,
- explains why prompt engineering alone became insufficient once agents
  began incorporating dynamic, tool-derived content into their context,
- treats "what information should be in the context, and when" as the
  central design question, going beyond prompt wording to cover tool-result
  processing, history retention policy, and retrieval timing,
- recommends selective, curated context over maximal context, citing the
  documented performance degradation (Context Rot, the Lost in the Middle
  effect) that comes from overfilling the context window even when the
  window is technically large enough to hold everything.

---

## Instructions for the AI

1. **Distinguish prompt from context precisely**
   - **Prompt:** the input text sent to the LLM, split into the system
     prompt (behavioral/role instructions, e.g., "You are a friendly travel
     guide") and user prompts/messages (the actual request).
   - **Context:** the full set of information the LLM references when
     generating a response — this includes the prompt, but also prior
     conversation history, tool execution results, retrieved documents, and
     anything else made available to the model. Prompts are one component
     of context, not a synonym for it.
   - Use the working-memory analogy when explaining this to a team: an LLM
     can't complete a task if crucial information is missing from its
     context, the same way a person can't solve a math problem given only
     half the equation — the model predicts its next output purely from
     what's actually present in the context window, drawing on patterns
     learned in training to use that information, not on anything outside
     it.

2. **Diagnose agent failures against the correct root cause**
   - **Insufficient model intelligence:** the model has the relevant
     information but still fails — e.g., a genuine logical or mathematical
     reasoning error. The fix here is a more capable model, or waiting on
     model improvements — it is not fixable through better context design.
   - **Information deficiency:** the model lacks necessary information in
     its context — e.g., asking about a company's vacation policy when the
     policy document was never retrieved into context. No amount of model
     capability fixes this; even the most capable model can't answer
     accurately without the relevant information present.
   - When diagnosing a failure, check information deficiency first: a
     significant share of real-world agent failures come from this cause
     rather than from model capability limits, so this should be the
     default hypothesis to rule out before concluding "the model isn't
     smart enough." Recommend verifying what was actually present in the
     context at generation time before recommending a bigger or different
     model.

3. **Explain the shift from prompt engineering to context engineering**
   - Describe the historical starting point: early LLM practice focused on
     prompt engineering — crafting system prompts and instruction phrasing
     (e.g., "let's think step by step," "you are an expert") to get more
     accurate or better-formatted outputs from an otherwise static prompt.
   - Explain what changed: once agents began using tools, context stopped
     being static. Web search results, code execution output, and database
     query results all get added to the context dynamically as the agent
     runs — the earlier "write one good static prompt" framing became
     insufficient once context was accumulating and changing across a
     multi-step run.
   - Define context engineering as the resulting, broader discipline:
     providing the LLM the information it needs, at the right time and in
     the right form — covering not just prompt wording, but how tool
     results are processed before being added to context, how much
     conversation history is retained, and when external knowledge should
     be retrieved. Treat prompt engineering as a subset of context
     engineering, not a competing discipline.

4. **Recommend selective context over maximal context**
   - Note that modern context windows have grown dramatically (commonly
     hundreds of thousands to a million tokens), which can tempt teams to
     assume "just put everything in context" is a safe default.
   - Push back on that assumption directly: research shows model
     performance degrades as context length increases, a phenomenon
     referred to as Context Rot — with the "Lost in the Middle" effect
     specifically documenting that models tend to miss important
     information located in the middle of long contexts, even when it's
     technically present.
   - Recommend the resulting design principle explicitly: maximize
     relevance, not volume. Providing more information is not the same as
     providing better information — when context is filled with
     unnecessary material, crucial details can get buried or the model's
     attention can become diluted across irrelevant content. Recommend
     actively curating what enters context rather than defaulting to
     including everything that might conceivably be useful (see
     python-lang-graph-context-engineering-strategies for the concrete
     strategies — especially Reduce and Isolate — that operationalize this
     principle).

---

## Decision points and guidance

- **Is this failure a reasoning error, or a missing-information error?**
  Check what was actually in context at generation time before assuming
  the model itself is the limiting factor — missing information is the more
  common cause in practice.
- **Is the agent's context static or dynamic?** Once tools are involved
  (search, execution, queries), treat context design as an ongoing
  engineering concern, not a one-time prompt-writing task.
- **Is "add more to the context" actually the right instinct here?** Check
  whether the addition is genuinely relevant, or whether it risks
  contributing to Context Rot by diluting the model's attention with
  low-value content.
- **Is a large context window being treated as a substitute for
  curation?** A bigger window increases *capacity*, not relevance — don't
  let window size substitute for deciding what actually belongs in context.

---

## Quality criteria

A strong context-engineering-fundamentals response should ensure that:

- **failures are correctly attributed:** information deficiency is checked
  before concluding the model itself is insufficiently capable,
- **prompt and context are not conflated:** the prompt is understood as one
  component of the broader context, not a synonym for it,
- **context is treated as dynamic:** tool results, retrieved documents, and
  history are recognized as part of the design surface, not just the
  initial system/user prompt,
- **relevance is prioritized over volume:** context design actively curates
  for relevance rather than defaulting to inclusion of everything
  available,
- **large context windows aren't mistaken for a solved problem:** window
  size is treated as capacity, with curation still required regardless of
  how much room is technically available.

---

## Example prompts

- "Our agent gave a wrong answer — help me figure out if that's a model
  capability issue or a missing-context issue before I switch models."
- "We just started giving our agent tool access and our old prompts don't
  seem to be enough anymore — what changed?"
- "Should we just stuff everything we might need into this agent's context
  window since it's large enough to hold it?"

---

## Related guidance

Use this tool alongside:

- python-lang-graph-context-engineering-strategies
- python-lang-graph-agent-suitability
- python-lang-graph-workflow-vs-agent-patterns
- python-lang-graph-agent-memory
