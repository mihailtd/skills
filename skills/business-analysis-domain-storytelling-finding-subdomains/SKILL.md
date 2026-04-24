---
name: business-analysis-domain-storytelling-finding-subdomains
description: Guides the use of Domain Storytelling heuristics to identify subdomains, define bounded contexts, and align team boundaries with software architecture.
---

# Finding Subdomains with Domain Storytelling

This skill helps analysts and architects use Domain Storytelling to split a
complex domain into manageable subdomains. It shows how to identify cohesive
activity clusters, define bounded contexts, and align team structure with
desired software architecture.

---

## When to use this skill

Use this skill when you need to:

- Break a large problem space into smaller, self-contained subdomains.
- Avoid a monolithic domain model that becomes a "big ball of mud."
- Discover domain boundaries from real business scenarios.
- Transition from problem-space subdomains to solution-space bounded contexts.
- Organize teams around architectural boundaries rather than layers.

---

## Outcome

Produce a scoped domain model that:

- captures meaningful scenarios as coarse-grained, as-is, pure domain stories,
- reveals cohesive activity clusters that suggest subdomain boundaries,
- uses bounded context criteria to transition subdomains into design intent,
- supports a context map that shows relationships between bounded contexts,
- guides team boundary decisions using the inverse Conway maneuver.

---

## Step-by-step workflow

1. **Find meaningful scenarios**
   - Identify a set of domain stories that cover essential parts of the domain.
   - Use concrete, end-to-end business examples rather than abstract flowcharts.

2. **Model the scenarios as pure, coarse-grained stories**
   - Work in an as-is, pure scope to remove existing software complexity.
   - Keep the stories coarse-grained to establish broad bearings in the problem
     space.
   - Use these stories as a high-level anchor for further analysis.

3. **Locate cohesive activity groups**
   - Inspect each story for activities that belong together.
   - Look for a sequence of related tasks that produces a meaningful result.
   - Keep actors outside of candidate subdomain clusters to show they span
     multiple areas.

4. **Cluster activities into subdomain candidates**
   - Group activities that share a clear outcome or cohesive purpose.
   - Use borders or labeled groups to make candidate subdomains visible.
   - Treat each cluster as a potential subdomain boundary.

5. **Apply subdomain boundary heuristics**
   - Identify one-way handoffs where actor A passes a work object to actor B and
     receives no return information.
   - Spot activities that produce a result independently and hand it off.
   - Look for different triggers, such as request-based versus time-based
     initiation.
   - Notice when the same work object has different meanings for different actors.
   - Flag cases where a result is created for a purpose outside the current
     story.

6. **Transition to bounded contexts**
   - Evaluate candidate subdomains against bounded context criteria:
     business value, domain alignment, sociopolitical factors, technical needs,
     and user experience.
   - Use this evaluation to turn problem-space subdomains into solution-space
     bounded contexts.
   - Create a context map to visualize how the bounded contexts relate and
     interact.

7. **Align team boundaries with contexts**
   - Avoid horizontal layer-based teams; prefer vertical, context-aligned teams.
   - Assign each bounded context exclusively to one team.
   - Allow a team to own multiple bounded contexts if needed.
   - Split teams only when size or efficiency demands it, and then reassign
     contexts accordingly.

---

## Decision points and guidance

- **Pure stories first:** Use domain-pure, as-is stories to reveal the naked problem
  space before adding software constraints.
- **Coarse-grained exploration:** Start broad to understand the landscape, then
  drill down later.
- **Actor placement:** Keep actors outside groups to show they cross boundaries.
- **Subdomain signals:** One-way handoffs, independent results, trigger changes,
  and language differences are strong boundary clues.
- **Bounded context criteria:** Balance business value, domain, sociopolitical,
  technical, and UX concerns when choosing final boundaries.
- **Team structure:** Use the inverse Conway maneuver to align communication and
  architecture.

---

## Quality criteria

A strong subdomain discovery model should be:

- **Cohesive:** boundaries separate activities that belong together.
- **Clear:** clusters are visible and well-labeled.
- **Domain-driven:** boundaries emerge from business behavior, not technical
  convenience.
- **Architectural:** the analysis supports bounded context design.
- **Organizationally aligned:** team boundaries reflect the chosen contexts.
- **Reviewable:** domain experts and architects agree on the boundaries.

---

## Example prompt

- "Show me how to find subdomains in a complex business domain using Domain
  Storytelling and pure, coarse-grained stories."
- "Explain how one-way handoffs and different triggers indicate domain
  boundaries."
- "Describe how to move from subdomains to bounded contexts and then align teams
  with those contexts."

---

## Related guidance

This skill complements:

- Architecture Domain Storytelling
- Business Analysis Scope
- Architecture Scenario-Based Modelling
- Business Analysis Domain Storytelling + User Story Mapping