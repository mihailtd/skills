---
name: dev-ops-infrastructure-as-code-pulumi-docker
description: Guides teams in building Docker images with Pulumi, covering Docker fundamentals, Pulumi Docker build definitions, caching, multi-stage builds, environment variables, and secrets management.
---

# Pulumi Docker and Container Build Strategy

This skill helps AI explain how Docker and Pulumi work together to create portable, repeatable container images for cloud native applications. It includes Docker fundamentals, Pulumi `docker.Image` usage, build configuration, caching, multi-stage builds, secret handling, environment variable strategies, and security best practices.

---

## When to use this skill

Use this skill when you need to:

- introduce Docker and containerization concepts for cloud applications,
- show how Pulumi can define Docker image builds as infrastructure code,
- explain build options, dependency management, and build-time optimization,
- demonstrate Docker caching and multi-stage build patterns,
- cover secrets and environment variable handling in Pulumi-managed containers,
- advise on secure, maintainable Docker build workflows.

---

## Outcome

Produce guidance that:

- explains Docker as the packaging layer for application portability and consistency,
- demonstrates defining Docker image creation with Pulumi TypeScript,
- shows build configuration, build args, and Dockerfile integration,
- covers caching and multi-stage build patterns for smaller, faster images,
- explains how to manage secrets and environment variables safely,
- provides actionable best practices for Docker builds with Pulumi.

---

## Instructions for the AI

1. **Explain Docker fundamentals first**
   - Describe Docker images, containers, Dockerfiles, and registries.
   - Emphasize portability, isolation, consistency, and how containers decouple apps from infrastructure.
   - Use the shipping container metaphor to reinforce uniform packaging across environments.

2. **Introduce Pulumi Docker integration**
   - Explain that Pulumi has a first-class Docker provider that can build images and create containers.
   - Mention that `@pulumi/docker` can build images from a local context and optionally push them to a registry.
   - Stress that this makes Docker builds part of the same IaC workflow as cloud resources.

3. **Show a simple Pulumi Docker image build example**
   - Provide a TypeScript example using `docker.Image` and a local application context.

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as docker from "@pulumi/docker";

const image = new docker.Image("typescript-app-image", {
    imageName: "myorg/typescript-app:latest",
    build: {
        context: "./app",
        dockerfile: "./app/Dockerfile",
    },
});

export const imageId = image.imageId;
export const imageName = image.imageName;
```

4. **Explain build options and dependencies**
   - Show how to pass `buildArgs`, `dockerfile`, `context`, and `extraOptions`.
   - Explain that dependencies can be managed by including package manifests and scripts inside the build context.
   - Use Pulumi config values to drive build-time settings.

```typescript
const config = new pulumi.Config();
const environment = config.require("environment");

const image = new docker.Image("my-app-image", {
    imageName: `myorg/my-app:${environment}`,
    build: {
        context: "./app",
        dockerfile: "./app/Dockerfile",
        buildArgs: {
            ENVIRONMENT: environment,
            API_URL: config.require("apiUrl"),
        },
    },
});
```

5. **Cover caching and build performance**
   - Explain Docker layer caching and why build ordering matters.
   - Show how to structure a Dockerfile to cache dependencies before copying application code.
   - Mention BuildKit and `docker buildx` support in Pulumi.

```dockerfile
# Dockerfile
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . ./
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
RUN npm install --only=prod
CMD ["node", "dist/index.js"]
```

6. **Explain multi-stage builds with Pulumi**
   - Describe how multi-stage builds produce smaller final images and improve security.
   - Include an example of a build stage for compilation and a runtime stage.

```typescript
const image = new docker.Image("multi-stage-image", {
    imageName: "myorg/multi-stage-app:latest",
    build: {
        context: "./app",
        dockerfile: "./app/Dockerfile.multi-stage",
        target: "production",
    },
});
```

```dockerfile
# Dockerfile.multi-stage
FROM node:18 AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . ./
RUN npm run build

FROM node:18-alpine AS production
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY package.json ./
RUN npm install --only=prod
CMD ["node", "dist/index.js"]
```

7. **Show environment variables and secrets management**
   - Explain that build arguments should not carry long-lived secrets into the final image.
   - Show using `pulumi.secret()` for values that must remain protected in the Pulumi state.
   - Explain runtime environment variables on containers, and external secret stores for production.

```typescript
const config = new pulumi.Config();
const dbPassword = config.requireSecret("dbPassword");

const appContainer = new docker.Container("app-container", {
    image: image.imageName,
    envs: pulumi.all([dbPassword]).apply(([password]) => [
        `NODE_ENV=production`,
        `DATABASE_URL=${password}`,
    ]),
});
```

8. **Explain secure build workflows**
   - Recommend avoiding hard-coded credentials in Dockerfiles or `buildArgs`.
   - Recommend external secret management (Vault, AWS Secrets Manager, GitHub Secrets) for sensitive values.
   - Mention private registry authentication and secure image pushing.

9. **Discuss Pulumi Docker build enhancements**
   - Mention `docker.Image` supports cross-platform builds, build kit, and rich logs.
   - Show how to push an image to a registry by attaching `registry` values.

```typescript
const registry = {
    server: "index.docker.io/v1/",
    username: process.env.DOCKER_USERNAME,
    password: process.env.DOCKER_PASSWORD,
};

const image = new docker.Image("pushable-image", {
    imageName: "myorg/my-app:latest",
    build: {
        context: "./app",
    },
    registry,
});
```

10. **List Docker build best practices**
    - Keep images small with multi-stage builds.
    - Order Dockerfile layers to maximize cache reuse.
    - Pin base image versions.
    - Separate build-time dependencies from runtime dependencies.
    - Use `.dockerignore` to exclude unnecessary files from the build context.
    - Avoid secrets in images; pass secrets at runtime.

11. **Explain the Pulumi value proposition**
    - Reinforce that Pulumi codifies Docker build declarations alongside cloud infrastructure.
    - Explain that Docker images become first-class infrastructure artifacts.
    - Note that this unifies application packaging and cloud provisioning in one TypeScript program.

---

## Decision points and guidance

- **Is this image for production?** Use multi-stage builds, minimal runtime base images, and dependency pinning.
- **Do you need dynamic build settings?** Use Pulumi config values and `buildArgs`.
- **Are secrets required at build time?** Prefer external secret stores and avoid baking them into the image.
- **Do you want faster rebuilds?** Optimize Dockerfile layer ordering and use BuildKit cache.
- **Should the image be pushed to a registry?** Configure `registry` credentials and secure token storage.

---

## Quality criteria

A strong response should ensure the guidance is:

- **clear:** explains basic Docker concepts before Pulumi specifics,
- **practical:** includes working TypeScript and Dockerfile examples,
- **efficient:** teaches caching and multi-stage builds,
- **secure:** avoids insecure secret handling patterns,
- **integrated:** shows how Docker builds fit into Pulumi IaC workflows.

---

## Example prompts

- "How do I build Docker images with Pulumi and TypeScript?"
- "What is the best way to use multi-stage Docker builds in Pulumi?"
- "How should I manage Docker build args and secrets with Pulumi?"
- "How do I push a Pulumi-built Docker image to a private registry?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-stack-configurations
- dev-ops-infrastructure-as-code-pulumi-testing
- dev-ops-infrastructure-as-code-pulumi-secrets
- dev-ops-infrastructure-as-code-pulumi-custom-providers
