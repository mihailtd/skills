---
name: dev-ops-infrastructure-as-code-pulumi-operational-excellence
description: Guides teams to implement continuous improvement, monitoring, auditing, and iteration for Pulumi-managed infrastructure.
---

# Pulumi Operational Excellence and Continuous Improvement

This skill helps AI explain how to keep infrastructure healthy and evolving with monitoring, auditing, iterative refinement, and feedback loops. It includes examples for Pulumi monitoring resources, log aggregation, and compliance checks.

---

## When to use this skill

Use this skill when you need to:

- explain the feedback loop of monitoring, auditing, and improvement,
- show how to provision monitoring and alerting resources with Pulumi,
- describe audit, compliance, and drift detection practices,
- recommend iterative infrastructure refinement and incident analysis,
- connect Pulumi IaC to operational maturity and reliability.

---

## Outcome

Produce guidance that:

- defines continuous improvement for cloud infrastructure,
- shows how to enable monitoring and alerting with Pulumi,
- explains auditing, compliance, and drift detection,
- encourages postmortem learning and incremental refinement,
- positions Pulumi as the foundation for operational feedback loops.

---

## Instructions for the AI

1. **Explain continuous improvement**
   - Define continuous improvement as ongoing monitoring, auditing, and refinement.
   - Tie it to Kaizen-style incremental change and operational feedback.
   - Emphasize that infrastructure quality is continually improved, not finished once deployed.

2. **Show monitoring examples**
   - Use Pulumi to create a CloudWatch alarm or equivalent monitoring resource.

```typescript
import * as aws from "@pulumi/aws";

const cpuAlarm = new aws.cloudwatch.MetricAlarm("cpuAlarm", {
    alarmDescription: "Trigger when CPU exceeds 75% for 5 minutes",
    namespace: "AWS/EC2",
    metricName: "CPUUtilization",
    comparisonOperator: "GreaterThanThreshold",
    threshold: 75,
    evaluationPeriods: 2,
    statistic: "Average",
    dimensions: {
        InstanceId: "i-0123456789abcdef0",
    },
});
```

3. **Show audit/compliance resources**
   - Describe setting up cloud config/evaluations or policy rules.

```typescript
import * as aws from "@pulumi/aws";

const configRule = new aws.cfg.Rule("publicBucketCheck", {
    source: {
        owner: "AWS",
        sourceIdentifier: "S3_BUCKET_PUBLIC_READ_PROHIBITED",
    },
});
```

4. **Explain log aggregation and diagnostics**
   - Mention centralized logs, dashboards, and alerts.
   - Explain how Pulumi can provision log groups, streams, and export targets.

5. **Describe drift detection and state reconciliation**
   - Explain `pulumi refresh` to sync state with live infrastructure.
   - Mention periodic drift detection as part of operational hygiene.

6. **Show iterative refinement workflow**
   - Recommend reviewing metrics, updating IaC, and redeploying improvements.
   - Use pull requests and CI/CD validation to verify changes.

7. **Cover incident response and postmortems**
   - Encourage blameless reviews and action items after outages.
   - Mention documentation of lessons learned and infrastructure improvements.

8. **Advise on governance and metrics**
   - Recommend KPIs like MTTR, incident frequency, and compliance status.
   - Mention the importance of dashboards and automated alerts.

9. **Warn about common operational gaps**
   - Avoid monitoring only uptime; include performance and security signals.
   - Avoid ignoring audit findings or failing to act on alerts.
   - Avoid one-time infrastructure reviews; make them regular.

---

## Decision points and guidance

- **Do you need better visibility?** Provision monitoring and central logging.
- **Do you need compliance assurance?** Add audit rules and policy checks.
- **Is infrastructure changing often?** Use drift detection and frequent validation.
- **Are incidents recurring?** Use postmortems and incremental improvement cycles.
- **Are you scaling operations?** Add metrics-based alerts and dashboards.

---

## Quality criteria

A strong response should ensure the guidance is:

- **operational:** focused on monitoring, auditing, and improvement,
- **proactive:** encourages detecting issues before they become outages,
- **actionable:** includes resource examples for alerts and compliance,
- **iterative:** treats infrastructure as continuously evolving,
- **team-aware:** encourages collaboration and blameless learning.

---

## Example prompts

- "How do I add monitoring for Pulumi-managed infrastructure?"
- "What should I audit in a Pulumi deployment?"
- "How do I use Pulumi to provision alarms and compliance rules?"
- "What is the continuous improvement cycle for cloud infrastructure?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-cicd
- dev-ops-infrastructure-as-code-pulumi-testing
- dev-ops-infrastructure-as-code-pulumi-debugging
