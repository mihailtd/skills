---
name: project-management-sprint-planning
description: Instructs the AI assistant to support AI-enhanced sprint planning with data-driven estimation, capacity planning, and risk mitigation while preserving agile collaboration.
---

# AI-Enhanced Sprint Planning Instructions

When supporting sprint planning, follow these instructions to provide
recommendations, analysis, and actionable guidance that complements team
judgment.

---

## Purpose

This tool helps the AI assist teams by:

- making sprint estimates more objective and less biased,
- aligning team capacity and skills with planned work,
- identifying risks, dependencies, and impediments early,
- integrating AI insights into existing agile workflows,
- keeping the team’s judgment central while using predictive analytics.

---

## Expected outcome

As the AI, your response should help teams produce a sprint planning approach that:

- uses historical data and machine learning to inform story sizing,
- matches team skills, availability, and capacity to sprint scope,
- highlights dependencies, impediments, and risk signals before the sprint,
- positions AI recommendations as support rather than automation,
- enables continuous improvement through post-sprint validation.

---

## Instructions for the AI

1. **Validate project data quality**
   - Ask about the availability of sprint history, story details, velocity, and
     effort data.
   - Emphasize the importance of clean, normalized data for accurate predictions.
   - Request details on estimates, actual outcomes, team roles, and availability.

2. **Provide data-driven estimation guidance**
   - Suggest comparing new stories with historical data using AI or ML.
   - Recommend probabilistic effort forecasts and story point ranges.
   - Offer tips for breaking large stories into smaller subtasks.

3. **Support capacity and resource planning**
   - Encourage assessment of team skills, availability, and past performance.
   - Flag potential skill imbalances, overloaded individuals, or missing expertise.
   - Propose scope adjustments or cross-training options when necessary.

4. **Highlight risks and mitigation options**
   - Identify dependencies, impediments, and retrospective feedback that may
     affect the sprint.
   - Recommend probability and impact scoring for risks.
   - Suggest mitigation actions to include in the sprint plan.

5. **Recommend workflow integration**
   - Advise on integrating AI outputs with planning tools like Jira or Rally.
   - Stress that AI insights should be visible during team discussions.
   - Remind teams to treat AI suggestions as inputs, not final decisions.

6. **Encourage continuous learning**
   - Recommend comparing AI predictions to actual sprint results.
   - Ask for team feedback on estimate accuracy and resource planning.
   - Suggest retraining models or refining rules based on outcomes.

---

## AI decision guidance

When you generate advice, remember:

- **AI is support, not replacement:** always frame recommendations as
  decision support.
- **Choose appropriate techniques:** suggest supervised learning for estimates,
  NLP for story analysis, and predictive analytics for forecasting risks.
- **Emphasize data quality:** poor inputs produce unreliable outputs.
- **Fit the existing workflow:** integrate into current tools and processes rather
  than inventing parallel workflows.
- **Monitor performance:** recommend tracking model accuracy and updating it
  after each sprint.

---

## Success criteria

Your recommendations should enable a sprint planning process that is:

- **Accurate:** forecasted effort aligns closely with actual delivery.
- **Balanced:** capacity planning accounts for skills, availability, and
  dependencies.
- **Transparent:** AI rationale is clear and explainable.
- **Collaborative:** the team retains ownership of decisions.
- **Adaptive:** the planning approach learns from new sprint results.

---

## Example prompts for the AI

- "Help me create a data-driven estimate for this sprint using past velocity
  and story content."
- "Show how to align my team’s skill mix and availability with sprint scope."
- "Recommend how to incorporate AI risk assessment into sprint planning in
  Jira."

---

## Related guidance

Use this tool alongside:

- business-analysis-domain-stories-to-requirements
- architecture-domain-storytelling
- business-analysis-scope
- business-analysis-domain-storytelling-user-story-mapping