---
name: project-management-scenario-organization
description: Explains how to organize BDD feature files, scenarios, tags, and directories for maintainability and reporting.
---

# Scenario Organization

This skill helps AI explain how to organize BDD feature files and scenarios using feature-level grouping, directories, Background, and tags.

## When to use this skill

Use this skill when you need to:

- organize feature files in a growing BDD test suite,
- decide how to name and group features,
- explain how tags help categorize and filter scenarios,
- recommend directory structure conventions,
- keep living documentation aligned with product capabilities.

## Outcome

Produce guidance that:

- recommends organizing feature files by capability or business functionality,
- explains the difference between feature files and story-level planning artifacts,
- describes when to use Background and when to avoid it,
- shows how tags can mark components, priorities, or cross-cutting concerns,
- advises on file structure for long-lived documentation.

## Instructions for the AI

1. **Explain feature file organization**
   - Recommend storing scenarios in .feature files grouped by capability or product area.
   - Warn against organizing feature files by release or by individual stories.
   - Emphasize that feature files are living documentation of product behavior.

2. **Teach directory conventions**
   - Suggest a logical directory structure based on features or capabilities.
   - Explain that directories should help readers find behavior, not releases.
   - Mention that many teams place feature files under a test resources/features directory.

3. **Describe Background usage**
   - Recommend Background for repeated setup that is relevant to all scenarios in the file.
   - Warn against using Background to hide important variation or state.
   - Explain that Background should simplify scenarios, not obscure them.

4. **Explain tags and filtering**
   - Show that tags can be applied to features, scenarios, and example tables.
   - Describe common tag uses: component, priority, issue link, slow, ui, regression.
   - Explain that tags are useful for filtering test runs and for reporting groups.

5. **Advise on reporting and living documentation**
   - Explain that feature file structure and tags both contribute to readable reports.
   - Recommend using tags to surface cross-cutting concerns such as security or performance.
   - Suggest using feature directories alongside tags for complementary organization.

## Decision points and guidance

- **If the feature set is growing large:** group features by business capability rather than release.
- **If scenarios are mix of UI and API tests:** use tags like @ui or @api to distinguish them.
- **If the same setup repeats in every scenario:** consider Background.
- **If feature files are organized by story cards:** recommend moving to a feature-based structure.
- **If reporting needs multiple views:** use tags to support alternate groupings.

## Quality criteria

A strong scenario-organization response should be:

- **logical:** it groups tests by capability or product area,
- **maintainable:** it avoids overly granular story files,
- **filterable:** it uses tags to support meaningful test runs,
- **transparent:** it makes the behavior easy to find,
- **documentary:** it keeps feature files aligned with living documentation.

## Example prompts

- "How should we organize our Gherkin feature files?"
- "When should we use tags in BDD scenarios?"
- "What is a good directory structure for feature files?"

## Related guidance

Use this advice alongside:

- project-management-executable-specifications
- project-management-writing-scenarios
- project-management-scenario-data-tables
- project-management-story-workshops
