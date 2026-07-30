---
name: business-analysis-scope
description: Guides the definition of story scope in Domain Storytelling using granularity, point-in-time, and domain purity to explore and drill into domains effectively.
---

# Business Analysis: Domain Storytelling Scope

This skill helps teams define the right scope for Domain Storytelling workshops.
It explains how to choose granularity, point-in-time, and domain purity so the
storytelling effort produces useful architectural insight and a shared domain
understanding.

---

## When to use this skill

Use this skill when you need to:

- Establish the right level of detail for a Domain Storytelling session.
- Decide whether to model the current state or the future state.
- Keep domain stories consistent and avoid mixing coarse and fine granularity.
- Separate business process modelling from existing software implementation.
- Explore a new domain and then drill down into subdomains.

---

## Outcome

Produce a scoped Domain Storytelling model that:

- uses a consistent level of granularity within each story,
- selects an appropriate point in time (`as-is` or `to-be`),
- chooses a domain purity level (`pure` or `digitalized`) based on the objective,
- provides a high-level overview for a new domain or a detailed view for a
  subdomain,
- supports architectural analysis, bounded-context discovery, and future
  software planning.

---

## Step-by-step workflow

1. **Choose granularity consistently**
   - Determine whether the story should be coarse-grained or fine-grained.
   - Use the ocean metaphor: sea-level for fine-grained work, kite/cloud for
     coarse summaries, and fish/clam for very low-level detail.
   - Keep a single story at one level; mixing levels usually indicates missing
     knowledge holders or unclear scope.

2. **Select the point in time**
   - Decide whether the story captures the current state (`as-is`) or the
     intended future state (`to-be`).
   - Use `as-is` stories to understand the existing domain and identify pain
     points.
   - Use `to-be` stories to design new software-supported processes.

3. **Define domain purity**
   - Choose `pure` for stories that focus on the business domain without existing
     software systems.
   - Use `digitalized` when the story must include software-enabled interactions.
   - For early domain exploration, prefer pure stories to avoid legacy technical
     bias.

4. **Explore a new domain with coarse-grained, pure, as-is scope**
   - Start with a broad story that omits existing software and captures the
     naked business process.
   - Use the customer viewpoint to model an end-to-end flow.
   - Invite a wide set of storytellers from different departments or organizations
     to cover the landscape.
   - Use this story as a high-level anchor for later detailed work.

5. **Identify architectural boundaries and subdomains**
   - Analyze the coarse-grained story to find natural domain partitions.
   - Use the story to spot potential bounded contexts, service boundaries, and
     areas that deserve further investigation.

6. **Drill down into subdomains with fine-grained, pure, as-is stories**
   - Focus on a narrower subdomain with a smaller, more specialized group.
   - Model the fine-grained work in detail while keeping the domain pure.
   - Add a leading and trailing sentence to capture handoffs and interfaces with
     adjacent subdomains.

7. **Use the scoped stories as a baseline**
   - Compare fine-grained `as-is` stories with future `to-be` stories to make
     changes visible.
   - Use the pure domain model as a foundation for code-level modelling.
   - Revisit scope when the team shifts from exploration to requirement definition.

---

## Decision points and guidance

- **Granularity matters:** Choose a single level for each story; do not mix
  coarse and fine detail in the same model.
- **As-is vs to-be:** Use current-state stories to understand what exists and
  future-state stories to design what should change.
- **Domain purity first:** Begin with pure, non-technical stories to avoid
  inheriting legacy implementation constraints.
- **Wide exploration early:** Use broad, high-level stories to establish context
  and identify subdomains before drilling down.
- **Focused drill-down:** Use fine-grained stories for detailed process analysis
  and implementation-ready domain modelling.

---

## Quality criteria

A well-scoped Domain Storytelling model should be:

- **Consistent:** the story maintains one level of granularity.
- **Clear:** the point in time and purity level are explicit.
- **Architectural:** it reveals boundaries, subdomains, or areas of change.
- **Relevant:** it matches the objective of exploration, design, or analysis.
- **Collaborative:** the scope is agreed by the workshop participants.

---

## Example prompt

- "Explain how to choose the right scope for a Domain Storytelling workshop when
  exploring a new business domain."
- "Show me how to start with a coarse-grained, pure, as-is story and then drill
  down into fine-grained subdomain stories."
- "Describe the difference between sea-level and kite-level granularity in
  Domain Storytelling and why each story should stay consistent."

---

## Related guidance

This skill complements:

- Architecture Domain Storytelling
- Architecture Scenario-Based Modelling
- Business Process Map
- Business Analysis Domain Storytelling + User Story Mapping