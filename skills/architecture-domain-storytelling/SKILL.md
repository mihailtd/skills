---
name: architecture-domain-storytelling
description: Guides collaborative domain storytelling as an architectural practice for capturing business scenarios, domain concepts, and shared understanding across teams.
---

# Architecture Domain Storytelling

This skill helps architects and cross-functional teams capture business domain
knowledge through collaborative storytelling. It is designed for workshop-style
sessions where domain experts, developers, analysts, and designers co-create a
shared understanding of how the business works.

---

## When to use this skill

Use this skill when you need to:

- Capture business processes and domain intent in a way that both business and
  technical stakeholders can understand.
- Build a shared language between domain experts, product teams, and developers.
- Uncover misunderstandings, contradictions, or gaps early in the design process.
- Shape software requirements around real domain behavior rather than implementation
  details.
- Create a narrative that informs architecture, requirements, and design.

---

## Outcome

Produce a domain story that:

- describes who acts, what they do, and what domain objects they work with,
- reads like a sequence of meaningful sentences,
- keeps actors and domain objects distinct and explicit,
- avoids conditional branching in a single story,
- reflects the spoken business scenario clearly for review and validation.

---

## Step-by-step workflow

1. **Prepare the session**
   - Gather domain experts, architects, developers, and analysts.
   - Clarify that the goal is to capture a real business scenario as a shared
     domain story.

2. **Listen for a concrete narrative**
   - Ask domain experts to tell a real-world example in their own words.
   - Keep the story grounded in what people and systems actually do.

3. **Identify the actors**
   - Determine who participates in the scenario: people, teams, or systems.
   - Represent actors with clear labels and simple visuals.
   - Keep each actor conceptually unique and avoid duplicating roles.

4. **Capture work objects for each sentence**
   - For each sentence, identify the domain object that is being acted upon.
   - Use a distinct object for each sentence to preserve clarity.
   - Label each object with a noun from the domain language.

5. **Describe activities explicitly**
   - Define the verbs that connect actors to work objects.
   - Keep the object separate from the verb (e.g., “searches for” + “available
     seats”).
   - Use the pattern: actor → activity → domain object.

6. **Model collaboration and handoffs**
   - Capture handovers as one actor passing a work object to another.
   - Capture collaboration as multiple actors working together on a domain object.
   - Add clarifying terms like “to”, “with”, or “in” when needed.

7. **Add sequence order**
   - Number the steps in the order they occur.
   - Place each sequence marker near the initiating actor for clarity.
   - Use shared numbers for actions that happen simultaneously or together.

8. **Annotate assumptions and rules**
   - Add notes for assumptions, triggers, or domain definitions.
   - Place clarifications near the relevant actor, object, or activity.

9. **Group related story segments**
   - Cluster related steps into meaningful groups, such as phases or scopes.
   - Label the group to explain its purpose.
   - Keep actors outside groups if they participate across multiple segments.

10. **Avoid technical request/response patterns**
    - Do not model back-and-forth system calls or implementation details.
    - Focus on the user’s intent and the business result.
    - Avoid drawing loops back to the same actor purely for technical reasons.

11. **Separate alternative flows**
    - Create a new domain story for important variations.
    - Do not include if/else branches or gateways in the same story.

12. **Review and validate**
    - Read the story aloud and confirm it makes sense as a sentence.
    - Validate the story with domain experts and technical stakeholders.
    - Refine labels and remove ambiguity.

---

## Decision points and guidance

- **Use domain language:** Prefer business terms over technical jargon.
- **Actors as subjects:** Actors should appear only once and remain the story
  subjects.
- **Explicit objects:** Keep domain objects visible and separate from activity labels.
- **One object per sentence:** This keeps the story readable and reduces overlap.
- **Linear storytelling:** Keep each story focused and avoid embedding decision
trees.

---

## Quality criteria

A strong domain story should be:

- **Collaborative:** created with input from domain experts and technical teams.
- **Readable:** easy to follow without decoding hidden meaning.
- **Explicit:** every actor, object, and activity is clearly labeled.
- **Domain-oriented:** based on actual business behavior.
- **Focused:** avoids conditional branching in a single story.
- **Validated:** confirmed by the stakeholders who know the domain best.

---

## Example prompt

- "Show me how to run a domain storytelling session that captures a pharmacy
  checkout scenario in business terms."
- "Explain why domain storytelling should model user intent rather than request/
  response technical interactions."
- "Describe how to capture a spoken business process as a sequence of actors,
  activities, and domain objects."

---

## Related guidance

This skill complements:

- Business Process Map
- Stakeholder Map
- Application/Information Concept Diagram
- Architecture Roadmap
