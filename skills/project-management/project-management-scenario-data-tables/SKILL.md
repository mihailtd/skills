---
name: project-management-scenario-data-tables
description: Guides teams to use tables in BDD scenarios to avoid duplication, express data clearly, and drive scenario outlines.
---

# Scenario Data Tables

This skill helps AI explain how to use tables in BDD scenarios, including embedded data, vertical record tables, examples tables, and Scenario Outline.

## When to use this skill

Use this skill when you need to:

- represent similar examples more concisely,
- avoid duplication in Given and Then steps,
- express test data or expected outcomes clearly,
- write table-driven scenarios using Scenario Outline,
- make feature files easier to read and maintain.

## Outcome

Produce guidance that:

- shows when tables are a better choice than repeated text,
- explains embedded tables in Given/Then and vertical record tables,
- describes Scenario Outline and Examples for data-driven cases,
- emphasizes using tables for business input and expected outcomes,
- warns against including incidental details in tables.

## Instructions for the AI

1. **Explain embedded step tables**
   - Describe how Given or Then steps can include a table of data values.
   - Show how table headers help make the table self-explanatory.
   - Give examples for account balances, flight data, or hotel records.

2. **Teach vertical record tables**
   - Explain that a single record can be represented with field names in the first column.
   - Recommend this format for individual objects with many fields.
   - Explain how it keeps the scenario concise and readable.

3. **Describe Scenario Outline and Examples**
   - Explain that Scenario Outline templates a scenario for multiple data sets.
   - Describe how the Examples table drives multiple scenario executions.
   - Recommend using it when a set of related cases differ only by input values.

4. **Advise on table design**
   - Recommend including only relevant columns.
   - Warn against tables with incidental or redundant data.
   - Suggest using a Notes column for business context when needed.

5. **Connect tables to living documentation**
   - Explain that tables produce readable reports and make examples easy to scan.
   - Note that table-driven scenarios can document boundary cases clearly.
   - Recommend using table headers that describe the business intent of each column.

## Decision points and guidance

- **If scenarios repeat the same data structure:** use a table.
- **If a scenario has many similar examples:** use Scenario Outline with Examples.
- **If a table contains irrelevant fields:** remove them or hide them in step implementation.
- **If the table is too wide:** break it into smaller focused tables or separate scenarios.
- **If the output is a list of values:** use a simple vertical table or list.

## Quality criteria

A strong scenario data table response should be:

- **concise:** it reduces duplication,
- **clear:** headers and values are easy to interpret,
- **focused:** it includes only business-relevant data,
- **maintainable:** it makes updates easier,
- **report-friendly:** it supports readable living documentation.

## Example prompts

- "How can we use tables in our Gherkin scenarios?"
- "When should I use Scenario Outline and Examples?"
- "How do I write a data-driven scenario with a table?"

## Related guidance

Use this advice alongside:

- project-management-executable-specifications
- project-management-writing-scenarios
- project-management-scenario-organization
- project-management-example-tables
