---
name: project-management-requirements-discovery-workshops
description: Guides AI to explain how requirements discovery workshops and Three Amigos conversations help teams illustrate features through examples and consensus.
---

# Requirements Discovery Workshops

This skill helps AI explain how teams should run requirements discovery workshops and Three Amigos conversations to clarify features, surface assumptions, and build shared understanding before implementation.

## When to use this skill

Use this skill when you need to:

- explain when a dedicated requirements discovery workshop is appropriate,
- show how the Three Amigos practice supports BDD conversations,
- compare workshop styles such as product backlog refinement, sprint planning, and focused discovery sessions,
- teach teams how to keep workshops short, productive, and outcome-oriented,
- warn against getting lost in technical details too early.

## Outcome

Produce guidance that:

- positions requirements discovery as the Illustrate phase of BDD,
- explains how concrete examples and business conversations drive acceptance criteria,
- describes the roles and benefits of a Three Amigos session,
- recommends the right meeting format for different levels of uncertainty,
- emphasizes collaboration, not handoff.

## Instructions for the AI

1. **Define the workshop purpose**
   - Explain that the goal is to understand, define, and agree on acceptance criteria.
   - Describe workshops as discovery conversations, not handoff ceremonies.
   - Stress that they should produce a shared mental model, not a finished design.

2. **Explain different workshop formats**
   - Large discovery workshops are useful for early-stage alignment, but expensive in people-hours.
   - Backlog refinement and sprint planning can work when stories are already roughly shaped.
   - Dedicated discovery workshops are best for new or complex features with high uncertainty.

3. **Describe the Three Amigos practice**
   - Recommend three perspectives: business/product, development, and testing/quality.
   - Explain that this trio uncovers edge cases, implementation risks, and business value.
   - Note that the group may include UX or domain experts depending on context.

4. **Keep workshops focused**
   - Recommend timeboxing sessions to 25–45 minutes for each story or feature.
   - Advise starting with the feature or story, then working through examples, rules, and questions.
   - Encourage capturing the outcomes, not trying to solve every detail immediately.

5. **Encourage active facilitation**
   - Suggest a facilitator who keeps the conversation grounded and records the results.
   - Recommend using a visual medium such as whiteboards, sticky notes, or online boards.
   - Advise the facilitator to log rules, examples, and open questions clearly.

6. **Warn against bad patterns**
   - Call out the anti-pattern of turning workshops into status updates or design reviews.
   - Warn against writing executable scenarios during the discovery workshop if it distracts the team.
   - Explain that examples should be captured as input to later formulation, not as the primary deliverable.

## Decision points and guidance

- **If the story is new and uncertain:** recommend a dedicated requirements discovery workshop.
- **If the story is already well understood:** recommend refining it in backlog refinement or sprint planning.
- **If the team is stuck on details:** suggest returning to examples and business outcomes.
- **If the workshop is too large or unfocused:** recommend smaller groups or split sessions.
- **If there is no shared agreement:** capture the question and decide what needs to be validated.

## Quality criteria

A strong requirements discovery workshop response should be:

- **collaborative:** it brings multiple perspectives together,
- **example-driven:** it uses concrete examples to explore the feature,
- **focused:** it keeps the discussion scoped and timeboxed,
- **transparent:** it makes assumptions and questions visible,
- **preparatory:** it produces enough clarity for subsequent Formulate work.

## Example prompts

- "How should our team prepare for a Three Amigos requirements discovery session?"
- "What is the best way to discuss an uncertain feature before sprint planning?"
- "How do we avoid wasted hours in requirements workshops?"

## Related guidance

Use this advice alongside:

- project-management-story-workshops
- project-management-rock-breakers
- project-management-feature-discovery-to-delivery
- project-management-speculate-phase
