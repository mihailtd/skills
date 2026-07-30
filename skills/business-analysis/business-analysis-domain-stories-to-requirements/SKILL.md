---
name: business-analysis-domain-stories-to-requirements
description: Guides the transformation of Domain Storytelling models into actionable requirements, user stories, and a structured backlog using user story mapping.
---

# Domain Stories to Requirements

This skill helps teams convert domain stories into structured functional
requirements and backlog items. It bridges collaborative domain modelling with
agile product planning to keep the context of business scenarios intact.

---

## When to use this skill

Use this skill when you need to:

- Turn Domain Storytelling output into actionable software requirements.
- Preserve business context while writing user stories.
- Avoid a flat backlog by organizing requirements with User Story Mapping.
- Keep requirements grounded in real scenarios rather than isolated features.
- Support continuous conversation between business experts and delivery teams.

---

## Outcome

Produce a requirements map that:

- starts from a happy-path domain story,
- establishes a backbone of high-level requirements,
- captures alternatives and edge cases as annotations or separate stories,
- extracts fine-grained requirements from detailed domain sentences,
- organizes requirements vertically beneath the backbone for prioritization.

---

## Step-by-step workflow

1. **Model the business process**
   - Start with a medium-grained to fine-grained domain story.
   - Focus on the most important scenario, typically the happy path.
   - This can be a pure or digitalized to-be story depending on your goal.

2. **Establish the backbone**
   - Walk through every activity in the domain story.
   - Ask whether the activity should be supported by the software.
   - Create a high-level kite-level requirement for supported activities.
   - Arrange these high-level requirements chronologically from left to right.

3. **Capture alternatives and exceptions**
   - Model alternative scenarios and edge cases separately if needed.
   - Use annotations on the domain story canvas for minor variations and rules.
   - Document errors, exceptions, and business rules as part of the story.

4. **Extract fine-grained requirements**
   - Walk through alternative stories and annotations.
   - For each new sentence, write one requirement at the kite or sea level.
   - Review annotations and formalize rules or exceptions as separate requirements
     when appropriate.

5. **Map to the backbone**
   - Place detailed sea-level requirements beneath the corresponding backbone
     activity.
   - Organize the map so the vertical dimension reflects detail and priority.
   - This creates a structured backlog that preserves the user journey.

---

## Writing requirements as user stories

- **Use the user story template:** `As a <type of user>, I want <goal> so that
  <reason>`. 
- **Map actor to subject:** the story’s actor should match the domain story
  subject.
- **Map activity and object to goal:** the action and work object become the story
  goal.
- **Map annotations to reason:** the why and constraints come from story
  annotations or surrounding conversation.
- **Treat stories as conversation starters:** user stories are a promise to have a
  conversation, not a complete specification.

---

## Building a backlog with User Story Mapping

- **Backbone (horizontal):** derive the backbone from the domain story’s sequence
  of activities. These high-level stories represent the main business flow.
- **Detailing (vertical):** place fine-grained stories beneath the backbone
  activities. Use alternative scenarios and annotations to create the lower
  rows.
- **Avoid the flat backlog trap:** keep requirements organized by flow and
  relationship rather than as an undifferentiated list.
- **Use the story map for planning:** discuss priorities, viable increments, and
  sprint readiness with the product owner while keeping the user context visible.

---

## Decision points and guidance

- **Happy-path first:** model the core, default scenario before exceptions.
- **Backbone from the story:** use the domain story’s activity sequence to form
  the horizontal map.
- **Separate major variations:** treat error cases as distinct stories, not
  branches in the same model.
- **Annotate minor variations:** use text notes for small deviations.
- **Map requirements vertically:** fine-grained requirements should sit beneath
  their related high-level backbone item.
- **Keep it conversational:** user stories are shorthand for ongoing discovery.

---

## Quality criteria

A strong requirements model should be:

- **Context-preserving:** each requirement links back to a domain story.
- **Structured:** requirements are organized by flow and detail.
- **Prioritized:** vertical placement reflects business value and readiness.
- **Collaborative:** the model supports discussion across roles.
- **Actionable:** the backlog is ready for refinement and implementation.
- **Traceable:** users can see how each requirement emerged from the scenario.

---

## Example prompt

- "Show me how to turn a domain story into a user story map with both kite-level
  and sea-level requirements."
- "Explain how to avoid a flat backlog by organizing requirements beneath a
  domain-story backbone."
- "Describe how to capture exceptions from domain stories as annotations and
  then decide which ones become separate requirements."

---

## Related guidance

This skill complements:

- Architecture Domain Storytelling
- Architecture Scenario-Based Modelling
- Business Analysis Scope
- Business Analysis Domain Storytelling + User Story Mapping