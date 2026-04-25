---
name: project-management-executable-specifications
description: Explains how to convert concrete examples into executable BDD scenarios and living documentation.
---

# Executable Specifications

This skill helps AI explain how teams turn concrete examples into executable scenarios using BDD, so requirements become living documentation that can be automated and validated.

## When to use this skill

Use this skill when you need to:

- explain how to convert feature examples into executable scenarios,
- show why executable specifications are valuable for business and teams,
- describe the role of feature files, scenarios, and Gherkin,
- help teams adopt living documentation practices,
- tie examples back to business goals and capabilities.

## Outcome

Produce guidance that:

- emphasizes that executable specifications are examples expressed in executable form,
- explains how examples become scenarios in feature files,
- shows how living documentation is generated from automated tests,
- highlights the communication benefit for stakeholders and teams,
- advises on when to automate scenarios and when to keep them as discovery.

## Instructions for the AI

1. **Define executable specifications**
   - Explain that they are examples written in a structured, executable format.
   - Connect them to business goals, features, and the examples discovered during discovery.
   - Stress that they are not implementation details but business-facing behavior descriptions.

2. **Describe the feature file structure**
   - Explain that feature files contain a Feature title, optional description, and one or more scenarios.
   - Recommend keeping feature titles aligned with capabilities or business functionality.
   - Mention that feature descriptions provide context and business intent.

3. **Show how examples become scenarios**
   - Explain that each concrete example can become a scenario or a scenario outline.
   - Use Given/When/Then as the standard structure for context, action, and expected result.
   - Note that scenarios should read like business examples, not test scripts.

4. **Explain living documentation**
   - Describe how automated scenarios become part of the build and produce readable reports.
   - Emphasize that this makes the requirements document live and verifiable.
   - Mention that stakeholders can read reports in terms of business behavior, not code.

5. **Advise on tool-agnostic automation**
   - Explain that the exact implementation is less important than keeping the scenarios readable.
   - Advise against letting tool syntax dominate the business language.

## Decision points and guidance

- **If the team asks whether to automate examples immediately:** recommend turning stable examples into executable specs, while keeping discovery examples lightweight until the behavior is well understood.
- **If business people are uncomfortable with syntax:** emphasize that the format is a communication tool and can be written in their native language.
- **If scenarios are too technical:** steer them back to business conditions and outcomes.
- **If the project has many examples:** recommend organizing them into feature files by capability and using scenario outlines to reduce duplication.

## Quality criteria

A strong executable specification response should be:

- **business-focused:** it starts with value, not implementation,
- **readable:** it uses clear feature titles and scenario names,
- **structured:** it explains Given/When/Then and feature files,
- **automatable:** it makes the transition to tests explicit,
- **living:** it frames scenarios as documentation kept current by execution.

## Example prompts

- "How do we turn our example discussions into executable BDD scenarios?"
- "What makes a good executable specification in a feature file?"
- "Why should business stakeholders care about living documentation?"

## Related guidance

Use this advice alongside:

- project-management-feature-description-vs-stories
- project-management-example-mapping
- project-management-story-workshops
- project-management-scenario-organization
