---
name: architecture-scenario-based-modelling
description: Guides the use of scenario-based modelling to capture multiple concrete domain stories, starting with the happy path and modeling important variations separately.
---

# Architecture Scenario-Based Modelling

This skill helps architects and teams capture domain behavior as a set of real
scenarios. It emphasizes the principle of "one diagram, one story" and uses
story-based modelling to build a representative map of the domain.

---

## When to use this skill

Use this skill when you need to:

- Explore a domain by modelling several concrete scenarios.
- Establish a shared understanding of the core, happy-path business flow.
- Separate major variations and error cases into distinct stories.
- Avoid overloading a single model with conditionals and branching logic.
- Maintain an overview of domain coverage while keeping each story readable.

---

## Outcome

Produce a scenario-based model that:

- captures the default happy path first,
- treats each major variation or error case as a separate story,
- uses concrete examples to expose domain complexity,
- supports a representative sample of scenarios rather than exhaustive coverage,
- provides an architectural anchor for connecting related scenarios.

---

## Step-by-step workflow

1. **Start with the happy path**
   - Ask domain experts for the ideal, default case.
   - Model the core process without exceptions, errors, or hypothetical "ifs".
   - Use this story to establish the main flow and the primary domain intent.

2. **Validate the core scenario**
   - Confirm the happy path with stakeholders.
   - Make sure the story explains why actors perform their activities.
   - Ensure it is clear, concrete, and meaningful.

3. **Identify major variations**
   - Ask what can go wrong or what important alternatives exist.
   - Capture each major variation as a separate domain story.
   - Start the new story at the point where it diverges from the happy path.

4. **Handle minor deviations with annotation**
   - Avoid modelling small optional steps as separate stories.
   - Instead, annotate the main story with brief notes about minor variations.
   - Keep the primary story focused and readable.

5. **Use real examples for difficult domains**
   - If a standard happy path does not exist, collect actual cases from the past.
   - Model a simple case, a moderately difficult case, and a highly complex case.
   - Use concrete examples to explore the range of domain behavior.

6. **Keep an overview**
   - Maintain a representative, not exhaustive, set of scenarios.
   - Use a high-level anchor story or map to connect the detailed cases.
   - Organize scenarios in a wiki or index to preserve discoverability.

7. **Use supporting overviews**
   - Link scenarios with a coarse-grained domain overview.
   - Use UML use case diagrams or summary maps to show which actors
     participate in which scenarios.
   - Keep the overview lightweight and focused on coverage.

---

## Decision points and guidance

- **One story per diagram:** Do not cram multiple variations into one model.
- **Happy path first:** Always model the default, 80% case before exceptions.
- **Separate major alternatives:** Important errors or alternatives deserve their
  own story.
- **Annotate minor exceptions:** Use text notes instead of extra diagrams for
  small deviations.
- **Representative sample:** Model enough scenarios to cover domain variability,
  but do not chase completeness.
- **Anchor with a big picture:** Use a high-level summary story to connect the
  detailed scenario models.

---

## Quality criteria

A strong scenario-based model should be:

- **Foundational:** the happy path is clear and well understood.
- **Modular:** major alternatives are modelled separately.
- **Concrete:** stories are based on real examples or realistic cases.
- **Manageable:** the scenario set is representative, not overwhelming.
- **Connected:** there is a clear overview that ties scenarios back to the domain.
- **Validated:** the scenarios are reviewed by domain experts.

---

## Example prompt

- "Show me how to capture a domain using scenario-based modelling, starting
  with the happy path and then creating separate stories for major variants."
- "Explain why 'one diagram, one story' is better for domain storytelling than
  trying to represent all branches in a single model."
- "Describe how to organize a set of domain scenarios so the team can see the
  overall domain map without losing readability."

---

## Related guidance

This skill complements:

- Architecture Domain Storytelling
- Business Process Map
- Stakeholder Map
- Application/Information Concept Diagram