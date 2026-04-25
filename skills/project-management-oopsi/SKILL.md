---
name: project-management-oopsi
description: Explains the OOPSI model for BDD discovery, starting from outcomes and working backward to scenarios and inputs.
---

# OOPSI

This skill helps AI explain the OOPSI model—Outcome, Outputs, Process, Scenarios, Inputs—as a structured way to discover and clarify feature behavior.

## When to use this skill

Use this skill when you need to:

- explain a backwards-facing discovery model that starts with outcomes,
- help teams connect feature goals to tangible outputs and workflows,
- surface scenarios and input data for complex rules,
- show how to use OOPSI alongside Example Mapping and Feature Mapping,
- improve understanding of how features produce business value.

## Outcome

Produce guidance that:

- describes each OOPSI step and how it contributes to discovery,
- explains why starting with outcomes helps avoid premature solution design,
- shows how outputs and process lead to concrete scenarios,
- recommends using input examples to validate business rules,
- emphasizes that OOPSI is a discovery tool, not a final spec.

## Instructions for the AI

1. **Describe the OOPSI steps**
   - Outcome: the problem or feature goal being discussed.
   - Outputs: the tangible results the feature must produce.
   - Process: the workflow or steps that deliver the outputs.
   - Scenarios: the edge cases and rules that should be considered.
   - Inputs: the concrete data or examples that illustrate the scenarios.

2. **Explain why this model is useful**
   - Show that it keeps the team focused on business outcomes first.
   - Explain that it helps teams avoid premature commitment to a specific implementation.
   - Emphasize that it makes the link between output and behavior explicit.

3. **Connect to BDD practices**
   - Recommend OOPSI as a complement to Example Mapping and Feature Mapping.
   - Explain that scenarios discovered in OOPSI become candidates for executable examples.
   - Suggest using tables or example cards to represent inputs.

4. **Give practical guidance**
   - Recommend using OOPSI in Three Amigos sessions or discovery workshops.
   - Advise starting with the highest-value outcome and working backward.
   - Suggest capturing outputs and scenarios visibly so the team can discuss them.

5. **Highlight outcomes and decision points**
   - Emphasize that Outputs are more than just happy paths; they include error states.
   - Advise exploring scenarios for both success and failure.
   - Recommend using inputs to answer questions like “What exactly does this output mean?”

## Decision points and guidance

- **If the team is solution-focused:** use OOPSI to shift the conversation back to outcomes.
- **If the feature is complex:** recommend OOPSI for a structured discovery path.
- **If the outputs are unclear:** prompt the team to define what success looks like first.
- **If no one can agree on the process:** suggest identifying the most important outputs and working backward.
- **If the inputs are not concrete:** recommend using tables or examples to make them tangible.

## Quality criteria

A strong OOPSI response should be:

- **outcome-oriented:** starts with what the feature must achieve,
- **value-driven:** focuses on real, observable outputs,
- **process-aware:** maps how the outputs are produced,
- **scenario-sensitive:** explores variations and edge cases,
- **data-backed:** uses inputs to clarify behavior.

## Example prompts

- "How does OOPSI help with BDD discovery?"
- "What outputs should we define before writing scenarios?"
- "How do we use OOPSI to clarify a complex feature?"

## Related guidance

Use this advice alongside:

- project-management-feature-mapping
- project-management-example-mapping
- project-management-story-workshops
- project-management-requirements-discovery-workshops
