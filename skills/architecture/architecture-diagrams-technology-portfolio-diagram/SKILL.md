---
name: architecture-diagrams-technology-portfolio-diagram
description: >
  Guide for creating a Technology Portfolio Diagram that inventories technology
  components, hardware, infrastructure software, and their hosting and vendor
  details for architecture lifecycle management.
---

# Technology Portfolio Diagram — Technology Component Catalog

A Technology Portfolio Diagram represents the organization’s hardware and
infrastructure software landscape. It supports Technology Portfolio Management by
tracking devices, system software, locations, suppliers, and purposes.

This skill is intended for architects who need a comprehensive view of the
technology environment to support lifecycle decisions, reduce clutter, and
manage infrastructure risk.

---

## When to use this skill

Use this skill when you need to:

- Catalog technology components across the organization.
- Understand whether components are physical or virtual.
- Identify on-premise versus cloud technology components.
- Track supplier and vendor relationships for infrastructure.
- Capture operating systems and versions for devices.
- Support lifecycle planning, rationalization, and risk assessment.

---

## Outcome

Produce a Technology Portfolio Diagram or catalog that shows:

- **Technology Component Name**: the identifier for the device or infrastructure
  software.
- **OS and OS Version**: the operating system running on the component.
- **Technology Location**: whether the component is on-premise or cloud-based.
- **Supplier**: the vendor or provider of the hardware or service.
- **Purpose**: the role the component plays, such as database server, file server,
  or web application host.

Include a lifecycle-aware inventory that is easy to maintain and, where useful,
show the technology services supported by each component.

---

## Step-by-step workflow

1. **Locate existing source data**
   - Start with the CMDB or configuration management records if available.
   - If no CMDB exists, work with IT operations, suppliers, or perform a physical
     inventory of server rooms and hosted services.

2. **Define the core artifacts**
   - Technology Component: the device or infrastructure software element.
   - Operating System and OS Version: the software stack on the component.
   - Technology Location: on-premise, cloud, or outsourced service.
   - Supplier: the vendor or provider of the technology.
   - Purpose: the specific technology service or role it provides.

3. **Gather data through interviews and inventory**
   - Meet with the CMDB administrator, IT operations, and service owners.
   - Ask for hardware names, virtualization status, OS details, hosting location,
     and supplier contracts.
   - Include both internal infrastructure and external cloud services.

4. **Capture the portfolio**
   - Store the inventory in the Architecture Repository or a dedicated catalog.
   - Add metadata for location, supplier, OS version, purpose, and lifecycle
     status.
   - Keep the data complete and accurate.

5. **Choose the right representation**
   - Use a tabular catalog when the portfolio includes many data columns or
     details, because diagrams can become cluttered.
   - Use a graphical diagram when the technology footprint is limited enough to
     remain readable.
   - If using a diagram, group components by technology service and location,
     and use green/technology-domain styling for devices and system software.

6. **Review and validate**
   - Confirm the portfolio with technology owners, infrastructure teams, and
     suppliers.
   - Verify OS versions, supplier relationships, and component purposes.
   - Update the repository and diagram to reflect any changes.

---

## Decision points and guidance

- **Diagram vs. catalog**: prefer a textual catalog for large portfolios and use a
  diagram only for smaller, focused technology landscapes.
- **Grouping**: organize by technology service at the top-level, then show the
  devices or system software that support those services.
- **Location boundaries**: separate on-premise and cloud components visually when
  using a diagram.
- **Supplier visibility**: include suppliers to support contract and service
  management decisions.
- **Color standard**: use technology-domain styling (green for passive technology
  elements) if modeling with ArchiMate conventions.

---

## Quality criteria

A strong Technology Portfolio Diagram should be:

- **Comprehensive**: covers all relevant hardware and infrastructure software.
- **Accurate**: reflects actual component location, OS version, and supplier data.
- **Maintainable**: stored in a repository so it can be updated as technology
  changes.
- **Readable**: avoids visual clutter by using a catalog for detailed inventories
  and diagrams for smaller landscapes.
- **Lifecycle-aware**: supports decisions about standardization, retirement, and
  consolidation.

---

## Example prompt

- "Create a Technology Portfolio Diagram that inventories all infrastructure
  components, their location, supplier, OS version, and purpose, and advise
  whether a catalog or diagram is more appropriate."
- "Validate the technology inventory with the CMDB owner and update the
  Architecture Repository."

---

## Related guidance

This skill complements architecture artifacts such as:

- Application Portfolio Diagram
- Business Process Map
- Information Map
- Organization Map

Use it when you need a technology-focused landscape that supports lifecycle and
infrastructure management decisions.
