---
name: architecture-diagrams-application-portfolio-diagram
description: >
  Guide for creating an Application Portfolio Diagram that maps the enterprise
  application landscape, supports portfolio management, and highlights lifecycle,
  redundancy, hosting location, and criticality.
---

# Application Portfolio Diagram — Enterprise Application Landscape

An Application Portfolio Diagram visually represents the organization’s
software applications and systems. It supports Application Portfolio Management
by showing what applications exist, where they are hosted, who supplies them,
what capabilities they provide, and how critical they are.

This skill is for architects who need a complete, holistic view of the IT
landscape to identify redundancy, manage lifecycle decisions, and align
applications with business value.

---

## When to use this skill

Use this skill when you need to:

- Catalog all applications and software components used across the organization.
- Detect redundant or overlapping applications.
- Understand the hosting location of applications (on-premise vs. cloud).
- Track vendor relationships and supplier ownership.
- Assess application criticality using CIA scoring.
- Support rationalization, standardization, and retirement decisions.

---

## Outcome

Produce an Application Portfolio Diagram that shows:

- **Application Components**: each deployed or used software application.
- **Application Location**: whether the application is on-premise or a cloud/SaaS
  service.
- **Application Function**: the business capability each application supports.
- **Supplier/Vendor**: who provides or operates the application.
- **CIA Score**: confidentiality, integrity, and availability criticality ratings.

The diagram should be grouped by location and enhanced with color views that
highlight mission-critical versus non-critical applications.

---

## Step-by-step workflow

1. **Define the core artifacts**
   - Application Component: the modular software or system providing functionality.
   - Application Location: the hosting classification, such as on-premise or cloud.
   - Application Function: the business capability or use case it supports.
   - Supplier: the vendor or internal team responsible for the application.
   - CIA Score: a criticality rating that captures confidentiality,
     integrity, and availability impact.

2. **Gather application data**
   - Conduct interviews with business units, IT operations, and application owners.
   - Ask which applications are used for each business function and whether they
     are hosted locally or in the cloud.
   - Capture everyday employee tools as well as enterprise systems.
   - Record supplier, lifecycle status, and criticality for each application.

3. **Capture the portfolio in a repository**
   - Store the application catalog in the Architecture Repository or a dedicated
     portfolio tool.
   - Include metadata for location, function, supplier, criticality, and
     ownership.
   - Keep the catalog complete and up to date.

4. **Build the diagram**
   - Draw location boundaries for on-premise and cloud applications.
   - Place application components inside the appropriate boundary.
   - Use color views based on CIA score to emphasize mission-critical,
     semi-critical, and non-critical systems.
   - Optionally cross-map applications to information concepts, business
     processes, or capabilities.

5. **Review and validate**
   - Confirm the portfolio with application owners, IT operations, and business
     stakeholders.
   - Verify the location, supplier, function, and criticality ratings.
   - Update the repository and diagram based on feedback.

---

## Decision points and guidance

- **Application scope**: include enterprise systems, departmental applications,
  and common employee tools. The portfolio should be comprehensive, not just
  flagship systems.
- **Location boundaries**: separate on-premise and cloud/SaaS clearly in the
  diagram to show hosting risk and management domains.
- **Criticality coloring**: use CIA scores to make the diagram instantly readable
  for risk and importance.
- **Redundancy detection**: treat identical or overlapping functionality as a
  signal for rationalization.
- **Cross-mapping**: link applications to information concepts or processes when
  you need to show data ownership, compliance boundaries, or application
  dependencies.

---

## Quality criteria

A high-quality Application Portfolio Diagram should be:

- **Comprehensive**: includes all significant applications and tools.
- **Accurate**: reflects actual deployment location and ownership.
- **Readable**: uses location boundaries and color views to surface criticality.
- **Actionable**: reveals redundancies, lifecycle decisions, and rationalization
  opportunities.
- **Repository-backed**: stored alongside the application catalog in the
  Architecture Repository.

---

## Example prompt

- "Create an Application Portfolio Diagram that maps all applications by
  on-premise and cloud location, includes supplier and function metadata, and
  color-codes systems by CIA criticality."
- "Validate the application inventory with stakeholders and identify redundant
  systems for potential consolidation."

---

## Related guidance

This skill complements architecture artifacts such as:

- Information Map
- Business Process Map
- Organization Map
- Application / Information Concept cross-maps

Use it when you need a landscape view of applications that supports portfolio
management, risk assessment, and IT rationalization.
