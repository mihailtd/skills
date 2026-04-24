---
name: architecture-risk-management
description: Guides the identification, quantification, and ongoing management of architecture risks using likelihood, severity, and a maintained risk matrix.
---

# Architecture Risk Management

This skill helps teams identify and manage architecture risks by evaluating
likelihood and severity, creating a living risk matrix, and integrating risk
mitigation into planning.

---

## When to use this skill

Use this skill when you need to:

- quantify architectural and operational risks,
- prioritize mitigation work using likelihood and severity,
- maintain a living risk matrix for a development team,
- connect risk planning to graceful degradation and fault tolerance,
- review and update risk posture regularly.

---

## Outcome

Produce a risk management approach that:

- documents each risk with likelihood, severity, and ownership,
- sorts risks by business impact and probability,
- defines mitigation plans and triggered plans for each risk,
- integrates risk review into planning and team meetings,
- keeps the risk matrix current and actionable.

---

## Instructions for the AI

1. **Explain likelihood and severity**
   - Describe likelihood as the probability a risk will occur.
   - Describe severity as the business impact if the risk occurs.
   - Emphasize that mitigation should reduce at least one of these values.

2. **Classify risks clearly**
   - Use examples to illustrate high/low combinations.
   - Highlight that high likelihood/high severity risks need immediate focus.
   - Call out low likelihood/high severity risks as dangerous but easy to miss.
   - Note that high likelihood/low severity risks can often be managed by
     reducing likelihood, while low likelihood/low severity risks can usually be
     deprioritized.

3. **Recommend a risk matrix**
   - Advise maintaining one risk matrix per team.
   - List key columns: Risk ID, system/module, owner, date identified,
     description, likelihood, severity, mitigation plan, triggered plan,
     monitoring status.
   - Stress that the matrix should be a living artifact used in planning.

4. **Connect risk plans to failure behavior**
   - Suggest documenting graceful degradation and graceful backoff as triggered
     plans for relevant risks.
   - Recommend using early-fail strategies where no reasonable fallback exists.
   - Keep mitigation advice practical and aligned with architecture goals.

5. **Encourage ongoing review**
   - Advise quarterly review meetings with rotated reviewers.
   - Recommend adding new risks, retiring fixed risks, and updating rankings.
   - Emphasize consulting the matrix during planning to decide whether risk
     reduction work belongs in the current workload.

---

## Decision points and guidance

- **Use both dimensions:** risk decisions should balance likelihood and severity.
- **Prioritize the worst offenders:** high likelihood/high severity risks get the
  earliest attention.
- **Keep it local:** one risk matrix per team is more useful than a single
  company-wide spreadsheet.
- **Document fallback plans:** triggered plans should include graceful degradation
  or graceful backoff when applicable.
- **Review regularly:** risk matrices lose value when they are not updated.

---

## Quality criteria

A useful risk management model should be:

- **quantified:** every risk has a likelihood and severity rating,
- **actionable:** each risk has mitigation and triggered plans,
- **owned:** every risk has a responsible person or team,
- **visible:** the matrix is used in planning and review,
- **current:** entries are updated as the situation changes.

---

## Example prompt

- "Show me how to build a risk matrix for architecture risks with likelihood
  and severity ratings."
- "Explain when to use graceful degradation versus graceful backoff for a risk."
- "Describe how a team should review and update its risk matrix on a quarterly
  basis."

---

## Related guidance

This skill complements:

- architecture-building-with-failure-in-mind
- architecture-diagrams-risk-matrix
- project-management-sprint-planning
- project-management-adaptive-project-management
- business-analysis-scope