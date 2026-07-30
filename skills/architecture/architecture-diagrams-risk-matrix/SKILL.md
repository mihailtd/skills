---
name: architecture-diagrams-risk-matrix
description: Guides the construction of a living risk matrix spreadsheet for visualizing and tracking architecture risks by likelihood, severity, mitigation, and monitoring status.
---

# Architecture Risk Matrix

This skill helps teams build a living risk matrix that captures architecture risks
in a structured, sortable spreadsheet. It focuses on tracking likelihood,
severity, mitigation plans, triggered plans, monitoring status, and lifecycle
updates.

---

## When to use this skill

Use this skill when you need to:

- visually prioritize architecture risks using a structured matrix,
- organize risk details in a sortable and filterable table,
- capture mitigation and triggered plans for each risk,
- make risk management a living artifact for planning and review,
- communicate critical risks clearly to stakeholders.

---

## Outcome

Produce a risk matrix spreadsheet that includes:

- a unique Risk ID and system/module owner,
- risk description and date identified,
- likelihood and severity ratings (Low / Medium / High),
- mitigation and triggered plans,
- monitoring status, current status, and ETA,
- optional cross-reference tracking IDs, history, and comments.

---

## Instructions for the AI

1. **Explain matrix structure**
   - Describe the need for a living table with sortable columns.
   - Recommend including Risk ID, System, Owner, Description, and Date Identified.
   - Emphasize separate Likelihood and Severity columns for visual prioritization.

2. **Define priority columns**
   - Recommend ranking Likelihood and Severity with Low/Medium/High.
   - Suggest optional numeric prefixes like `1-Low`, `2-Medium`, `3-High` to ease sorting.
   - Explain how High/High risks should surface to the top of the matrix.

3. **Capture action and tracking details**
   - Recommend columns for Mitigation Plan and Triggered Plan.
   - Add Monitoring status, current Status, and ETA.
   - Suggest linking risks to work items via Tracking ID.

4. **Include additional context**
   - Advise adding History of triggers and Comments for context.
   - Recommend using the matrix in planning sessions, not as a static document.
   - Explain that a living risk matrix should be reviewed and updated regularly.

5. **Cross-reference risk management**
   - Mention that this matrix supports the broader architecture risk management
     process.
   - Recommend using the matrix as the operational companion to risk
     quantification and mitigation strategies.

---

## Decision points and guidance

- **Sort by severity and likelihood:** make High/High risks easy to find.
- **Use owned rows:** each risk should have a named owner.
- **Keep it actionable:** mitigation and triggered plans should be clear.
- **Keep it alive:** update the matrix during planning and review cycles.
- **Use it in meetings:** filter to critical risks for stakeholder reporting.

---

## Quality criteria

A strong risk matrix should be:

- **sortable:** easy to reorder by likelihood, severity, or status,
- **actionable:** risks link to mitigation and triggered plans,
- **owned:** each risk has a responsible owner,
- **current:** entries are updated regularly,
- **visible:** used as a planning and review artifact.

---

## Example prompt

- "Show me how to structure a risk matrix spreadsheet for architecture risks."
- "Explain which columns I should include in a living risk matrix."
- "Describe how to use the matrix to identify and report High/High risks in a
  planning session."

---

## Related guidance

This skill complements:

- architecture-risk-management
- architecture-building-with-failure-in-mind