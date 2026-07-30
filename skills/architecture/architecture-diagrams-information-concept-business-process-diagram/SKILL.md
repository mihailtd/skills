---
name: architecture-diagrams-information-concept-business-process-diagram
description: >
  Guide for creating an Information Concept/Business Process Diagram or matrix
  that cross-maps information concepts to business processes and highlights
  where information circulates across the organization.
---

# Information Concept / Business Process Diagram — Information Flow Cross-Map

An Information Concept/Business Process Diagram visually maps the organization’s
information concepts to the business processes that use them. It combines the
process view with an information color view to reveal where key information
flows through the enterprise.

This skill is for architects who need to understand how information concepts are
used by processes and what the impact is when processes or applications change.

---

## When to use this skill

Use this skill when you need to:

- Cross-map information concepts to business processes.
- Show where specific information circulates across the organization.
- Assess the impact of process change or application replacement on information
  flow.
- Highlight which processes rely on critical information concepts.
- Create a readable view of information usage without cluttering the diagram.

---

## Outcome

Produce an Information Concept/Business Process Diagram or matrix that shows:

- **Business Functions**: the high-level process groupings in the enterprise.
- **Business Processes**: the specific processes nested under functions.
- **Information Concepts**: the business vocabulary elements from the
  Information Map.
- **Cross-mapping**: which processes use or access which information concepts.

The result should make it easy to identify information usage, process impact,
and the relationship between data and operations.

---

## Step-by-step workflow

1. **Reuse existing artifacts**
   - Start with the Information Map for information concepts and their definitions.
   - Start with the Business Process Map for process functions and process owners.

2. **Define the cross-map scope**
   - Choose the set of information concepts to track (e.g., Agreement, Payment,
     Customer, License).
   - Choose the set of business functions and processes to include in the view.

3. **Map concepts to processes**
   - Identify which processes access, use, or produce each information concept.
   - Capture the relationship in the Architecture Repository or a cross-mapping
     table.

4. **Build the diagram or matrix**
   - Create the base business process diagram with functions and processes.
   - Apply a color view to highlight processes that use each information concept.
   - If using a matrix, list processes and concepts and mark their intersections.

5. **Use color views**
   - Assign distinct colors to the tracked information concepts.
   - Highlight business processes to show which concepts they use without drawing
     many explicit lines.
   - Use this approach to keep the diagram readable.

6. **Review and validate**
   - Confirm the process-concept mappings with process owners and data owners.
   - Verify that the colored visualization accurately reflects process usage.
   - Update the map if any processes or information concepts are missing.

---

## Decision points and guidance

- **Diagram vs matrix**: choose a matrix when the emphasis is on explicit cross-
  mapping and a diagram when the emphasis is on a readable process view with an
  overlay of information usage.
- **Color-view design**: use colors to represent concepts rather than drawing many
  lines, which can make the model cluttered.
- **Process selection**: include only the processes relevant to the tracked
  information concepts to preserve clarity.
- **Information concept scope**: focus on the most critical concepts for the
  analysis, especially those that drive regulatory, compliance, or application
  replacement decisions.
- **Tool support**: use an architecture tool that can render cross-mappings and
  color views to simplify the diagram.

---

## Quality criteria

A strong Information Concept/Business Process Diagram should be:

- **Readable**: the process view remains clear and uncluttered.
- **Mapped**: information concepts are clearly associated with the relevant
  processes.
- **Impactful**: highlights where changes to a process or application will affect
  information flow.
- **Validated**: reviewed by process and data owners.
- **Repository-backed**: stored with the underlying concept and process mappings
  for reuse.

---

## Example prompt

- "Create an Information Concept/Business Process Diagram that shows which
  processes use the Agreement, Payment, and License concepts, using a color view
  to keep the diagram readable."
- "Validate the information-process mappings with process owners and use the
  result to assess application replacement impact."

---

## Related guidance

This skill complements other architecture artifacts such as:

- Information Map
- Business Process Map
- Application / Information Concept Matrix
- Organization Map

Use it when you need a cross-portfolio view of how information flows through
processes and functions.
