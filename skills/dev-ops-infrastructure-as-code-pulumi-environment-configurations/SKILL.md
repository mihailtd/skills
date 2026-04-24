---
name: dev-ops-infrastructure-as-code-pulumi-environment-configurations
description: Guides teams to manage environment-specific Pulumi configuration, secrets, network, scaling, and deployment settings with practical examples.
---

# Pulumi Environment-Specific Configuration

This skill helps AI explain how to design environment-specific Pulumi configurations, manage secrets per stack, and adapt infrastructure to dev, staging, and production environments with code examples.

---

## When to use this skill

Use this skill when you need to:

- explain how Pulumi supports environment-aware infrastructure setups,
- describe stack-specific secrets, credentials, regions, and instance sizing,
- show how to dynamically configure networking, tagging, and scaling per environment,
- illustrate best practices for safe secret handling, drift remediation, and resource tagging,
- help teams manage multiple environments efficiently.

---

## Outcome

Produce guidance that:

- explains environment-specific configuration as a fundamental pattern for scalable Pulumi stacks,
- demonstrates how Pulumi stack settings and `pulumi.getStack()` are used to vary behavior,
- includes code snippets for dynamic API endpoints, AWS region selection, security group CIDR ranges, autoscaling, Lambda settings, and secret management,
- warns against storing secrets in code while showing secure Pulumi secret patterns,
- describes how to keep environment-specific logic readable and maintainable.

---

## Instructions for the AI

1. **Describe environment-specific Pulumi config**
   - Explain that Pulumi stack config lets each environment pass its own parameters without changing code.
   - Mention that `pulumi.Config()` and stack outputs are the standard patterns.
   - Emphasize that environment-specific config is essential for dev/staging/prod differences.

2. **Show environment-specific endpoint selection**
   - Provide a code sample for choosing API endpoints by stack.

```typescript
import * as pulumi from "@pulumi/pulumi";

const stack = pulumi.getStack();
const apiEndpoint = stack === "dev"
    ? "https://dev.api.example.com"
    : stack === "staging"
    ? "https://staging.api.example.com"
    : "https://api.example.com";

export const endpoint = apiEndpoint;
```

3. **Show region-specific provider configuration**
   - Demonstrate choosing AWS regions or cloud-specific config by stack.

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const region = pulumi.getStack() === "prod" ? "eu-west-1" : "us-west-2";
const provider = new aws.Provider("aws", { region });

const bucket = new aws.s3.Bucket("exampleBucket", { bucket: `app-${pulumi.getStack()}` }, { provider });
```

4. **Show environment-specific secret handling**
   - Explain secure secret usage instead of hard-coding credentials.

```typescript
import * as pulumi from "@pulumi/pulumi";

const cfg = new pulumi.Config();
const dbPassword = cfg.requireSecret("dbPassword");

const db = new aws.rds.Instance("exampleDB", {
    engine: "mysql",
    instanceClass: "db.t3.micro",
    username: "admin",
    password: dbPassword,
});
```

5. **Show conditional resource sizing and autoscaling**
   - Demonstrate varying capacity or memory per environment.

```typescript
const stack = pulumi.getStack();
const instanceType = stack === "prod" ? "m5.large" : "t3.micro";

const instance = new aws.ec2.Instance("web", { instanceType });
```

6. **Show environment-specific network configuration**
   - Provide a security group example that uses different CIDR blocks.

```typescript
const cidr = stack === "dev" ? "10.0.0.0/24" : "10.1.0.0/24";

const sg = new aws.ec2.SecurityGroup("web-sg", {
    ingress: [{ protocol: "tcp", fromPort: 80, toPort: 80, cidrBlocks: [cidr] }],
});
```

7. **Show dynamic tagging by environment**
   - Demonstrate automated tags that reflect stack context.

```typescript
const tags = { Environment: stack, Project: "MyApp" };
const instance = new aws.ec2.Instance("instance", { instanceType: "t3.micro", tags });
```

8. **Explain environment-specific component reuse**
   - Recommend creating components that accept stack-aware inputs and produce consistent resources.
   - Mention packaging reusable components as NPM modules if using TypeScript.

9. **Warn about secret and config anti-patterns**
   - Explain that secrets should not be stored in code or plaintext config files.
   - Encourage using Pulumi secrets and external secret providers for environment-specific credentials.
   - Advise keeping stack logic simple and avoiding excessive conditional complexity.

---

## Decision points and guidance

- **Is this value environment-specific?** If yes, place it in stack config or environment-specific settings.
- **Is the value sensitive?** If yes, use `requireSecret` and encrypted Pulumi secrets.
- **Does the deployment need a different region or provider?** If yes, branch by stack or provider configuration.
- **Does the environment need different capacity or scaling?** If yes, parameterize autoscaling and instance size per stack.
- **Is the environment logic becoming tangled?** If yes, move it into reusable components or helper functions.

---

## Quality criteria

A strong response should ensure the guidance is:

- **stack-aware:** every environment can drive behavior through stack config,
- **secure:** secrets are protected and never hard-coded,
- **dynamic:** resource definitions adapt based on environment context,
- **modular:** environment-specific logic is easy to reuse,
- **simple:** dynamic configuration remains readable and maintainable.

---

## Example prompts

- "How do I configure Pulumi for dev, staging, and production without copying code?"
- "How should I manage environment-specific AWS regions in Pulumi?"
- "What is the right way to handle secrets per Pulumi stack?"
- "How can I conditionally provision resources based on the Pulumi stack?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-stack-configurations
- dev-ops-infrastructure-as-code-pulumi-workspaces
- dev-ops-infrastructure-as-code-pulumi-secrets
- dev-ops-infrastructure-as-code-configuration-management
