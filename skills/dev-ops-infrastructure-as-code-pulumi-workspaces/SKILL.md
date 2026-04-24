---
name: dev-ops-infrastructure-as-code-pulumi-workspaces
description: Guides teams to use Pulumi workspaces, Git versioning, CI/CD integration, and Pulumi Deployments for collaborative IaC projects.
---

# Pulumi Workspaces and Collaboration

This skill helps AI explain how Pulumi workspaces support team collaboration, how Git and CI/CD integrate with Pulumi projects, and how Pulumi Deployments scale modern infrastructure workflows.

---

## When to use this skill

Use this skill when you need to:

- describe how Pulumi workspaces isolate configurations, state, and resources for concurrent development,
- explain why workspace isolation reduces conflict and enables scoped environment settings,
- describe Git versioning practices for Pulumi infrastructure code,
- explain CI/CD integration patterns for Pulumi deployments,
- introduce Pulumi Deployments as a managed automation platform for infrastructure updates.

---

## Outcome

Produce guidance that:

- explains workspaces as independent units within a Pulumi project,
- shows how each workspace can maintain separate cloud provider credentials, regions, and deployment parameters,
- stresses the value of workspace isolation for developers working on distinct features,
- connects Git branches and commits to safe infrastructure evolution,
- describes how CI/CD pipelines can validate, test, and deploy Pulumi stacks from Git,
- introduces Pulumi Deployments as a way to trigger updates, enforce stack settings, and minimize local credential exposure.

---

## Instructions for the AI

1. **Explain Pulumi workspace isolation**
   - Describe a workspace as an isolated environment within a Pulumi project, with its own config, state, and runtime settings.
   - Emphasize that workspaces allow multiple developers to work on the same project concurrently without overwriting each other’s state.
   - Give examples such as one workspace targeting networking while another targets database configuration.

2. **Describe scoped workspace configuration**
   - Explain that workspaces can carry provider-specific settings like AWS region, Azure subscription, or GCP project.
   - Describe how scoped config lets each workspace operate in its designated environment.
   - Advise using workspaces to keep environment-specific details separate from shared infrastructure code.

3. **Connect Git versioning to Pulumi code**
   - Explain that Pulumi projects and stacks should be versioned in Git like any application.
   - Describe commit-based history, branch workflows, and merge reviews for infrastructure changes.
   - Emphasize that Git allows auditability, rollback, and parallel development for Pulumi code.

4. **Explain CI/CD integration patterns**
   - Describe how Git commits can trigger CI/CD pipelines that run Pulumi validation, policy checks, and deployments.
   - Explain that pipelines should lint, test, and validate IaC before applying changes to target environments.
   - Advise correlating branches with deployment targets, e.g. feature branches for previews and main branch for staging/production.

5. **Introduce Pulumi Deployments**
   - Explain that Pulumi Deployments is a managed platform for running Pulumi updates at scale.
   - Describe its components: compute, configuration, and composition.
   - Explain that deployment settings include source control, credentials, build prerequisites, OIDC auth, and stack parameters.

6. **Describe deployment triggers and automation**
   - Explain that Deployments can be triggered by Git pushes, pull request reviews, REST API calls, Review Stacks, or remote automation.
   - Emphasize that this flexibility allows teams to choose workflows beyond traditional CI/CD.
   - Encourage using deployment settings once and reusing them across triggers.

7. **Encourage collaboration best practices**
   - Recommend clear communication channels, code reviews, and shared documentation for Pulumi projects.
   - Advise using Git branches and PRs to isolate work and review infrastructure changes.
   - Recommend using workspace isolation and deployment platforms to reduce local credential exposure and protect sensitive settings.

---

## Decision points and guidance

- **Does the team need to work concurrently on the same project?** If yes, use separate workspaces for isolation.
- **Are environment settings different across clouds or contexts?** If yes, scope them to the appropriate workspace.
- **Is infrastructure code stored in Git?** If no, move the Pulumi project into version control immediately.
- **Should infrastructure changes be reviewed before deployment?** If yes, use Git branches, pull requests, and pipeline validation.
- **Do deployments need to be triggered from more than one source?** If yes, consider Pulumi Deployments and its multi-trigger capabilities.

---

## Quality criteria

A strong response should ensure the guidance is:

- **isolated:** workspaces keep state and config separate for concurrent developers,
- **scoped:** workspace-specific settings are kept in the right context,
- **versioned:** Pulumi code is managed in Git with clear history,
- **automated:** CI/CD pipelines validate and deploy infrastructure changes,
- **managed:** Pulumi Deployments centralize stack settings and triggers,
- **collaborative:** teams communicate, review, and document Pulumi workflows.

---

## Example prompts

- "Why should we use separate Pulumi workspaces for each developer?"
- "How do Git and branches fit into Pulumi infrastructure development?"
- "What should a Pulumi CI/CD pipeline do before `pulumi up`?"
- "How are Pulumi Deployments different from a normal GitHub Actions workflow?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-secrets
- dev-ops-infrastructure-as-code-pulumi-environments
- dev-ops-infrastructure-as-code-cloud-security
- dev-ops-infrastructure-as-code-configuration-management
