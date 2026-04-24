---
name: dev-ops-infrastructure-as-code-pulumi-debugging
description: Guides teams to diagnose and troubleshoot Pulumi stacks, with TypeScript and CLI debugging examples, configuration validation, logging, state inspection, and collaborative debugging practices.
---

# Pulumi Stack Debugging and Troubleshooting

This skill helps AI explain how to debug Pulumi stacks effectively. It covers configuration validation, deployment diagnostics, resource troubleshooting, logging, state inspection, and collaborative DevOps debugging workflows with TypeScript and Pulumi CLI examples.

---

## When to use this skill

Use this skill when you need to:

- explain debugging strategies for Pulumi infrastructure code,
- diagnose configuration and deployment failures in Pulumi stacks,
- interpret Pulumi CLI errors, previews, and stack diagnostics,
- show how to use Pulumi logs, `--debug`, state export/import, and previews,
- teach collaborative workflows between developers and operators for IaC debugging.

---

## Outcome

Produce guidance that:

- defines the common sources of Pulumi stack failures,
- shows how to detect and resolve configuration, dependency, and resource issues,
- uses Pulumi CLI commands to preview, validate, refresh, and export state,
- includes TypeScript examples for in-program logging and config validation,
- explains how to use Pulumi diagnostics with logs, stack history, and state inspection,
- encourages collaborative debugging and CI/CD validation practices.

---

## Instructions for the AI

1. **Explain Pulumi debugging principles**
   - Define debugging in the context of infrastructure as code as the process of aligning desired state with actual state.
   - Mention that Pulumi debugging covers configuration errors, resource misconfigurations, dependency issues, and runtime problems.
   - Emphasize that Pulumi’s programming model introduces both code validation needs and infrastructure validation needs.

2. **Cover configuration error detection**
   - Explain how `pulumi config` values and `pulumi.Config` calls can fail if required values are missing or invalid.
   - Show how to validate config values proactively with helper functions.

```typescript
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();
function requireEnv(key: string): string {
    const value = config.get(key);
    if (!value) {
        pulumi.log.error(`Missing Pulumi config value: ${key}`);
        throw new Error(`Missing required config: ${key}`);
    }
    return value;
}

const domainName = requireEnv("domainName");
const environment = config.get("environment") || "dev";

pulumi.log.info(`Deploying stack ${pulumi.getStack()} to ${environment}`);
```

3. **Describe Pulumi CLI debug workflows**
   - Show how to use `pulumi preview --diff` to catch issues before deployment.
   - Explain `pulumi up --debug` for more verbose CLI output.
   - Mention `pulumi refresh` for reconciling state with live infrastructure.

```bash
pulumi login
pulumi stack select dev
pulumi preview --diff
pulumi up --yes --debug
pulumi refresh --yes
```

4. **Explain resource diagnostics and dependency tracing**
   - Describe how failed resources often expose the underlying cause in their error messages.
   - Explain how to inspect resource dependency graphs with `pulumi stack export` and `pulumi stack output`.
   - Recommend using `pulumi logs` if the resource is an application service that emits logs.

5. **Show TypeScript logging and diagnostics in Pulumi programs**
   - Explain the use of `pulumi.log.info`, `pulumi.log.warn`, and `pulumi.log.error`.
   - Show logging resource inputs, computed values, and failure conditions.

```typescript
const apiKey = config.requireSecret("apiKey");

pulumi.log.info("Starting deployment for notification service.");

const service = new SomeResource("notificationService", {
    endpoint: config.require("endpoint"),
    apiKey: apiKey,
});

pulumi.log.info(`Built resource ${service.name}`);
```

6. **Show state inspection and rollback support**
   - Explain how `pulumi stack export` creates a JSON snapshot and `pulumi stack import` can restore state.
   - Show using `pulumi stack output` and `pulumi stack history` for debugging.

```bash
pulumi stack export --stack dev > dev-stack.json
pulumi stack import --stack dev < dev-stack.json
pulumi stack output
pulumi stack history --stack dev
```

7. **Describe common error categories**
   - Syntax and type errors in TypeScript code.
   - Missing or invalid config values.
   - Resource property validation failures.
   - Dependency ordering and circular dependency issues.
   - Provider authentication failures and cloud API errors.

8. **Explain advanced debugging techniques**
   - Use `pulumi up --skip-preview` sparingly and only after preview validation.
   - Use `pulumi stack output` to inspect values used by downstream resources.
   - Use `pulumi state` commands or exported state to compare expected vs actual resources.
   - Mention `pulumi stack rm` only when you want to delete a broken stack completely.

9. **Encourage collaborative debugging practices**
   - Emphasize sharing CLI output, stack previews, and state snapshots with team members.
   - Recommend combining Pulumi logs with cloud provider logs (CloudWatch, Azure Monitor, etc.).
   - Advocate blameless postmortems and shared troubleshooting notes.

10. **Link debugging to CI/CD validation**
    - Recommend gating deployments with `pulumi preview` in CI.
    - Suggest adding `pulumi validate` or custom validation scripts to CI pipelines.
    - Encourage using dedicated test stacks for development and staging.

---

## Decision points and guidance

- **Is the failure a config issue or a resource issue?** Use `pulumi preview --diff` and config validation first.
- **Is the stack’s live state stale?** Use `pulumi refresh` to reconcile.
- **Is the error coming from a cloud API?** Inspect resource-specific logs and provider error details.
- **Do you need an audit trail?** Use `pulumi stack history` and exported state snapshots.
- **Should the issue be resolved collaboratively?** Capture logs, stack exports, and CLI output and share with the team.

---

## Quality criteria

A strong Pulumi debugging response should ensure the guidance is:

- **diagnostic:** identifies both config and resource failure modes,
- **actionable:** includes exact Pulumi CLI commands and TypeScript examples,
- **preventive:** encourages preview, validation, and config checks before deploy,
- **observable:** uses logs, stack output, and state inspection to reveal issues,
- **collaborative:** connects debugging to team workflows and CI/CD validation.

---

## Example prompts

- "How do I debug a failed Pulumi deployment in TypeScript?"
- "What Pulumi CLI commands help find configuration errors before deploy?"
- "How do you inspect Pulumi state and stack outputs when a resource fails?"
- "What are best practices for collaborative Pulumi stack debugging?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-testing
- dev-ops-infrastructure-as-code-pulumi-cicd
- dev-ops-infrastructure-as-code-pulumi-secrets
- dev-ops-infrastructure-as-code-pulumi-stack-configurations
