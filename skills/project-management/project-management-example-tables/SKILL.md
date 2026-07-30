---
name: project-management-example-tables
description: Shows how to use tables to represent cases and counterexamples when illustrating requirements in BDD discovery.
---

# Example Tables

This skill helps AI explain when and how to use tables to capture complex requirements, inputs, and expected outcomes during BDD conversations.

## When to use this skill

Use this skill when you need to:

- describe requirements involving data transformation, business rules, or eligibility,
- represent multiple cases and counterexamples in a compact format,
- compare inputs and expected outcomes clearly,
- help stakeholders validate rules without verbose text,
- support later automation with scenario tables.

## Outcome

Produce guidance that:

- explains why tables are a good format for data-centric examples,
- shows how tables help teams discuss business rules and exceptions,
- recommends when to use tables versus other techniques like cards,
- encourages capturing notes and edge cases alongside tables,
- connects tabular examples to executable scenarios.

## Instructions for the AI

1. **Explain the value of tables**
   - Show that tables make inputs, outputs, and rules easier to compare.
   - Explain that tables are especially useful for requirements with multiple fields or conditions.
   - Emphasize that tables are conversation tools, not formal documentation.

2. **Describe good table practices**
   - Recommend keeping table columns clear and focused on the key variables.
   - Advise including both positive examples and counterexamples.
   - Suggest annotating the table with rules, assumptions, or notes.

3. **Show when to use tables**
   - Use tables for eligibility rules, calculations, discount logic, status transitions, and data validation.
   - Avoid tables when the requirement is purely workflow or UX-driven; use maps or stories instead.
   - Recommend combining tables with Example Mapping or Feature Mapping when needed.

4. **Link to scenario formulation**
   - Explain that table rows often map directly to scenarios or examples.
   - Suggest using table rows as the basis for scenario outlines or decision tables.
   - Advise teams to refine the table as they learn more during discovery.

5. **Warn against overfitting**
   - Explain that tables should capture representative cases, not every possible combination.
   - Recommend keeping the table manageable and adding notes for exceptions.
   - Advise against turning the table into a rigid specification without context.

## Decision points and guidance

- **If the requirement has many input variables:** recommend a table.
- **If the requirement is a simple user journey:** use Example or Feature Mapping instead.
- **If stakeholders are confused by text alone:** use a table to make cases visible.
- **If the table is becoming too large:** split it into smaller, more focused tables.
- **If the rule needs a better example:** add a counterexample row with the reason.

## Quality criteria

A strong Example Tables response should be:

- **clear:** the table columns and rows are easy to read,
- **balanced:** it includes both valid examples and counterexamples,
- **contextual:** it includes notes or rules that explain the examples,
- **useful:** it helps stakeholders compare cases quickly,
- **bridging:** it prepares the team for later scenario automation.

## Example prompts

- "When should we use a table to describe a business rule?"
- "How can we turn this eligibility rule into a table of examples?"
- "What should a good example table contain for BDD discovery?"

## Related guidance

Use this advice alongside:

- project-management-example-mapping
- project-management-feature-mapping
- project-management-oopsi
- project-management-story-workshops
