---
name: project-management-writing-scenarios
description: Teaches how to write effective BDD scenarios using Given/When/Then, background, pending scenarios, and clear scenario titles.
---

# Writing Scenarios

This skill helps AI explain how to write good BDD scenarios in Gherkin, including step structure, scenario naming, Background, And/But usage, and pending placeholders.

## When to use this skill

Use this skill when you need to:

- teach teams how to write readable and maintainable Gherkin scenarios,
- explain the purpose of Given, When, Then, And, But, Background, and Scenario Outline,
- help writers avoid overly detailed or script-like scenarios,
- demonstrate how to record future work with pending scenarios,
- make scenarios easy for businesspeople and developers to understand.

## Outcome

Produce guidance that:

- defines the standard scenario structure: Given, When, Then,
- explains how to write meaningful scenario titles and descriptions,
- recommends using And/But to improve readability,
- explains when to use Background and when not to,
- shows how to mark unimplemented scenarios as pending.

## Instructions for the AI

1. **Explain scenario anatomy**
   - Describe Given as preconditions or context.
   - Describe When as the action or event under test.
   - Describe Then as the expected outcome or business result.
   - Note that And and But are formatting helpers, not extra logic.

2. **Teach step focus**
   - Advise keeping each step focused on one condition or result.
   - Recommend splitting compound steps when they mix unrelated concerns.
   - Stress that steps should describe business behavior rather than implementation details.

3. **Describe scenario naming**
   - Recommend titles that summarize the unique value or business example.
   - Explain that titles are important for living documentation and reporting.
   - Suggest phrasing titles in terms of what is special about the example, not the test mechanics.

4. **Explain Background usage**
   - Recommend Background for repeated setup that is relevant to every scenario in the file.
   - Warn against using Background to hide important scenario-specific context.
   - Show that Background should make scenarios more concise, not more confusing.

5. **Cover pending scenarios**
   - Explain that empty scenarios can be used as placeholders for future work.
   - Note that they appear as pending in reports and help capture requirements early.
   - Advise using them sparingly for work that is recognized but not yet defined.

## Decision points and guidance

- **If a scenario is too long:** recommend splitting it into smaller, more focused scenarios.
- **If the scenario is too technical:** recommend using business-focused language and abstracting implementation details.
- **If the same context repeats in every scenario:** recommend using Background.
- **If the team wants to record future examples:** suggest using pending scenarios with titles only.
- **If the scenario feels like a script:** remind them that Given/When/Then should describe behavior, not instructions.

## Quality criteria

A good scenario-writing response should be:

- **clear:** the scenario structure and meaning are easy to read,
- **consistent:** step usage follows Given/When/Then semantics,
- **focused:** each scenario addresses a single example or rule,
- **concise:** it avoids unnecessary detail,
- **documentary:** titles and descriptions add business sense.

## Example prompts

- "How do we write a good Gherkin scenario?"
- "When should we use Background in a feature file?"
- "What does a pending scenario look like?"

## Related guidance

Use this advice alongside:

- project-management-executable-specifications
- project-management-scenario-data-tables
- project-management-scenario-patterns
- project-management-story-workshops
