---
name: dev-ops-infrastructure-as-code-pulumi-custom-providers
description: Guides teams in building Pulumi custom providers and resources for extensible, proprietary, and hybrid infrastructure use cases.
---

# Pulumi Custom Providers and Resources

This skill helps AI explain the full process of developing, packaging, testing, and publishing custom Pulumi providers and custom resources. It includes practical TypeScript examples for provider schema design, CRUD implementation, packaging, validation, and distribution.

---

## When to use this skill

Use this skill when you need to:

- explain the end-to-end custom provider development lifecycle,
- design provider and resource schemas for proprietary APIs,
- implement create/read/update/delete operations in Pulumi providers,
- package custom providers as plugins or reusable npm modules,
- validate custom providers with unit and integration tests,
- publish provider packages to registries or private repositories.

---

## Outcome

Produce guidance that:

- details provider and resource schema design,
- outlines CRUD lifecycle implementations for custom resources,
- shows how to implement a Pulumi provider SDK in TypeScript,
- explains packaging, testing, and publishing workflows,
- includes a concrete notification service example,
- distinguishes component resources from full custom providers.

---

## Instructions for the AI

1. **Describe the development flow**
   - Explain that custom provider development begins with schema design and ends with packaging and publication.
   - Break the flow into these stages: provider schema, resource schema, CRUD implementation, SDK development, packaging, testing, publishing.

2. **Explain provider and resource schema design**
   - Describe provider schema as the configuration and authentication contract for the provider.
   - Describe resource schema as the input/output shape of a custom resource.
   - Mention authentication parameters, endpoints, API keys, and resource attributes.

3. **Show a concrete NotificationService example**
   - Provide TypeScript examples for both `NotificationProvider` and `NotificationResource`.
   - Use `pulumi.dynamic.ResourceProvider` and `pulumi.dynamic.Resource` for the provider-backed resource.

```typescript
import * as pulumi from "@pulumi/pulumi";

interface NotificationProviderArgs {
    apiKey: pulumi.Input<string>;
    endpoint: pulumi.Input<string>;
}

interface NotificationInputs {
    message: string;
    recipient: string;
}

class NotificationProvider implements pulumi.dynamic.ResourceProvider {
    constructor(private readonly apiUrl: string) {}

    async create(inputs: any): Promise<pulumi.dynamic.CreateResult> {
        const response = await fetch(`${this.apiUrl}/services`, {
            method: "POST",
            headers: {
                "Content-Type": "application/json",
                "Authorization": `Bearer ${inputs.apiKey}`,
            },
            body: JSON.stringify({
                message: inputs.message,
                recipient: inputs.recipient,
            }),
        });

        const data = await response.json();
        const serviceId = data.id;

        return {
            id: serviceId,
            outs: {
                serviceId,
                message: inputs.message,
                recipient: inputs.recipient,
            },
        };
    }

    async update(id: pulumi.ID, olds: any, news: any): Promise<pulumi.dynamic.UpdateResult> {
        await fetch(`${this.apiUrl}/services/${id}`, {
            method: "PATCH",
            headers: {
                "Content-Type": "application/json",
                "Authorization": `Bearer ${news.apiKey}`,
            },
            body: JSON.stringify({
                message: news.message,
                recipient: news.recipient,
            }),
        });

        return { outs: { ...news, serviceId: id } };
    }

    async delete(id: pulumi.ID, props: any): Promise<void> {
        await fetch(`${this.apiUrl}/services/${id}`, {
            method: "DELETE",
            headers: {
                "Authorization": `Bearer ${props.apiKey}`,
            },
        });
    }
}

interface NotificationResourceArgs {
    apiKey: pulumi.Input<string>;
    endpoint: pulumi.Input<string>;
    message: pulumi.Input<string>;
    recipient: pulumi.Input<string>;
}

export class NotificationResource extends pulumi.dynamic.Resource {
    public readonly serviceId!: pulumi.Output<string>;

    constructor(name: string, args: NotificationResourceArgs, opts?: pulumi.CustomResourceOptions) {
        super(new NotificationProvider(args.endpoint as string), name, {
            ...args,
            serviceId: undefined,
        }, opts);
    }
}
```

4. **Describe CRUD operations clearly**
   - Explain that `create()` provisions the resource and returns a unique ID plus output state.
   - Explain that `read()` can refresh resource state from the external service if needed.
   - Explain that `update()` applies changes and returns new outputs.
   - Explain that `delete()` removes the external resource cleanly.

5. **Explain SDK implementation in TypeScript**
   - Describe using the Pulumi dynamic provider model for quick custom provider development.
   - Mention that full provider plugins can also be implemented using Pulumi’s provider SDKs in Go, Python, or C#.
   - Show how schema and resource types translate to TypeScript interfaces and classes.

6. **Show example usage in a Pulumi program**
   - Demonstrate instantiating the custom resource and exporting values.

```typescript
import * as pulumi from "@pulumi/pulumi";
import { NotificationResource } from "./NotificationResource";

const notificationService = new NotificationResource("myNotificationService", {
    apiKey: pulumi.config.requireSecret("notificationApiKey"),
    endpoint: "https://api.notificationservice.com",
    message: "Hello from Pulumi!",
    recipient: "example@example.com",
});

export const notificationServiceId = notificationService.serviceId;
```

7. **Explain packaging and plugin distribution**
   - State that provider code can be packaged as an npm module or a Pulumi provider plugin.
   - Explain `pulumi plugin install` for plugin-based providers and `npm install` for package-based SDKs.
   - Mention packaging metadata, versioning, and the importance of publishing release artifacts.

8. **Describe testing and validation**
   - Recommend unit tests for provider logic and resource input validation.
   - Recommend integration tests inside Pulumi stacks using `@pulumi/pulumi/x/automation` or stack references.
   - Example unit test skeleton:

```typescript
import { NotificationProvider } from "./NotificationProvider";

test("create sends message and returns id", async () => {
    const provider = new NotificationProvider("https://api.notificationservice.com");
    const result = await provider.create({
        apiKey: "test-key",
        endpoint: "https://api.notificationservice.com",
        message: "Hi",
        recipient: "user@example.com",
    });

    expect(result.id).toBeDefined();
    expect(result.outs.serviceId).toEqual(result.id);
});
```

9. **Explain publishing and distribution**
   - Cover publishing to npm, private registries, or a Pulumi plugin repository.
   - Advise including README examples, schema docs, and compatibility/version notes.
   - Mention private artifactory or GitHub Packages for internal distribution.

10. **Cover advanced extension patterns**
    - Describe using `registerResourceModule` for local provider modules.
    - Explain component resources vs custom providers, and when to use each.
    - Show how a provider-backed resource can expose a clean, typed API while delegating lifecycle to the provider.

11. **List best practices for maintainability**
    - Modular code structure: split provider, resources, and API clients.
    - Consistent naming conventions for resource types and properties.
    - Strong input validation and helpful error messages.
    - Use secrets for API credentials and avoid embedding them in code.
    - Version provider packages semantically and maintain backward compatibility.
    - Document resource arguments, outputs, and example adoption patterns.
    - Provide integration tests simulating real API behavior and failure modes.
    - Implement lifecycle cleanup to avoid orphaned resources.
    - Monitor provider performance and external API retries.
    - Plan upgrades and deprecation carefully.

12. **Explain when not to build a full provider**
    - Recommend component resources when the integration can be expressed by composing existing providers.
    - Reserve full custom providers for services with real lifecycle management needs or unsupported APIs.
    - Warn that custom providers increase maintenance and should only be used when required.

---

## Decision points and guidance

- **Is the service already supported by a built-in provider?** Use it first.
- **Is the integration a simple grouping of resources?** Use a component resource.
- **Does the resource require external lifecycle operations and state reconciliation?** Use a custom provider.
- **Will the provider be reused by multiple teams or projects?** Package it as a reusable module.
- **Do you need strong validation and secret management?** Put these checks in provider schema and tests.

---

## Quality criteria

A strong response should ensure the guidance is:

- **complete:** covers schema, CRUD, packaging, testing, and publishing,
- **practical:** includes real TypeScript examples and provider patterns,
- **secure:** treats API keys and credentials as secrets,
- **maintainable:** organizes provider code into reusable modules,
- **correct:** chooses component resources when they are sufficient and custom providers only when necessary.

---

## Example prompts

- "How do I build a custom Pulumi provider for a notification service?"
- "What are the steps to package and publish a Pulumi provider plugin?"
- "How should I test a Pulumi dynamic resource provider?"
- "When should I use component resources instead of custom providers?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-stack-configurations
- dev-ops-infrastructure-as-code-pulumi-environment-configurations
- dev-ops-infrastructure-as-code-pulumi-workspaces
- dev-ops-infrastructure-as-code-pulumi-testing
