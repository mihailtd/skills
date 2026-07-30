---
name: project-management-feature-description-vs-stories
description: Helps AI explain how to distinguish features from User Stories, describe features clearly, and organize work into manageable chunks.
---

# Features and User Stories

This skill helps AI explain how to describe and organize features, how features differ from User Stories, and how to use both to plan delivery effectively.

## When to use this skill

Use this skill when you need to:

- explain the distinction between capabilities, features, and User Stories,
- describe how features deliver business value and how stories help plan them,
- help teams avoid committing to technical solutions too early,
- recommend approaches for breaking large features into manageable work,
- show how to use feature descriptions and stories together in BDD.

## Outcome

Produce guidance that:

- defines a feature as deliverable functionality that supports a user capability,
- defines a User Story as a planning artifact that breaks work into more manageable pieces,
- explains why features are expressed in business terms,
- shows how to describe a feature using outcome-based language,
- recommends when and how to split features into stories.

## Instructions for the AI

1. **Clarify terminology**
   - Explain that a capability is what users need to do, a feature is what the software delivers, and a User Story is how the team plans the delivery.
   - Mention that some teams use different vocabularies, but the distinction helps keep discovery and delivery separate.
   - Highlight that a feature can still be broken into smaller features or stories depending on size.

2. **Describe good feature descriptions**
   - Recommend using outcome-focused language such as “In order to..., As a..., I want...” or “As a..., I want..., so that...”.
   - Emphasize starting with the business goal and stakeholder, not the implementation.
   - Suggest keeping features high-level enough to preserve options while specific enough to be understood.

3. **Explain feature decomposition**
   - Recommend breaking large features into smaller features or stories when they are too big for a single release.
   - Warn against decomposing work too early around implementation details such as screens or technical components.
   - Use examples like online renewal, notification email, and configurable incentives to show the difference.

4. **Explain User Stories**
   - Describe User Stories as story cards that help plan, estimate, and confirm work.
   - Recommend attaching acceptance criteria, priority, and rough size to stories.
   - Explain that stories can evolve as stakeholders provide feedback during implementation.

5. **Encourage feature/story feedback loops**
   - Suggest using stories as conversation starters, not as contracts.
   - Recommend showing early story work to stakeholders to discover missing requirements.
   - Note that stories can be discarded once the feature is delivered, while feature-level descriptions remain useful as product behavior documentation.

## Decision points and guidance

- **If asked whether a feature is the same as a story:** explain they are different levels of planning.
- **If asked what to do with large backlog items:** recommend slicing them into smaller features or stories.
- **If stakeholders are too solution-focused:** steer the discussion back to goals and outcomes.
- **If uncertainty remains:** suggest delaying detailed story definition until the last responsible moment.

## Quality criteria

A strong Features and User Stories response should be:

- **clear:** it explains the difference between feature-level and story-level planning,
- **business-oriented:** it emphasizes why the work is needed,
- **pragmatic:** it allows features and stories to evolve,
- **conservative:** it avoids premature solution design,
- **feedback-driven:** it uses stories to discover and validate details.

## Example prompts

- "How do we describe features so they can be broken into stories later?"
- "What is the difference between a feature and a User Story in BDD?"
- "How should we split a large feature into smaller delivery pieces?"

## Related guidance

Use this advice alongside:

- project-management-product-backlog-refinement
- project-management-story-workshops
- project-management-real-options
- project-management-deliberate-discovery
