---
name: project-management-example-mapping
description: Teaches Example Mapping as a lightweight BDD technique for discovering rules, examples, and unknowns in story conversations.
---

# Example Mapping

This skill helps AI explain how Example Mapping works and how teams can use it to identify business rules, examples, and questions during BDD discovery.

## When to use this skill

Use this skill when you need to:

- show how to discover acceptance criteria using rules, examples, and questions,
- teach teams to keep example conversations rapid and breadth-first,
- help teams capture the right level of detail without jumping to implementation,
- explain how to keep unknowns visible and manageable,
- connect example maps to later executable specifications.

## Outcome

Produce guidance that:

- defines the four Example Mapping card types: story, rule, example, and question,
- explains how to structure a session around a single story or feature,
- shows how examples are used to validate and clarify rules,
- teaches the team to record open questions as first-class outcomes,
- describes how Example Mapping supports later scenario formulation.

## Instructions for the AI

1. **Describe the Example Mapping process**
   - Start with a User Story or feature statement on a story card.
   - Add business rules or acceptance criteria as rule cards.
   - Add concrete examples and counterexamples as example cards.
   - Add questions or uncertainties as question cards.

2. **Explain why it works**
   - Show that it makes requirements concrete without over-specifying.
   - Emphasize that rules and examples help the team build a shared mental model.
   - Explain that questions keep uncertainty visible and prevent false assumptions.

3. **Share facilitation guidance**
   - Recommend timeboxing each story session to 20–30 minutes.
   - Advise keeping cards lightweight and readable.
   - Recommend a facilitator who records and organizes the map.

4. **Teach good card habits**
   - Rule cards should capture business constraints or acceptance criteria.
   - Example cards should be phrased as “the one where...” to make them memorable.
   - Question cards should capture anything the team cannot answer immediately.
   - Encourage adding examples for both normal and edge cases.

5. **Link to tangible outcomes**
   - Explain that a finished Example Map should show the story, its rules, examples, and open questions.
   - Suggest that the session output becomes input for the Formulate phase.
   - Recommend updating the map as the team learns more.

## Decision points and guidance

- **If the conversation is drifting:** bring it back to the story and the next missing example.
- **If a rule is unclear:** ask for an example or counterexample immediately.
- **If a question cannot be answered:** add a pink card and move on.
- **If the story is too broad:** split it into smaller stories or features.
- **If the team is writing Gherkin too early:** remind them that Example Mapping is discovery, not automation.

## Quality criteria

A strong Example Mapping response should be:

- **structured:** it follows story → rules → examples → questions,
- **concrete:** it uses real examples to clarify rules,
- **visible:** it records assumptions and unknowns,
- **fast:** it keeps the session lightweight,
- **preparatory:** it prepares the team for later scenario writing.

## Example prompts

- "How do we use Example Mapping to discover acceptance criteria?"
- "What is the difference between a rule card and an example card?"
- "How should we handle open questions during Example Mapping?"

## Related guidance

Use this advice alongside:

- project-management-requirements-discovery-workshops
- project-management-feature-mapping
- project-management-story-workshops
- project-management-formulate-phase
