---
name: dev-ops-infrastructure-as-code-pulumi-cost-management
description: Guides teams to optimize cloud infrastructure costs and resource usage with Pulumi and TypeScript, including right-sizing, autoscaling, tagging, and budget-aware provisioning.
---

# Pulumi Cost Optimization and Resource Management

This skill helps AI explain how to manage cloud cost and resource efficiency using Pulumi. It covers right sizing, autoscaling, tagging, reserved capacity, waste reduction, and infrastructure budgeting with TypeScript examples.

---

## When to use this skill

Use this skill when you need to:

- explain cost optimization principles for Pulumi-managed infrastructure,
- show how to implement right-sizing and autoscaling in Pulumi,
- describe tagging and resource reporting for cost visibility,
- advise on reserved capacity, spot instances, and idle resource cleanup,
- connect resource allocation decisions to business performance and budgets.

---

## Outcome

Produce guidance that:

- defines how to optimize cost without sacrificing reliability,
- shows TypeScript examples for sizing resources and dynamic scaling,
- explains tag-based cost attribution and budget controls,
- highlights waste reduction patterns like cleanup and idle detection,
- reinforces the relationship between resource allocation and cost efficiency.

---

## Instructions for the AI

1. **Explain cost optimization fundamentals**
   - Define cost optimization as balancing performance and expense.
   - Explain that cloud cost is variable and requires continuous review.
   - Mention right-sizing, reserved capacity, spot/low-priority instances, and autoscaling.

2. **Show a right-sizing example**
   - Use Pulumi config to choose instance size by stack.

```typescript
import * as aws from "@pulumi/aws";
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();
const environment = pulumi.getStack();
const instanceType = environment === "prod" ? "t3.large" : "t3.micro";

const server = new aws.ec2.Instance("appServer", {
    instanceType,
    ami: "ami-0abcdef1234567890",
});

export const serverType = server.instanceType;
```

3. **Explain autoscaling and elastic workloads**
   - Show how scaling policies help match capacity to demand.

```typescript
import * as aws from "@pulumi/aws";

const group = new aws.autoscaling.Group("webAsg", {
    availabilityZones: ["us-west-2a", "us-west-2b"],
    desiredCapacity: 2,
    minSize: 1,
    maxSize: 5,
    launchConfiguration: new aws.ec2.LaunchConfiguration("webLc", {
        imageId: "ami-0abcdef1234567890",
        instanceType: "t3.medium",
    }).name,
});
```

4. **Show budget-aware tagging**
   - Explain cost allocation tags and automated tagging.

```typescript
const tags = {
    Project: "payments",
    Environment: pulumi.getStack(),
    CostCenter: "FIN-1234",
};

const bucket = new aws.s3.Bucket("dataBucket", {
    tags,
});
```

5. **Describe reserved capacity and spot usage**
   - Explain when to use reserved instances for stable load and spot instances for batch/workloads.
   - Mention that spot capacity is cheaper but less reliable, so use it for stateless or fault tolerant workloads.

6. **Explain cleanup and idle resource detection**
   - Explain that orphaned resources and unused snapshots waste money.
   - Recommend periodic audits and policies to remove unused resources.

7. **Show a cost-aware provisioning pattern**
   - Use Pulumi config to switch between development and production provisioning.

```typescript
const buildForCost = config.getBoolean("costOptimized") ?? false;
const desiredCapacity = buildForCost ? 1 : 3;

const appAsg = new aws.autoscaling.Group("appAsg", {
    minSize: desiredCapacity,
    maxSize: 6,
    desiredCapacity,
    // ...
});
```

8. **Mention monitoring and alerts**
   - Recommend setting budgets and spend alerts in the cloud provider.
   - Mention using cost dashboards to detect anomalies.

9. **Warn about common pitfalls**
   - Avoid using the smallest instance if it causes performance degradation.
   - Avoid overprovisioning in development just because it is easier.
   - Avoid leaving idle resources in test or staging environments.

---

## Decision points and guidance

- **Is the workload predictable?** Use reserved or committed capacity.
- **Is the workload variable?** Use autoscaling and burstable instances.
- **Does the resource need long-term retention?** Tag it and review retention policies.
- **Can the resource be deleted after use?** Automate cleanup for transient environments.
- **Should budgets gate deployments?** Yes — integrate budget alerts and cost checks into CI/CD.

---

## Quality criteria

A strong response should ensure the guidance is:

- **cost-aware:** makes cost tradeoffs visible,
- **flexible:** uses config-driven sizing and autoscaling,
- **tagged:** encourages resource metadata for reporting,
- **balanced:** avoids underprovisioning while eliminating waste,
- **governed:** integrates with budgets and spend monitoring.

---

## Example prompts

- "How do I optimize Pulumi infrastructure costs in a cloud deployment?"
- "What Pulumi patterns help right-size resources for production and dev?"
- "How can I use Pulumi to tag resources for cost allocation?"
- "What is the best way to mix reserved and spot instances with Pulumi?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-stack-configurations
- dev-ops-infrastructure-as-code-pulumi-testing
- dev-ops-infrastructure-as-code-pulumi-cicd
