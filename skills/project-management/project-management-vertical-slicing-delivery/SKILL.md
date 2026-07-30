---
name: project-management-vertical-slicing-delivery
description: Guides teams to slice delivery work through the system and expose technical risk early with functional walking skeletons.
---

# Vertical Slicing Delivery

This skill helps AI recommend delivery strategies that build a feature in cross-cutting
slices, so teams can validate technical assumptions and uncover risks early.

---

## When to use this skill

Use this skill when you need to:

- plan delivery around end-to-end slices instead of isolated layers,
- expose integration and technical risks early,
- build confidence in a feature before polishing it,
- avoid delivering incomplete, untestable verticals,
- keep teams aligned on what a minimal walking skeleton should include.

---

## Outcome

Produce guidance that:

- defines a first slice as a working end-to-end skeleton,
- explains how later slices build toward releasable functionality,
- emphasizes learning from technical risk, not only user-facing value,
- recommends slice boundaries that cover key dependencies,
- helps teams measure progress using meaningful milestones.

---

## Instructions for the AI

1. **Define the first slice**
   - Describe the smallest cross-cutting set of work that produces an end-to-end path.
   - Explain that this slice is not releasable but validates technical assumptions.
   - Encourage the team to include integration points, data flow, and basic end-to-end behavior.

2. **Describe the second slice**
   - Show how the team builds toward real functionality and quality.
   - Recommend adding business logic, performance considerations, and more realistic data.
   - Emphasize learning from technical gaps exposed by the first slice.

3. **Describe the third slice**
   - Explain that the final slice refines polish, usability, and release readiness.
   - Recommend incorporating feedback from the first two slices.
   - Warn that this slice is where the feature becomes coherent and customer-facing.

4. **Prioritize risky work earlier**
   - Advise teams to put the most uncertain or costly dependencies in the first slice.
   - Highlight that early risk discovery reduces surprise and improves predictability.

5. **Use slices as learning milestones**
   - Recommend treating each slice as a checkpoint rather than a customer release.
   - Advise teams to capture lessons and adjust plans based on what they learn.

---

## Quality criteria

A strong vertical slicing recommendation should be:

- **end-to-end:** the first slice crosses the system boundary,
- **risk-focused:** early work targets the most uncertain areas,
- **iterative:** later slices build on learning,
- **practical:** every slice has a clear learning goal,
- **coherent:** the path remains aligned with the final feature vision.

---

## Example prompts

- "Help me plan a delivery approach that exposes technical risk before polish."
- "What should our first walking skeleton slice include for this feature?"
- "How can we slice a feature into meaningful development milestones without releasing incomplete work?"

---

## Related guidance

Use this tool alongside:

- project-management-opening-mid-endgame-strategy
- project-management-risk-story-mapping
- project-management-estimate-with-measurement