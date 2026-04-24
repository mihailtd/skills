---
name: dev-ops-infrastructure-as-code-pulumi-secrets
description: Guides secure Pulumi secrets and configuration management practices for infrastructure and application deployments.
---

# Pulumi Secrets and Configuration Management

This skill helps AI explain how Pulumi handles configuration and secrets, why separating configuration from code matters, and how to use secure provider integrations to keep sensitive values protected.

---

## When to use this skill

Use this skill when you need to:

- describe Pulumi configuration management and why it improves consistency, scalability, and flexibility,
- explain how Pulumi separates config from code and supports environment-specific parameters,
- define how Pulumi secrets differ from plain configuration values,
- advise on secure secret storage, access control, and runtime retrieval,
- connect Pulumi secrets to cloud provider vaults and managed secret systems.

---

## Outcome

Produce guidance that:

- explains configuration management as settings and environment-specific behavior separate from application or infrastructure code,
- shows how Pulumi config values can come from environment variables, config files, or cloud-native sources,
- emphasizes why secrets must be encrypted and access-controlled in Pulumi,
- describes Pulumi secret providers and integration patterns for AWS KMS, Azure Key Vault, Google Cloud KMS, Vault, and external secret stores,
- recommends dynamic secret handling when deployments need runtime-sensitive credentials,
- highlights auditability, version control, and traceability for config and secret changes.

---

## Instructions for the AI

1. **Describe Pulumi configuration management clearly**
   - Explain that Pulumi allows settings, variables, and environment-specific parameters to be managed separate from infrastructure code.
   - Emphasize that this separation makes deployments consistent, portable, and easier to maintain.
   - Explain that Pulumi configuration can be defined across multiple environments without changing the program.

2. **Explain what Pulumi secrets are**
   - Define secrets as sensitive values such as passwords, API keys, certificates, and tokens that should never be stored in plaintext.
   - Explain that Pulumi marks secrets in config so they are encrypted in storage and kept safe in transit.
   - Clarify that a Pulumi secret is used like a normal config value inside programs but is protected by Pulumi’s secret handling.

3. **Describe secure secret storage and provider integration**
   - Recommend using managed secret stores when possible, such as AWS Secrets Manager, Azure Key Vault, Google Cloud Secret Manager, HashiCorp Vault, or 1Password.
   - Explain that Pulumi can integrate with these providers to fetch secrets dynamically during deployments.
   - Advise that provider-managed secrets should be consumed through configuration abstractions rather than embedded directly into code.

4. **Talk about dynamic secret retrieval**
   - Explain that Pulumi programs can resolve secrets at runtime from external sources when needed.
   - Highlight that dynamic secrets are valuable because they can be short-lived and minimize credential exposure.
   - Note that not all systems support dynamic secrets, so Pulumi ESC or provider plugins may be needed to bridge the gap.

5. **Encourage strong access control and auditing**
   - Recommend restricting who can read and modify Pulumi config and secrets.
   - Explain that Pulumi ESC integrates with identity and RBAC systems so environment and secret changes can be audited.
   - Advise using version control for config definitions and tracking changes to infrastructure parameters as audit evidence.

6. **Warn about common mistakes**
   - Warn against storing secrets in plaintext files, source control, or chat logs.
   - Warn against over-splitting or duplicating secret config across environments.
   - Advise against treating a secret-managed environment as a pass-through for insecure values.

---

## Decision points and guidance

- **Is the value sensitive?** If yes, treat it as a secret and protect it with encryption and access controls.
- **Is this configuration environment-specific?** If yes, keep it in Pulumi config or ESC rather than hard-coding it.
- **Is the secret used by multiple deployments or applications?** If yes, prefer a managed secret provider and share it securely.
- **Do you need runtime secret resolution?** If yes, use Pulumi ESC or provider plugins for dynamic secret retrieval.
- **Are config changes auditable?** If no, add identity-backed access controls and logging around environment or secret edits.

---

## Quality criteria

A strong response should ensure the guidance is:

- **secure:** secrets are encrypted, access-controlled, and not stored in plaintext,
- **separate:** configuration is managed independently from code,
- **portable:** environment differences are handled through config rather than code changes,
- **integrated:** secret management uses provider-native or ESC integrations,
- **auditable:** changes to configuration and secrets are tracked and reviewable.

---

## Example prompts

- "How should we manage Pulumi secrets so they stay out of source control?"
- "What’s the difference between Pulumi config and Pulumi secret config?"
- "How can Pulumi integrate with Azure Key Vault and AWS Secrets Manager?"
- "When should we use dynamic secrets in a Pulumi deployment?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-environments
- dev-ops-infrastructure-as-code-configuration-management
- dev-ops-infrastructure-as-code-cloud-security
