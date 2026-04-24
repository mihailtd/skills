---
name: architecture-diagrams-organization-map

description: >
  Guide for creating an Organization Map diagram that visualizes internal
  business units, external partners, and the working relationships between them.
  Use this skill when you need an enterprise ecosystem view instead of a simple
  hierarchical org chart.
---

# Organization Map — Internal and External Interaction Diagram

An Organization Map is a relationship-focused architecture artifact. It
captures how the enterprise actually operates by showing key internal entities,
external stakeholders, partners, and suppliers and the real working links
between them.

This is not a traditional reporting hierarchy. Instead, it is a network or web
with the organization at the center and the primary collaborations radiating
outward.

---

## When to use this skill

Use this skill when you need to:

- Visualize the enterprise ecosystem as a collaboration network.
- Show how departments and divisions interact with each other.
- Reveal which external partners, suppliers, and stakeholder groups support
  those interactions.
- Surface real operational relationships that are not visible in a normal org
  chart.

---

## Outcome

Produce a diagram that clearly answers:

- Which internal units are core to the organization?
- Who do they work with internally?
- Which external partners or suppliers do they depend on?
- What are the actual working relationships, not just formal reporting lines?

The finished model should be ready for inclusion in an Architecture Repository
as a reusable enterprise ecosystem artifact.

---

## Step-by-step workflow

1. **Collect the baseline org structure**
   - Start with the existing organizational chart or governance model.
   - Identify internal divisions, departments, business units, and board-level
     sponsors.
   - Capture the formal structure first so you can compare it with actual
     collaboration.

2. **Interview the units**
   - Talk to the teams, unit leaders, and business owners.
   - Ask about their day-to-day collaborations, shared workflows, and handoffs.
   - Identify the external partners, suppliers, vendors, and stakeholder groups
     they rely on.
   - Capture both regular interactions and critical one-off relationships.

3. **Build the network diagram**
   - Place the core organization in the center.
   - Add internal entities (divisions, departments, units) around the center.
   - Add external entities (partners, suppliers, stakeholder groups) beyond the
     internal units.
   - Connect entities with relationship lines representing real working links.

4. **Label the relationship types**
   - Use line labels or visual styling to show the nature of the interaction:
     collaboration, data exchange, service delivery, compliance oversight,
     escalation path, etc.
   - Distinguish internal collaboration from external dependency.

5. **Validate with stakeholders**
   - Review the diagram with the teams and leaders represented on the map.
   - Confirm it reflects real working relationships, not only structure.
   - Adjust the model where stakeholders identify missing connections or
     inaccurate interaction patterns.

6. **Store the result**
   - Save the completed Organization Map in the Architecture Repository.
   - Attach metadata or notes that explain the interaction types, owners, and
     update cadence.
   - Keep it versioned so future architecture initiatives can reuse it.

---

## Decision points and guidance

- **Internal vs. external placement**: keep the core organization in the center,
  internal units closest to center, and external partners/suppliers further out.
- **Relationship lines**: draw a line for a real interaction only when it is used
  in practice, not only when a governance document says it exists.
- **Level of detail**: include the units that matter to the ecosystem being
  modeled. Do not add every sub-team if it obscures the main relationships.
- **Stakeholder coverage**: ensure you include both business-facing and
  operational stakeholders for a full ecosystem view.

---

## Quality criteria

A strong Organization Map should be:

- **Accurate**: reflects real-day collaboration relationships.
- **Actionable**: shows who depends on whom and why.
- **Balanced**: includes both internal units and external parties.
- **Readable**: uses a network/web layout rather than a strict top-down tree.
- **Validated**: reviewed and confirmed by the stakeholders shown.
- **Reusable**: captured in the Architecture Repository with context for future
  use.

---

## Example prompt

- "Create an Organization Map for the enterprise, placing the corporate center
  in the middle, showing internal divisions, board-level oversight, key
  suppliers, and external partners, and labeling actual working relationships."
- "Validate the Organization Map with stakeholders and store the final version
  in the architecture repository."

---

## Related guidance

Use this skill whenever you need a richer enterprise view than a normal org
chart. It complements architecture artifacts such as capability maps,
application landscapes, and stakeholder analyses.
