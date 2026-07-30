---
name: architecture-diagrams-information-map
description: >
  Guide for creating an Information Map that models key business information
  concepts, their definitions, categories, and relationships for enterprise
  architecture.
---

# Information Map — Business Information Concept Architecture Diagram

An Information Map models the organization’s business information concepts and
their relationships. It captures the corporate vocabulary that different
departments use, enabling consistent communication and better alignment between
business strategy, capabilities, and initiatives.

This skill is for architects who need to translate business language into a
structured information model and use it as a foundation for process, capability,
and regulatory analysis.

---

## When to use this skill

Use this skill when you need to:

- Define the organization’s key information concepts and business vocabulary.
- Align information concepts with products, services, capabilities, and
  strategic goals.
- Support consistent communication across business and IT teams.
- Link information concepts to regulatory requirements and business processes.
- Build a reusable information model for architecture and transformation work.

---

## Outcome

Produce an Information Map that shows:

- **Information Concepts**: the business nouns that matter to the organization.
- **Concept Categories**: whether each concept is primary (independent) or
  secondary (dependent on another concept).
- **Concept Definitions**: business definitions written without action verbs.
- **Related Concepts**: the relationships between concepts, especially primary-
  secondary dependencies.

The result can be captured as a tabular model, a graphical business object
diagram, or a combined color view that cross-maps concepts with processes,
functions, or capabilities.

---

## Step-by-step workflow

1. **Listen for business nouns**
   - Observe organizational conversations, meetings, and stakeholder interviews.
   - Capture the nouns people use naturally, such as customer, account, product,
     agreement, claim, or policy.
   - Treat each noun as a potential information concept.

2. **Review laws, regulations, and business context**
   - Identify regulatory concepts, compliance requirements, and statutory
     terms that matter to the business.
   - Include concepts that are required by law or governance.

3. **Define the core artifacts**
   - Information Concept: the concept name.
   - Concept Category: primary if it exists independently, secondary if it
     depends on another object.
   - Concept Definition: a clear, noun-focused definition without action verbs.
   - Related Concepts: bidirectional relationships to other concepts.

4. **Collect and consolidate concepts**
   - Gather concepts from interviews, documentation, and existing models.
   - Record them in the Architecture Repository with definitions, categories,
     and relationships.
   - Organize concepts into a consistent vocabulary and remove duplicates.

5. **Build the map**
   - Create a tabular map or a graphical diagram using business object symbols.
   - Show concept categories and link related concepts visually or in a
     relationship column.
   - Optionally, create color views that overlay information concepts on
     business processes or functions.

6. **Validate and maintain**
   - Review the map with business owners and subject matter experts.
   - Confirm the definitions, categories, and relationships.
   - Store the map in the Architecture Repository as a maintained source of
     truth.

---

## Decision points and guidance

- **Primary vs secondary**: classify a concept as primary when it can exist by
  itself; classify it as secondary when it requires another business object to
  exist.
- **Definition style**: keep definitions noun-centric and avoid verbs or process
  language.
- **Duplicate concepts**: consolidate synonyms and ensure one canonical concept
  name per business idea.
- **Visual representation**: use an information concept table for clarity and a
  graphical business object diagram where relationships add value.
- **Tool choice**: a formal architecture tool is preferred, because it supports
  linking concepts to capabilities, processes, and regulatory artifacts.

---

## Quality criteria

A strong Information Map should be:

- **Consistent**: uses a clear, shared vocabulary across the organization.
- **Complete**: includes the key deliverables, products, services, and regulatory
  information concepts.
- **Well-defined**: each concept has a precise, action-free definition.
- **Related**: concepts are linked through meaningful relationships.
- **Repository-backed**: stored in the Architecture Repository with metadata and
  cross-mappings.

---

## Example prompt

- "Create an Information Map for our business vocabulary, classify concepts as
  primary or secondary, define each concept clearly, and relate them to each
  other."
- "Validate the Information Map with business stakeholders and link it to our
  process and capability models."

---

## Related guidance

Use this skill alongside other architecture artifacts such as:

- Business Process Map
- Business Roles Map
- Organization Map
- Capability Map

It is especially valuable when you need a language-based model that can be
cross-mapped to processes, capabilities, and regulatory requirements.
