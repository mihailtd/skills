---
name: architecture-diagrams-domain-modelling
description: Guides the creation of domain story diagrams using a pictographic language that captures actors, work objects, activities, and collaboration clearly without branching logic.
---

# Domain Modelling Story — Pictographic Domain Stories

This skill helps architects and analysts capture domain behaviour as a sequence of
simple domain stories using a pictographic grammar: who does what with what, and
with whom. It is ideal for communicating business intent without becoming a
complex flowchart.

---

## When to use this skill

Use this skill when you need to:

- Model a business scenario as a sequence of domain actions.
- Explain how actors interact with domain objects without exposing internal
  branching logic.
- Capture domain concepts in a visual story that is easy for stakeholders to
  read.
- Use a shared pictographic notation to keep domain language explicit.
- Compare alternative paths by creating separate domain stories instead of
  folding conditionals into one diagram.

---

## Outcome

Produce a domain story diagram that contains:

- **Actors**: people, groups, or systems that play an active role in the story.
- **Work objects**: the things actors create, use, exchange, or share.
- **Activities**: labelled arrows showing what actors do with the work objects.
- **Sequence numbers**: chronological order markers for each action.
- **Annotations and groups**: supporting notes and logical clusters for clarity.

The story should read like a sentence: **actor → activity → work object → other
actor**, with each step clearly numbered and visually separated.

---

## Step-by-step workflow

1. **Identify the actors**
   - Determine the people, teams, or systems involved in the scenario.
   - Use simple icons, stick figures, or system symbols.
   - Draw actors larger than work objects and place them only once per story.

2. **Identify the work objects**
   - Capture the domain nouns: documents, permits, tickets, requests, results,
     or digital items.
   - Draw one work object icon per sentence to avoid overlapping arrows.
   - Allow the same work object type to appear multiple times in the story.

3. **Define the activities**
   - Write verbs from the domain language that describe what each actor does.
   - Draw arrows from the actor to the work object and label them with the
     activity.
   - Keep the work object explicit rather than embedding it inside the verb.

4. **Number the sequence**
   - Add sequence numbers at the start of each arrow near the initiating actor.
   - Circle the numbers for visibility.
   - If actions occur concurrently or collaboratively, they may share a number.

5. **Use annotations**
   - Add contextual notes for assumptions, triggers, or domain definitions.
   - Place story-level annotations in the margins and detail-level notes next to
     the relevant element.

6. **Group related elements**
   - Draw a boundary around related activities when they belong together.
   - Label the group with its meaning, such as a location, phase, or optional
     area.
   - Keep actors outside groups if they span multiple clusters.

7. **Apply color sparingly**
   - Use color only to highlight important differences, requirements, or changes.
   - Include a legend whenever color conveys meaning.
   - Prefer black-and-white diagrams for simplicity unless the audience needs
     emphasis.

8. **Review and separate alternatives**
   - Avoid adding conditionals, gateways, or decision logic to the same story.
   - Model important alternatives as separate domain stories instead.
   - Validate each story with domain experts and confirm the sequence order.

---

## Decision points and guidance

- **Actor granularity:** Use role/function names, not individual people.
- **Work object repetition:** Repeat work object icons across sentences when the
  same concept is used in different actions.
- **Arrow labels:** Keep activity labels short and domain-specific.
- **Sequence clarity:** Use separate numbers for sequential steps, and share a
  number for concurrent collaborations.
- **Diagram scope:** Keep each domain story focused on one coherent scenario.
  Avoid combining too many alternatives into one canvas.

---

## Quality criteria

A strong domain story should be:

- **Readable:** stakeholders can follow the story from start to finish.
- **Explicit:** actors, activities, and work objects are all visible.
- **Domain-led:** language comes from the business domain, not technical jargon.
- **Non-branching:** the story remains a linear sequence; alternatives are
  separate stories.
- **Validated:** the sequence and objects are confirmed by domain experts.

---

## Example prompt

- "Create a domain story diagram for a customer checking out at a pharmacy,
  using actors, work objects, and numbered activities."
- "Describe how to draw a domain story that shows a pharmacist reviewing a
  prescription, creating a fulfillment request, and sending it to the patient."
- "Explain why domain stories should model alternatives as separate stories
  rather than using if/else branches."

---

## Related guidance

This skill complements other architecture diagrams such as:

- Business Process Map
- Organization Map
- Stakeholder Map
- Application/Information Concept Diagram
