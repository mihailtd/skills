---
name: architecture-diagrams-business-process-map
description: >
  Guide for creating a Business Process Map / Business Process Diagram that
  links business processes to their functions, owners, and architectural impact.
---

# Business Process Map — Process-to-Function Architecture Diagram

A Business Process Map captures the processes used to deliver an end product or
service. It links those processes to their higher-level business functions and
process owners, providing a foundation for optimization, automation,
compliance, and transformation.

This skill is intended for Enterprise Architecture work where process clarity and
alignment between business operations and technology are critical.

---

## When to use this skill

Use this skill when you need to:

- Document the organization’s business processes and how they group under
  business functions.
- Identify process owners and accountability for key process steps.
- Reveal duplicate or overlapping processes across functions.
- Link business processes to supporting applications, systems, or capabilities.
- Prepare for automation, compliance review, or business transformation.

---

## Outcome

Produce a Business Process Map or Business Process Diagram that shows:

- **Business Functions**: high-level areas of capability or work.
- **Business Processes**: the specific sequences of steps that achieve a defined
  result.
- **Process Owners**: the business actor, team, or role accountable for the
  process.
- **Function-to-process grouping**: which processes belong to which functions.

The visual should be clean and easy to read, ideally avoiding duplicate process
entries across multiple functions. The full process inventory should be stored
in the Architecture Repository for traceability.

---

## Step-by-step workflow

1. **Review existing documentation**
   - Start with department descriptions, intranet pages, process catalogs, and
     existing business capability models.
   - Extract high-level business functions from these sources.

2. **Identify the artifacts**
   - Define Business Functions: collections of behaviors or work the organization
     performs (e.g., Order Management, Payment Processing, Talent Acquisition).
   - Define Business Processes: specific sequences of work that produce outcomes
     (e.g., process customer orders, onboard new employees).
   - Define Business Actors / Roles: the team, department, or role responsible
     for the process.

3. **Conduct departmental interviews**
   - Talk directly to the people doing the work.
   - Ask what actions they perform, whether the work is manual or system-driven,
     who owns the actions, and how the function is executed.
   - Capture the real processes, not only the formal names from documents.

4. **Consolidate and clean the process list**
   - Record all process candidates in the Architecture Repository.
   - Map each process to a single primary business function and process owner.
   - Remove duplicate or overlapping process definitions where they represent
     the same work.

5. **Create the Business Process Map**
   - Group processes under their business functions visually.
   - Annotate each process with its owner and any supporting application or
     capability links.
   - Use the map to highlight opportunities for optimization, automation, or
     compliance control.

6. **Validate and capture**
   - Review the diagram with process owners and function leaders.
   - Confirm the ownership, function grouping, and process scope.
   - Store the final process map and supporting artifacts in the Architecture
     Repository for future reference.

---

## Decision points and guidance

- **Process placement**: each process should belong to one primary business
  function. If a process appears in multiple functions, treat that as a warning
  sign of duplication or overlap.
- **Visual cleanliness**: keep the diagram focused on high-value processes and
  functions. Omit low-impact processes from the visual if necessary, but record
  them in the repository.
- **Ownership granularity**: use teams or roles for ownership unless a specific
  individual is required for accountability.
- **Tool choice**: use a dedicated architecture tool or repository rather than a
  spreadsheet or generic drawing tool, so the map can be maintained and linked
  to other architecture artifacts.

---

## Quality criteria

A strong Business Process Map should be:

- **Complete**: covers the major business functions and the processes they own.
- **Accurate**: reflects the actual work performed, not just how it is supposed to
  work.
- **Clean**: avoids duplicate process entries across functions.
- **Actionable**: highlights automation, optimization, or compliance opportunities.
- **Repository-backed**: captured in the Architecture Repository with process,
  function, and owner metadata.

---

## Example prompt

- "Create a Business Process Map that groups processes under their business
  functions, identifies process owners, and highlights any overlapping process
  definitions."
- "Validate the Business Process Diagram with stakeholders and store the
  resulting processes in the Architecture Repository."

---

## Related guidance

This skill complements architecture artifacts such as:

- Business Roles Map
- Organization Map
- Business Capability Map
- Function / Process Matrix

Use it when you need a process-oriented view of the organization that supports
operational alignment, automation, and transformation planning.
