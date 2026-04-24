---
name: dev-ops-infrastructure-as-code-pulumi-testing
description: Guides teams to test Pulumi-managed infrastructure with unit tests, integration and end-to-end validation, policy checks, CI/CD, and resilience testing.
---

# Pulumi Infrastructure as Code Testing

This skill helps AI explain how to build a comprehensive Pulumi IaC testing strategy that covers unit testing, integration and end-to-end validation, provisioning and configuration checks, policy enforcement, and pipeline automation.

---

## When to use this skill

Use this skill when you need to:

- explain why Pulumi infrastructure code needs dedicated testing methodologies,
- distinguish Pulumi IaC testing from traditional application testing,
- describe unit, integration, and workflow testing for Pulumi programs,
- recommend Pulumi-specific mocking, validation, and policy-as-code practices,
- support Pulumi deployments with CI/CD validation and resilient infrastructure checks.

---

## Outcome

Produce guidance that:

- frames IaC testing as verification of provisioning, configuration, dependencies, and compliance,
- shows how Pulumi testing helps catch drift, misconfiguration, and security gaps before deployment,
- explains unit testing Pulumi programs with mocks and assertions,
- describes integration and end-to-end tests for multi-resource workflows,
- encourages automation with CI/CD, linting, policy checks, and managed deployment validation,
- highlights resilience testing through failure scenarios and recovery validation.

---

## Instructions for the AI

1. **Explain IaC testing principles**
   - Define IaC testing as validation of infrastructure provisioning, desired state, configuration, dependencies, and compliance.
   - Contrast IaC testing with application testing: the goal is infrastructure correctness and operational resilience rather than business logic behavior.
   - Emphasize that IaC testing prevents drift, reduces outages, and improves security.

2. **Describe Pulumi unit testing**
   - Explain that Pulumi unit tests validate individual resources and resource definitions in isolation.
   - Encourage using Pulumi runtime mocks to simulate cloud provider behavior without deploying real resources.
   - Recommend asserting resource properties, tags, permissions, and configuration values with test frameworks like Mocha and Chai or equivalent.
   - Note that unit tests should be fast, isolated, and focused on Pulumi program logic rather than cloud provider implementation.

3. **Explain Pulumi mocking patterns**
   - Describe how Pulumi `runtime.setMocks()` lets tests create fake resource IDs and outputs.
   - Explain that mocks are useful for testing custom component resources, resource arguments, and output transformations.
   - Advise mocking external services and provider APIs so tests do not depend on live infrastructure.

4. **Describe integration and end-to-end testing**
   - Explain that integration tests verify interactions between multiple Pulumi-managed resources and services.
   - Recommend testing real workflows such as API Gateway to Lambda to database, or service-to-service communication across cloud resources.
   - Distinguish end-to-end testing as validating a complete user or system workflow, including provisioning, runtime behavior, and data flow.
   - Suggest running these tests in realistic test environments created with Pulumi IaC.

5. **Describe workflow validation and recovery testing**
   - Explain that workflow validation checks the provisioning and orchestration path from stack creation to application runtime.
   - Recommend failure scenario testing by simulating network issues, resource unavailability, or other faults.
   - Advise validating recovery behaviors, such as auto-scaling, failover, and retry logic.

6. **Cover policy and compliance checks**
   - Describe Pulumi Policy as Code to enforce security, cost, and compliance requirements before deployment.
   - Explain that policy checks are an essential validation layer in pipeline testing.
   - Mention example checks such as ensuring encryption is enabled, public access is blocked, and required tags are present.

7. **Encourage CI/CD and automation**
   - Recommend integrating Pulumi tests into CI/CD pipelines so every commit is validated.
   - Explain that pipelines should run linting, unit tests, policy checks, preview validation, and if appropriate, test stack deployments.
   - Advise using Git branch-based workflows with preview stacks for pull requests and gated deployment to staging/production.

8. **Warn about common IaC testing challenges**
   - Explain that cloud complexity, stateful resources, and external dependencies make IaC testing challenging.
   - Warn against testing only with real infrastructure; use a mix of mocked unit tests and real integration tests.
   - Recommend using infrastructure as code to provision test environments so tests are repeatable and consistent.

9. **Recommend collaboration and documentation**
   - Encourage documenting test expectations, stack test conventions, and how to run Pulumi tests.
   - Advise teams to review test coverage for infrastructure resources, policies, and workflows.
   - Recommend using code reviews to validate changes to both Pulumi programs and their associated tests.

---

## Decision points and guidance

- **Is this a resource-level check?** If yes, use Pulumi unit tests and mocks.
- **Is this a multi-resource behavior or workflow?** If yes, use integration or end-to-end tests.
- **Does this change affect security, compliance, or governance?** If yes, add policy-as-code checks and audit validation.
- **Do you need fast feedback?** If yes, keep unit tests lightweight and mock cloud interactions.
- **Do you need production-like confidence?** If yes, run tests in dedicated Pulumi test stacks or environments.
- **Are tests part of the Git workflow?** If yes, enforce them in CI/CD pipelines and pull request validations.

---

## Quality criteria

A strong Pulumi testing response should ensure the guidance is:

- **infrastructure-aware:** focused on provisioning, config, and drift,
- **layered:** combining unit, integration, and workflow validation,
- **mock-friendly:** using Pulumi mocks for fast, isolated tests,
- **realistic:** validating end-to-end workflows in realistic environments,
- **automated:** integrated into Git-based CI/CD pipelines,
- **compliant:** reinforced by policy and security checks.

---

## Example prompts

- "How do we unit test Pulumi infrastructure code without deploying resources?"
- "What should I validate in a Pulumi integration test for an API Gateway/Lambda/DynamoDB workflow?"
- "How do I use Pulumi Policy as Code in my CI pipeline?"
- "What are the key IaC testing practices for Pulumi-managed deployments?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-secrets
- dev-ops-infrastructure-as-code-pulumi-environments
- dev-ops-infrastructure-as-code-pulumi-workspaces
- dev-ops-infrastructure-as-code-cloud-security
- dev-ops-infrastructure-as-code-configuration-management
