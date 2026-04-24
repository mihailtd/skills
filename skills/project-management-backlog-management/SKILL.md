---
name: project-management-backlog-management
description: Instructs the AI assistant to support backlog management with NLP-based refinement, dependency identification, and prioritization guidance while preserving agile collaboration.
---

# AI-Enhanced Backlog Management Instructions

When assisting with backlog management, follow these instructions to provide
analysis, refinement recommendations, and prioritization support that help teams
keep their backlog structured and aligned with product goals.

---

## Purpose

This tool helps the AI assistant by:

- analyzing backlog items with NLP to identify themes and priorities,
- detecting dependencies and technical relationships between stories,
- recommending optimizations such as splitting, merging, or clarifying stories,
- preserving human judgment and collaboration in backlog decisions,
- improving backlog hygiene and reducing manual grooming effort.

---

## Expected outcome

As the AI, your response should help teams produce backlog management outcomes that:

- use NLP to categorize user stories into meaningful themes,
- surface hidden dependencies and shared domain entities,
- flag poorly written or overly large backlog items,
- suggest actionable refinements while leaving final decisions to the team,
- keep backlog structure transparent and explainable.

---

## Instructions for the AI

1. **Assess backlog quality with NLP**
   - Analyze story text for common patterns, keywords, and sentiment.
   - Categorize items into themes such as performance, security, or new features.
   - Detect poor structure, missing acceptance criteria, or overly broad scope.

2. **Identify dependencies and relationships**
   - Use named entity recognition and dependency parsing to extract actors,
     actions, and domain objects.
   - Highlight stories that share the same objects, data entities, or processes.
   - Recommend where hidden dependencies may require coordination or sequencing.

3. **Recommend refinement actions**
   - Suggest splitting large or complex stories into smaller, manageable pieces.
   - Identify duplicates or overlapping stories that could be merged.
   - Propose clearer acceptance criteria based on the language patterns of high-
     quality stories.

4. **Advise on prioritization**
   - Use text signals like urgency keywords and business value language to
     estimate relative priority.
   - Advise teams to validate AI-based priority suggestions against business
     context and stakeholder input.
   - Present recommendations with rationale, not as hard rules.

5. **Encourage explainability and collaboration**
   - Provide transparent reasoning for your suggestions.
   - Warn teams against overreliance on AI recommendations.
   - Emphasize that AI is an assistive tool and the product owner retains final
     accountability.

6. **Support tool integration**
   - Recommend integration with backlog systems like Jira Align or other agile
     tools when available.
   - Advise on maintaining consistent story formats and clean text data.
   - Suggest metrics such as reduced grooming time or improved prioritization
     accuracy if the team is tracking them.

---

## AI decision guidance

When generating backlog management support, keep these principles in mind:

- **Preserve human judgment:** present AI insights as recommendations, not
  decisions.
- **Demand explainability:** always explain why a story was categorized or
  prioritized a certain way.
- **Avoid black-box outputs:** if the model is uncertain, state that clearly.
- **Support collaboration:** encourage team discussion around AI findings.
- **Use clean data:** note if inconsistent or poorly formatted backlog content
  may affect the analysis.

---

## Success criteria

A useful AI backlog management response should promote a backlog that is:

- **Organized:** stories are grouped by theme and flow.
- **Clear:** items are well-written and actionable.
- **Connected:** dependencies are visible and managed.
- **Prioritized:** the team can see what matters most and why.
- **Trustworthy:** recommendations are transparent and explainable.

---

## Example prompts for the AI

- "Analyze our backlog and identify stories that should be split or clarified."
- "Help me find hidden dependencies between these user stories."
- "Recommend themes and priorities for a backlog of feature requests."

---

## Related guidance

Use this tool alongside:

- project-management-sprint-planning
- business-analysis-domain-stories-to-requirements
- architecture-domain-storytelling
- business-analysis-domain-storytelling-user-story-mapping