---
name: business-analysis-domain-storytelling-user-story-mapping
description: Guides the combination of Domain Storytelling with User Story Mapping to connect domain scenarios with agile backlog structure and product planning.
---

# Domain Storytelling + User Story Mapping

This skill helps business analysts and product teams use Domain Storytelling as
a narrative foundation for User Story Mapping. It connects concrete domain
scenarios to a structured, prioritized backlog so teams can plan software with a
clear understanding of the business journey.

---

## When to use this skill

Use this skill when you need to:

- Translate domain scenarios into an actionable product backlog.
- Preserve the business context behind user stories and avoid a flat backlog.
- Plan a viable product increment with a shared understanding of the user journey.
- Combine narrative domain models with agile story mapping for cross-functional
  collaboration.
- Keep requirements connected to real business scenarios rather than isolated
  feature ideas.

---

## Outcome

Produce a combined model that:

- starts with medium-grained or fine-grained domain stories for the future
  process,
- creates a horizontal backbone of high-level activities from the happy path,
- captures exceptions and alternative scenarios as separate stories or annotations,
- translates detailed domain steps into vertical user story detail,
- yields a user story map organized by flow and priority.

---

## Step-by-step workflow

1. **Start with “to-be” domain stories**
   - Model the future business process using domain stories.
   - Begin with the important happy-path scenarios and focus on the default
     flow first.

2. **Build the backbone horizontally**
   - Unroll the domain story sentences into the backbone of the story map.
   - Translate each activity into a high-level user story if it should be
     supported by the software.
   - Arrange these backbone stories from left to right in chronological order.

3. **Capture alternatives and exceptions**
   - Model alternative scenarios and error cases as separate domain stories.
   - Use textual annotations for minor variations that do not require a full
     separate story.
   - Keep the main backbone focused on the core journey.

4. **Detail the requirements vertically**
   - Walk through the alternative domain stories and annotations.
   - Translate specific activities into more detailed user stories below the
     corresponding backbone activity.
   - Create a vertical hierarchy from kite-level to sea-level or fish-level
     stories.

5. **Map and prioritize**
   - Place detailed stories beneath their high-level backbone activities.
   - Prioritize stories vertically according to business value and release
     readiness.
   - Use the map to identify viable product increments and sprint candidates.

6. **Validate and iterate**
   - Review the combined map with product owners, domain experts, and developers.
   - Ensure each story is grounded in a real scenario and that priorities reflect
     the business context.
   - Revisit fine-grained domain stories as needed when a backlog item is
     selected for implementation.

---

## Decision points and guidance

- **Happy path first:** Begin with the core scenario and avoid including
  exception logic in the main backbone.
- **One diagram, one story:** Model major alternatives as separate stories, not
  branches in the same map.
- **Backbone from domain sentences:** Use the activity sequence from domain
  storytelling to define the high-level flow.
- **Annotate minor variations:** Handle small deviations with notes rather than
  extra stories.
- **Keep the map alive:** Update it as backlog priorities shift and as domain
  understanding improves.
- **Reconnect implementation:** When a story is ready to build, revisit the
  supporting domain story to preserve the original business intent.

---

## Quality criteria

A strong combined model should be:

- **Context-rich:** user stories are anchored in domain scenarios.
- **Flow-oriented:** the map shows the chronological business journey.
- **Prioritized:** vertical detail reflects business value and implementation
  readiness.
- **Balanced:** covers enough scenarios to represent the domain without
  overwhelming the team.
- **Traceable:** each backlog item can be traced back to the domain story that
  motivated it.
- **Collaborative:** reviewed and refined by domain experts, product owners, and
  delivery teams.

---

## Example prompt

- "Show me how to turn a happy-path domain story into the backbone of a user
  story map and then add exceptions as detailed stories underneath."
- "Explain how combining Domain Storytelling with User Story Mapping prevents a
  flat backlog and preserves business context."
- "Describe how to map alternative domain scenarios into vertical stories for a
  product increment."

---

## Related guidance

This skill complements:

- Architecture Domain Storytelling
- Architecture Scenario-Based Modelling
- Business Process Map
- Stakeholder Map