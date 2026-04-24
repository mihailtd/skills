---
name: architecture-diagrams-business-roles-map
description: >
  Guide for creating a Business Roles Map that captures governance, accountabilities,
  business actors, and role responsibilities in an enterprise architecture context.
---

# Business Roles Map — Governance and Accountability Diagram

A Business Roles Map is an architecture artifact that goes beyond a standard
org chart. It models the actual governance structure, responsibilities, and
accountabilities of business actors, showing both formal reporting relationships
and informal role-based behavior.

This map is useful for Enterprise Architecture because it aligns roles with
capabilities, reveals informal influencers, and clarifies who is responsible and
accountable for business outcomes.

---

## When to use this skill

Use this skill when you need to:

- Capture business role accountability across the organization.
- Connect business actors to the capabilities they support.
- Represent formal hierarchy and informal responsibilities together.
- Document who is responsible, accountable, consulted, or informed for a
  business activity.
- Place the architecture function within the broader governance landscape.

---

## Outcome

Produce a Business Roles Map that clearly shows:

- Business Actors: individuals, teams, departments, or external entities that
  play a role in the business.
- Business Roles: the distinct responsibilities, functions, and accountabilities
  assigned to those actors.
- Governance relationships: who reports to whom, who owns decisions, and who
  holds secondary or informal roles.
- Role-to-capability alignment: which roles support key business capabilities.

The visual should stay clean by showing only the most significant roles, while
still capturing all role information in the Architecture Repository.

---

## Step-by-step workflow

1. **Gather formal structure**
   - Start from the existing organizational chart, governance model, or RACI
     documentation.
   - Identify departments, business units, functions, and the formal reporting
     lines that already exist.

2. **Identify business actors and roles**
   - Define business actors: the people, teams, departments, and external
     stakeholders involved in business processes.
   - Define business roles: the responsibilities, duties, and accountabilities
     each actor carries.
   - Capture secondary roles, dotted-line responsibilities, and shared account
     relationships.

3. **Conduct interviews**
   - Talk directly to employees, leaders, and process owners.
   - Ask who reports to whom, who is accountable for specific outcomes, and who
     plays additional or informal roles.
   - Avoid relying only on static org charts; capture the lived governance model.

4. **Map roles to capabilities**
   - Link each significant business role to the business capabilities it supports.
   - Use the map to identify gaps, overlaps, or understaffed capability areas.

5. **Create the visual map**
   - Plot business actors and roles in a structured diagram, similar to a detailed
     org chart.
   - Show reporting hierarchy, accountability chains, and significant secondary
     roles.
   - Keep the visual uncluttered by including only high-value roles.

6. **Validate and store**
   - Review the diagram with stakeholders and role owners.
   - Confirm the accountability and responsibility relationships are accurate.
   - Record all roles, including omitted minor ones, in the Architecture
     Repository so the map remains complete and auditable.

---

## Decision points and guidance

- **Visual detail level**: Display only the most significant roles to keep the
  map readable. Capture the full set of roles in the Architecture Repository.
- **Formal vs. informal roles**: Include informal roles if they materially affect
  governance or decision-making, but label them clearly.
- **Actor granularity**: Use teams or departments for structural clarity, and
  individuals only when specific personal accountability matters.
- **Role alignment**: Prefer role-based representation over person-based
  reporting if the same person holds multiple functions.

---

## Quality criteria

A high-quality Business Roles Map should be:

- **Governance-aware**: shows responsibility and accountability, not just
  hierarchy.
- **Role-centric**: clearly separates business actors from the roles they fill.
- **Aligned**: links business roles to capabilities or critical processes.
- **Clean**: avoids visual noise by limiting the map to the most important roles.
- **Validated**: confirmed by the people and teams represented in the map.
- **Repository-backed**: all roles are stored in the Architecture Repository,
  even if they are not shown visually.

---

## Example prompt

- "Create a Business Roles Map that shows business actors, governance
  responsibilities, and role accountability across the enterprise."
- "Capture the architecture function's place in the governance structure and
  link roles to business capabilities."

---

## Related guidance

This skill complements other architecture artifacts such as capability maps,
stakeholder maps, and organization maps. Use it when you need a role-based view
of governance that is stronger than a standard org chart.
