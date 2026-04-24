---
name: dev-ops-infrastructure-as-code-pulumi-configuration-strategies
description: Guides teams to manage Pulumi configuration, stack profiles, multi-cloud deployments, test environments, and rollback strategies with code examples.
---

# Pulumi Configuration Strategies

This skill helps AI explain how Pulumi supports flexible infrastructure through configuration files, stack profiles, environment variables, componentization, multi-cloud deployments, dynamic scaling, stateful resource handling, and rollback workflows.

---

## When to use this skill

Use this skill when you need to:

- explain Pulumi stack configuration and environment-specific profiles,
- describe secure environment variable and secret management,
- teach conditional resource creation and reusable components,
- show multi-region and multi-cloud deployment strategies,
- demonstrate dynamic scaling and testing environment setup,
- describe stateful resource policies and rollback/disaster recovery.

---

## Outcome

Produce guidance that:

- explains Pulumi stacks as independent environment instances,
- shows how configuration files and stack config separate environment settings from code,
- emphasizes secrets and environment variables for secure runtime configuration,
- demonstrates conditional logic, templates, and componentization for reusable infrastructure,
- includes multi-region and multi-cloud provider examples,
- describes dynamic scaling, environment-specific test stacks, and rollback strategies.

---

## Instructions for the AI

1. **Explain stack-based configuration**
   - Describe stacks as distinct instances of the same infrastructure definition.
   - Explain that `pulumi config` and stack-specific config files let teams define `dev`, `staging`, and `prod` settings separately.
   - Mention `pulumi stack select` and how each stack has its own configuration and state.

2. **Show environment variables and secrets**
   - Explain using environment variables for runtime settings and `pulumi.Config` for stack values.
   - Stress that secrets should be stored as encrypted Pulumi secrets, not hard-coded.
   - Example:

```typescript
import * as pulumi from "@pulumi/pulumi";

const cfg = new pulumi.Config();
const apiKey = cfg.requireSecret("apiKey");
const region = process.env.AWS_REGION || "us-west-2";

export const serviceKey = apiKey;
export const awsRegion = region;
```

3. **Teach conditional logic and abstraction**
   - Explain that Pulumi programs can use normal programming logic to vary resource creation by stack.
   - Show reusable helper functions and components for common patterns.
   - Example:

```typescript
const stack = pulumi.getStack();
const isProd = stack === "prod";

function createBucket(name: string, isProd: boolean) {
    return new aws.s3.Bucket(name, {
        bucket: `${name}-${stack}`,
        acl: isProd ? "private" : "public-read",
    });
}

const bucket = createBucket("app-data", isProd);
```

4. **Show templates and componentization**
   - Explain how components encapsulate infrastructure patterns and reduce duplication.
   - Provide a component example and instantiate it for different environments.
   - Example:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

class WebApp extends pulumi.ComponentResource {
    constructor(name: string, opts?: pulumi.ComponentResourceOptions) {
        super("custom:infra:WebApp", name, {}, opts);
        new aws.s3.Bucket(`${name}-assets`, { acl: "private" }, { parent: this });
    }
}

const appDev = new WebApp("webapp-dev");
const appProd = new WebApp("webapp-prod");
```

5. **Explain versioning and dependency management**
   - Emphasize version control for Pulumi programs, config files, and component libraries.
   - Recommend package-lock or `npm shrinkwrap` and explicit dependency versions for reproducible stacks.
   - Mention that stack config values should also be treated as part of deployable configuration.

6. **Show custom profiles and stack labels**
   - Explain custom profiles as stack-specific parameters or helper functions.
   - Example:

```typescript
const env = pulumi.getStack();
const profile = env === "prod" ? "high-performance" : "standard";

const instanceType = profile === "high-performance" ? "m5.large" : "t3.micro";
```

7. **Show multi-region and multi-cloud deployments**
   - Demonstrate using multiple providers for AWS and Azure.
   - Example:

```typescript
import * as aws from "@pulumi/aws";
import * as azure from "@pulumi/azure";

const awsProvider = new aws.Provider("awsProvider", { region: "us-west-2" });
const azureProvider = new azure.Provider("azureProvider", { location: "West US" });

const awsBucket = new aws.s3.Bucket("awsBucket", {}, { provider: awsProvider });
const azureStorage = new azure.storage.Account("azureStorage", {
    resourceGroupName: "rg",
    accountTier: "Standard",
    accountReplicationType: "LRS",
}, { provider: azureProvider });
```

8. **Show dynamic scaling and responsive resources**
   - Explain environment-responsive scaling based on the current stack.
   - Example:

```typescript
import * as aws from "@pulumi/aws";

const env = pulumi.getStack();
const desiredCapacity = env === "prod" ? 10 : 5;

const group = new aws.autoscaling.Group("exampleGroup", {
    minSize: 1,
    maxSize: desiredCapacity,
    desiredCapacity,
});
```

9. **Show environment-specific testing environments**
   - Explain using dedicated stacks for test, staging, or performance environments.
   - Example:

```typescript
const testEnv = new pulumi.StackReference("org/test-environment");
const bucketName = testEnv.get("bucketName");
const testBucket = new aws.s3.Bucket("testBucket", {
    bucket: pulumi.interpolate`${bucketName}`,
});
```

10. **Address stateful resource management**
    - Explain special care for stateful resources such as databases or file systems.
    - Example:

```typescript
const db = new aws.rds.Instance("exampleDatabase", {
    engine: "mysql",
    instanceClass: "db.t3.micro",
    allocatedStorage: 20,
    backupRetentionPeriod: pulumi.getStack() === "prod" ? 7 : 1,
});
```

11. **Show rollback and disaster recovery patterns**
    - Explain using stack history and preview validation to reduce deployment risk.
    - Example pseudo-code:

```typescript
import * as pulumi from "@pulumi/pulumi";

const rollbackStack = new pulumi.StackReference("org/rollback-environment");

pulumi.runtime.setProgram(async () => {
    try {
        await pulumi.preview({ stack: rollbackStack.name });
        await pulumi.update({ stack: rollbackStack.name });
    } catch (err) {
        console.error("Deployment failed. Initiating rollback:", err);
        await pulumi.destroy({ stack: rollbackStack.name });
    }
});
```

12. **Warn about complexity and manageability**
    - Advise keeping environment logic readable and avoiding overly complex conditional branches.
    - Recommend limiting secrets, config, and environment-specific behavior to clear, well-documented abstractions.

---

## Decision points and guidance

- **Should this setting vary by environment?** If yes, move it into stack config or stack-specific logic.
- **Is this value sensitive?** If yes, use Pulumi secrets and avoid hard-coding.
- **Will you deploy to more than one cloud or region?** If yes, use multiple providers and isolate cloud-specific logic.
- **Is the environment logic getting complicated?** If yes, refactor into reusable components.
- **Do you need a separate test environment?** If yes, use stack references or dedicated test stacks.

---

## Quality criteria

A strong response should ensure the guidance is:

- **config-driven:** environments are defined through stack config rather than code edits,
- **secure:** secrets are encrypted and isolated per stack,
- **reusable:** components and templates reduce duplication,
- **multi-cloud-ready:** providers can be configured for AWS/Azure/GCP,
- **scalable:** dynamic scaling is environment-aware,
- **resilient:** stateful resources and rollback procedures are handled deliberately.

---

## Example prompts

- "How do I use Pulumi stacks and configs to manage dev, staging, and prod?"
- "What is the best way to define different AWS regions for each Pulumi stack?"
- "How can I use Pulumi to safely manage environment-specific secrets?"
- "How do I keep Pulumi environment-specific infrastructure reusable?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-stack-configurations
- dev-ops-infrastructure-as-code-pulumi-secrets
- dev-ops-infrastructure-as-code-pulumi-workspaces
- dev-ops-infrastructure-as-code-pulumi-testing
