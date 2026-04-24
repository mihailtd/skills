---
name: architecture-diagrams-technology-application-function-map
description: >
  Guide for creating a Technology/Application Function Map that aligns underlying
  technology services with the application functions they support.
---

# Technology/Application Function Map — Technology Service to Application Function Alignment

A Technology/Application Function Map is a tabular architectural deliverable that
links technology services to application functions. It reveals whether the technology
purpose actually matches the application capability it supports, and it helps
architects detect redundant environments providing identical services.

This skill is useful when you need a practical, data-driven connection between the
Technology Portfolio Catalog and the Application Portfolio Catalog.

---

## When to use this skill

Use this skill when you need to:

- Align technology services with application functions.
- Expose redundant technology environments supporting the same application capability.
- Validate that infrastructure purpose supports business application needs.
- Compare technology portfolio and application portfolio data in a single view.
- Support rationalization and consolidation conversations.

---

## Outcome

Produce a Technology/Application Function Map with columns for:

- **Technology Component**: the physical or virtual device from the technology portfolio.
- **Technology Service**: the action-oriented service provided by the component.
- **Application Name**: the application component from the application portfolio.
- **Application Function**: the business capability the application supports.

The map should make it easy to see whether multiple devices provide the same technology
service for applications that deliver the same function.

---

## Step-by-step workflow

1. **Reuse existing catalogs**
   - Extract technology component names and purposes from the Technology Portfolio Catalog.
   - Extract application names and functions from the Application Portfolio Catalog.

2. **Translate purpose into service**
   - Rewrite the technology component purpose as a technology service.
   - For example, convert "database server" into "database service" or "web application host" into "web service."

3. **Build the tabular map**
   - Create a table with four adjacent columns: Technology Component, Technology Service,
     Application Name, and Application Function.
   - Align technology service next to application function for direct comparison.

4. **Populate relationships**
   - Record which technology component supports which application.
   - Show the supporting technology service and the application function it enables.

5. **Analyze for redundancy**
   - Look for rows where multiple technology components provide the same service.
   - Identify whether the supported applications perform the same function.
   - Highlight parallel environments or duplicate services for review.

6. **Review and store**
   - Validate the map with application owners, technology owners, and infrastructure teams.
   - Keep the map in the Architecture Repository as a reconciled view between technology and application portfolios.

---

## Decision points and guidance

- **Table format**: prefer a tabular map over a graphical diagram for this deliverable.
- **Service naming**: use service-oriented wording for technology purposes, not device-centric terms.
- **Redundancy focus**: treat identical technology service/application function pairs as
  candidates for rationalization.
- **Source of truth**: use the established portfolio catalogs rather than gathering new
  raw data for this map.

---

## Quality criteria

A strong Technology/Application Function Map should be:

- **Aligned**: directly connects technology services to application functions.
- **Clear**: uses a compact tabular layout with adjacent service/function columns.
- **Insightful**: surfaces redundant services and duplicate application capabilities.
- **Validated**: reviewed by both technology and application stakeholders.
- **Repository-backed**: stored as a reconciled artifact that links the two portfolios.

---

## Example prompt

- "Create a Technology/Application Function Map that pairs technology components and their services with application names and functions, and highlight redundant service/function alignments."
- "Use the technology portfolio and application portfolio catalogs to build a reconciled table and validate it with infrastructure and application owners."

---

## Related guidance

This skill works best alongside:

- Application Portfolio Diagram
- Technology Portfolio Diagram
- Business Process Map
- Information Map

Use it when you need a cross-portfolio view that evaluates whether technology service delivery matches application function needs.
