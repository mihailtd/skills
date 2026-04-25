---
name: project-management-real-options
description: Teaches Real Options for managing uncertainty, delaying commitment, and making better feature decisions in software projects.
---

# Real Options

This skill helps AI explain the Real Options mindset for software development, especially in BDD and agile projects where uncertainty is high and decisions should be deferred until the last responsible moment.

## When to use this skill

Use this skill when you need to:

- explain why teams should avoid committing to a solution too early,
- teach the value of keeping options open while discovery continues,
- describe how to balance flexibility with delivery deadlines,
- help teams make better release and implementation decisions,
- connect Real Options to BDD and hypothesis-driven planning.

## Outcome

Produce guidance that:

- describes options as valuable opportunities, not procrastination,
- shows that options expire and must be exercised before a delivery horizon,
- explains the effort cost of maintaining flexibility,
- encourages teams to defer decisions until they know enough,
- ties Real Options to release planning and product discovery.

## Instructions for the AI

1. **Define the Real Options principles**
   - Explain that options have value, options expire, and never commit early unless you know why.
   - Use a simple analogy such as airline ticket options or financial options.
   - Emphasize that the cost of an option is the effort required to keep flexibility.

2. **Explain why options matter in software**
   - Describe how software teams face uncertainty in requirements, architecture, and user needs.
   - Show that premature commitment can create waste if chosen solutions prove wrong.
   - Explain that keeping options open allows teams to choose the best solution after learning more.

3. **Link Real Options to BDD and backlog work**
   - Recommend preserving options while refining features and stories.
   - Note that early discovery techniques such as Impact Mapping help identify where the riskiest options lie.
   - Explain that Real Options works especially well when combined with Deliberate Discovery.

4. **Show how options expire**
   - Explain that an option expires when there is no longer enough time to implement a particular solution before delivery.
   - Advise teams to track the latest moment at which alternative approaches remain viable.
   - Recommend acting sooner if the option’s expiration approaches and information is sufficient.

5. **Recommend practical habits**
   - Suggest creating lightweight, flexible designs rather than locking in architecture too early.
   - Advise using configuration, abstraction, or modularity to preserve future choices.
   - Recommend reviewing options during backlog refinement and release planning.

## Decision points and guidance

- **If the team is uncertain about implementation:** recommend delaying the decision until more evidence is available.
- **If a choice must be made:** advise making it only when enough information exists and when the cost of delay outweighs the option value.
- **If asked about technical debt:** explain that premature commitment is a source of avoidable debt.
- **If the team is under schedule pressure:** help them distinguish between necessary commits and avoidable early lock-in.

## Quality criteria

A strong Real Options response should be:

- **disciplined:** it treats flexibility as a conscious choice,
- **time-aware:** it understands option expiry relative to delivery,
- **evidence-driven:** it commits only when enough knowledge exists,
- **pragmatic:** it balances future flexibility with current progress,
- **aligned:** it supports BDD’s hypothesis-validation mindset.

## Example prompts

- "How can our team use Real Options to avoid premature decisions?"
- "When should we commit to a technical solution for a feature?"
- "What does it mean to keep options open in backlog refinement?"

## Related guidance

Use this advice alongside:

- project-management-deliberate-discovery
- project-management-feature-description-vs-stories
- project-management-product-backlog-refinement
- project-management-story-workshops
