---
name: dev-ops-infrastructure-as-code-pulumi-stack-configurations
description: Guides teams to build adaptable Pulumi stacks using dynamic stack configuration, conditional resources, reusable components, and stack-aware behaviors.
---

# Pulumi Stack Configurations and Dynamic Infrastructure

This skill helps AI explain how Pulumi stack configurations enable flexible, environment-aware infrastructure. It covers dynamic stack configuration principles, reusable parameterization, conditional resource creation, stack composition, and runtime stack context logic with concrete Pulumi code examples.

---

## When to use this skill

Use this skill when you need to:

- explain why dynamic stack configurations are essential for adaptable IaC,
- describe how Pulumi uses stack context and configuration as code,
- show practical examples of environment-based resource behavior,
- teach teams how to build reusable, composable Pulumi infrastructure,
- help engineers manage cloud-specific settings across multiple stacks.

---

## Outcome

Produce guidance that:

- defines dynamic stack configuration as the ability to adjust infrastructure behavior based on stack context,
- positions configuration-as-code as a core Pulumi pattern for reusable environments,
- shows how Pulumi stack settings, context-aware logic, and conditional resources work together,
- includes code snippets for dynamic S3 bucket naming, conditional resources, stack-based tagging, and dynamic outputs,
- explains how dynamic stacks support multiple clouds, test environments, and production deployments.

---

## Instructions for the AI

1. **Explain dynamic stack configuration principles**
   - Describe dynamic configuration as infrastructure that adapts based on stack, environment variables, or runtime conditions.
   - Emphasize environment context awareness, parameterization, conditional deployment, and automation.
   - Contrast dynamic stack configuration with static, one-size-fits-all infrastructure.

2. **Show Pulumi config as code**
   - Explain that Pulumi programs treat configuration as first-class code inputs.
   - Show how `new pulumi.Config()` reads stack-specific values and how code can branch based on those values.
   - Example:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const cfg = new pulumi.Config();
const environment = cfg.require("environment");

const bucket = new aws.s3.Bucket("dynamicBucket", {
    bucket: `my-${environment}-bucket`,
    acl: environment === "production" ? "private" : "public-read",
});
```

3. **Teach parameterization and reuse**
   - Explain how functions and components let teams reuse infrastructure logic across stacks.
   - Show a reusable bucket component and different instances for dev/prod.
   - Example:

```typescript
class MyBucket extends pulumi.ComponentResource {
    bucket: aws.s3.Bucket;

    constructor(name: string, acl: pulumi.Input<string>, opts?: pulumi.ComponentResourceOptions) {
        super("custom:infra:MyBucket", name, {}, opts);
        this.bucket = new aws.s3.Bucket(name, { acl }, { parent: this });
    }
}

const bucketDev = new MyBucket("my-dev-bucket", "public-read");
const bucketProd = new MyBucket("my-prod-bucket", "private");
```

4. **Show conditional resource creation**
   - Explain that Pulumi programs can include normal programming logic to create or skip resources based on stack conditions.
   - Example:

```typescript
const stack = pulumi.getStack();
const createDebugBucket = stack !== "production";

if (createDebugBucket) {
    new aws.s3.Bucket("debugBucket", { acl: "private" });
}
```

5. **Explain stack-aware tagging and outputs**
   - Demonstrate dynamic resource tags and outputs that vary by stack.
   - Example:

```typescript
const environment = pulumi.getStack();
const tags = { Environment: environment, Project: "MyApp" };

const instance = new aws.ec2.Instance("exampleInstance", {
    instanceType: "t3.small",
    tags,
});

export const instanceIp = instance.publicIp.apply(ip =>
    ip ? `Running in ${environment} with IP ${ip}` : "No IP yet",
);
```

6. **Describe multi-cloud flexibility**
   - Explain how the same Pulumi codebase can support AWS, Azure, and GCP by parameterizing providers and resources.
   - Recommend using stack config and components to isolate cloud-specific cases.
   - Example:

```typescript
const cloud = cfg.require("cloud");
let provider;

if (cloud === "aws") {
    provider = new aws.Provider("aws", { region: cfg.require("awsRegion") });
} else if (cloud === "azure") {
    provider = new azure.Provider("azure", { features: {} });
}
```

7. **Advise on stack composition**
   - Explain the benefit of composing stacks from reusable components and smaller modules.
   - Recommend packaging shared infrastructure as libraries or modules.
   - Mention that stack composition reduces duplication and improves maintainability.

8. **Warn about configuration drift and state management**
   - Explain that dynamic stacks still need careful state management and drift detection.
   - Recommend keeping configuration in the stack and using Pulumi previews to inspect changes before apply.
   - Explain that overly dynamic code can make drift harder to reason about, so keep stack logic clear.

---

## Decision points and guidance

- **Is the stack target environment-specific?** If yes, use stack config and `pulumi.Config()`.
- **Is the stack behavior conditional?** If yes, use programming logic to create or skip resources.
- **Do you need reusable environment shapes?** If yes, use component resources and helper functions.
- **Are you targeting multiple clouds?** If yes, parameterize providers and isolate cloud-specific code.
- **Is the configuration changing frequently?** If yes, keep dynamic logic simple and track it in stack config.

---

## Quality criteria

A strong response should ensure the guidance is:

- **adaptive:** stack behavior changes based on stack context and environment,
- **parameterized:** infrastructure settings are reusable and not hard-coded,
- **composable:** smaller components and stacks are assembled clearly,
- **stack-aware:** stack settings and tags reflect the environment,
- **manageable:** dynamic configuration is easy to reason about and audit.

---

## Example prompts

- "How do I make one Pulumi codebase run in dev, staging, and production?"
- "How can Pulumi stack config change resource names and ACLs by environment?"
- "What is the best way to conditionally create resources in Pulumi?"
- "How do I build reusable Pulumi components for multiple stacks?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-environments
- dev-ops-infrastructure-as-code-pulumi-workspaces
- dev-ops-infrastructure-as-code-pulumi-testing
- dev-ops-infrastructure-as-code-pulumi-secrets
