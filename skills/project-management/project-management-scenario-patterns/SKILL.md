---
name: project-management-scenario-patterns
description: Teaches good and bad BDD scenario patterns, including declarative style, independence, persona usage, and anti-patterns.
---

# Scenario Patterns

This skill helps AI explain expressive BDD scenario patterns and the common anti-patterns to avoid when writing Gherkin scenarios.

## When to use this skill

Use this skill when you need to:

- explain what good Gherkin looks like,
- teach the difference between declarative and imperative scenarios,
- identify common anti-patterns such as test scripts or chained scenarios,
- recommend how to keep scenarios independent and focused,
- help the team use actors or personas effectively.

## Outcome

Produce guidance that:

- defines declarative scenarios that describe behavior, not implementation,
- shows why scenarios should do one business rule well,
- explains how personas and meaningful actors improve readability,
- warns against manual-test-script style and brittle chained scenarios,
- encourages hiding incidental details and focusing on essentials.

## Instructions for the AI

1. **Describe the declarative style**
   - Explain that good scenarios describe what the user wants to achieve.
   - Contrast this with imperative scenarios that list clicks and field interactions.
   - Recommend step wording that focuses on business behavior.

2. **Teach the one-rule principle**
   - Recommend scenarios focus on a single rule or example.
   - Suggest breaking complex journeys into multiple independent scenarios.
   - Explain that focused scenarios are easier to troubleshoot and maintain.

3. **Explain independence**
   - Recommend that each scenario set up its own initial state.
   - Warn against scenario sequences that depend on earlier scenarios.
   - Note that independent scenarios produce more reliable test suites.

4. **Use meaningful actors and personas**
   - Recommend using named actors or personas instead of generic “I” or “user”.
   - Explain that personas help surface what details are important.
   - Suggest soap opera personas when formal UX personas are unavailable.

5. **Cover anti-patterns**
   - Identify test-script scenarios that use “verify” or “check” language.
   - Warn against long end-to-end scenarios that do too much.
   - Warn against incidental detail that distracts from the business rule.

## Decision points and guidance

- **If a scenario reads like instructions:** recommend making it more declarative.
- **If it tests multiple rules at once:** recommend splitting it into smaller scenarios.
- **If a scenario depends on previous tests:** recommend making it independent.
- **If actors are generic:** recommend using meaningful names or personas.
- **If the scenario is fragile:** look for UI or ordering detail that can be hidden behind stable business language.

## Quality criteria

A strong scenario-patterns response should be:

- **declarative:** it describes outcomes, not mechanics,
- **focused:** it tests one rule or example well,
- **independent:** it can run in isolation,
- **actor-aware:** it uses meaningful stakeholders,
- **anti-pattern-aware:** it avoids test-script and brittle flows.

## Example prompts

- "What makes a good BDD scenario?"
- "How do we avoid writing scenario test scripts?"
- "When should scenarios be independent?"

## Related guidance

Use this advice alongside:

- project-management-writing-scenarios
- project-management-scenario-organization
- project-management-scenario-data-tables
- project-management-feature-mapping
