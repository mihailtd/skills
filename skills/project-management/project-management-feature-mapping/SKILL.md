---
name: project-management-feature-mapping
description: Explains Feature Mapping as a way to explore user journeys, workflows, and complex examples in BDD discovery sessions.
---

# Feature Mapping

This skill helps AI explain how Feature Mapping breaks examples into steps, rules, and outcomes so teams can explore complex requirements and discover edge cases.

## When to use this skill

Use this skill when you need to:

- explore user journeys or workflows with more structure,
- discover variations and alternate flows for a feature,
- document examples with steps and consequences,
- surface dependencies, rules, and open questions in a single map,
- prepare complex features for BDD scenario formulation.

## Outcome

Produce guidance that:

- defines Feature Mapping card types: feature/story, rule, example, step, consequence, and question,
- explains how to start with a concrete example and work through the associated steps,
- shows how to use consequences to capture expected outcomes,
- helps teams group related flows and explore alternatives,
- links Feature Mapping to later acceptance criteria and executable scenarios.

## Instructions for the AI

1. **Explain the Feature Mapping structure**
   - Start with a feature or story as the anchor.
   - Add examples as concrete user journeys.
   - Break each example into steps that represent actions, preconditions, or workflow stages.
   - Add consequence cards to capture the expected outcome for each example.

2. **Describe how to discover rules and alternatives**
   - Ask “What else could happen here?” at each step.
   - Add new examples or flows for alternate outcomes.
   - Record rules that explain why each flow behaves differently.
   - Use questions to capture uncertainty and assumptions.

3. **Show how to handle complex domains**
   - Recommend Feature Mapping for workflows, data transformations, and policies.
   - Suggest using it when tabular examples alone are too rigid.
   - Explain how it helps teams see the larger story without losing detailed variation.

4. **Teach grouping and organization**
   - Recommend grouping related flows under higher-level rule headings.
   - Suggest visually separating successful flows from exceptions and refunds.
   - Encourage using underlined or titled rule cards to indicate themes.

5. **Link the map to BDD outcomes**
   - Explain that the map helps identify the most valuable scenarios to automate.
   - Suggest using it as the basis for writing Given/When/Then scenarios later.
   - Emphasize that the map is a discovery tool, not a final specification.

## Decision points and guidance

- **If the feature is too simple:** Example Mapping may be enough.
- **If the feature involves journeys or many outcomes:** recommend Feature Mapping.
- **If the team is losing sight of the user:** bring the conversation back to concrete steps.
- **If questions accumulate:** add them as pink cards and decide who will answer them.
- **If the map becomes too messy:** group related examples or split the feature into smaller maps.

## Quality criteria

A strong Feature Mapping response should be:

- **journey-oriented:** it follows user steps through the feature,
- **outcome-focused:** it captures consequences explicitly,
- **variant-aware:** it explores alternate flows and edge cases,
- **assumption-conscious:** it keeps questions visible,
- **handoff-ready:** it prepares the team for later scenario formulation.

## Example prompts

- "How should we map a complex booking workflow with BDD?"
- "What kinds of examples should we use in Feature Mapping?"
- "How do we turn Feature Mapping into acceptance criteria?"

## Related guidance

Use this advice alongside:

- project-management-example-mapping
- project-management-requirements-discovery-workshops
- project-management-story-workshops
- project-management-oopsi
