---
name: architecture-diagrams-application-information-concept-diagram
description: >
  Guide for creating an Application/Information Concept Diagram or matrix that
  cross-maps application components to the information concepts they expose and
  highlights security and data usage risks.
---

# Application / Information Concept Diagram — Application Data Exposure Map

An Application/Information Concept Diagram visually maps software applications
and systems to the information concepts they handle. It shows where business
information is made available in the IT landscape and helps identify systems
that require additional security protections.

This skill is for architects who need to understand which applications provide
access to sensitive information concepts and how information exposure aligns
with application risk.

---

## When to use this skill

Use this skill when you need to:

- Cross-map applications to information concepts.
- Identify which systems expose sensitive data.
- Assess application security needs based on the information they handle.
- Understand where specific information is available for consultation.
- Link application portfolio and information models for risk analysis.

---

## Outcome

Produce an Application/Information Concept Diagram or matrix that shows:

- **Application Components**: the software systems from the Application Portfolio.
- **Information Concepts**: the business information elements from the Information Map.
- **Cross-mapping**: which applications access or expose which information concepts.
- **CIA risk view**: optional color coding based on confidentiality, integrity, and
  availability criticality.

The result should make it easy to see where sensitive concepts are handled and
which systems require stronger security or review.

---

## Step-by-step workflow

1. **Reuse existing artifacts**
   - Start with the Application Portfolio Catalog for application components,
     location, function, supplier, and CIA score.
   - Start with the Information Map for information concepts and their definitions.

2. **Gather cross-map relationships**
   - Identify which applications expose, use, or provide access to each
     information concept.
   - Capture those relationships in the Architecture Repository or a matrix.

3. **Create the base layout**
   - Use the Application Portfolio Diagram structure with on-premise and cloud
     location boundaries.
   - Place application components within their hosting boundaries.

4. **Apply the information color view**
   - Assign distinct colors to critical information concepts.
   - Highlight applications based on the concepts they access, rather than
     drawing many explicit lines.
   - Use the CIA score to emphasize sensitive or high-risk applications.

5. **Assess security risk**
   - Identify applications that handle sensitive concepts like medical history,
     agreement data, or payment information.
   - Flag those systems for confidentiality, integrity, and availability controls.

6. **Review and validate**
   - Confirm the application-to-concept mappings with application owners, data
     owners, and security stakeholders.
   - Update the diagram and repository as systems or data usage changes.

---

## Decision points and guidance

- **Diagram vs matrix**: use a matrix when precise relationships are the priority and
  a diagram when you want a high-level application view with an information
  overlay.
- **Color-view design**: use colors to represent information concepts and risk
  classes to keep the diagram readable.
- **Application scope**: include both on-premise and cloud application components.
- **Security focus**: emphasize applications that handle sensitive concepts and
  use CIA score coloring where available.
- **Tool support**: prefer an architecture tool that supports cross-mapping and
  color views to reduce visual clutter.

---

## Quality criteria

A strong Application/Information Concept Diagram should be:

- **Mapped**: clearly links applications to the information concepts they expose.
- **Secure**: highlights systems that handle sensitive information.
- **Readable**: avoids excessive lines by using color views and location boundaries.
- **Validated**: confirmed by application owners and data stewards.
- **Repository-backed**: stored with the application and information models for reuse.

---

## Example prompt

- "Create an Application/Information Concept Diagram that shows which applications
  expose the Agreement, Payment, and Medical history concepts and highlights high-
  risk systems."
- "Validate the cross-map with data owners and use CIA scores to identify security
  priorities."

---

## Related guidance

This skill complements other architecture artifacts such as:

- Application Portfolio Diagram
- Information Map
- Information Concept/Business Process Diagram
- Technology/Application Function Map

Use it when you need a combined application and information view for security
and data exposure analysis.
