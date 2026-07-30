---
name: dev-ops-infrastructure-as-code-pulumi-environments
description: Guides teams to use Pulumi Environments, Secrets, and Configurations (ESC) for secure, auditable environment and secret management.
---

# Pulumi Environments, Secrets, and Configurations (ESC)

This skill helps AI explain how Pulumi ESC streamlines environment, secret, and configuration management for cloud infrastructure and application deployments.

---

## When to use this skill

Use this skill when you need to:

- explain what Pulumi ESC is and why it matters for modern infrastructure workflows,
- describe how environments and configs are created, listed, edited, and deleted with the esc CLI,
- discuss how ESC integrates with identity, RBAC, and audit logging,
- describe supported providers and dynamic secret retrieval,
- explain how to bundle related config values into environments for repeatable deployments.

---

## Outcome

Produce guidance that:

- describes Pulumi ESC as a product for managing environment-specific config and secrets across infrastructure and applications,
- explains how ESC environments separate config from code and support multiple cloud execution contexts,
- highlights that ESC values can be static or dynamically imported from provider-managed sources,
- shows how ESC integrates with Pulumi Cloud identity, SAML IdPs, and RBAC for secure team access,
- describes how ESC can be used by Pulumi, Terraform, Cloudflare Workers, GitHub Actions, Docker, and other tools,
- encourages bundling small stories or related settings into larger environment constructs to avoid backlog and config chaos.

---

## Instructions for the AI

1. **Explain Pulumi ESC clearly**
   - Describe ESC as a centralized way to manage environments, secrets, and config values outside of Pulumi IaC code.
   - Explain that ESC extends Pulumi’s infrastructure management to include secure configuration and secret orchestration.
   - Emphasize that ESC can be used independently of Pulumi IaC and can feed other tools and runtime contexts.

2. **Describe ESC environment lifecycle commands**
   - Describe how to create environments with `esc env init [<org-name>/]<environment-name>`.
   - Explain how to list environments with `esc env ls` and filter by organization.
   - Describe setting values with `esc env set <key> <value>` and retrieving values with `esc env get <environment-name> <key>`.
   - Explain that `esc env get` returns plain static values but not resolved dynamic or secret values.
   - Describe editing an environment with `esc env edit <environment-name>` and opening it for resolution with `esc env open <environment-name>`.
   - Explain how to delete an environment with `esc env rm [<org-name>/]<environment-name>`.

3. **Explain provider support and integrations**
   - List key ESC providers such as `aws-login`, `aws-secrets`, `azure-login`, `azure-secrets`, `gcp-login`, `gcp-secrets`, `vault-login`, and `vault-secrets`.
   - Explain that these providers allow ESC to source credentials and secrets from cloud providers or Vault.
   - Emphasize that provider-managed secrets can be used safely in ESC environments without exposing them as plaintext.

4. **Connect ESC to identity and audit control**
   - Explain that Pulumi ESC integrates with Pulumi Cloud identity and RBAC.
   - Mention that it supports SAML IdPs like Azure AD, Microsoft Entra ID, Okta, and Google Workspace.
   - Advise that teams should use these integration points to enforce who can access or change environment values.

5. **Describe hierarchical composition and bundling**
   - Explain that ESC supports composing multiple environments together to reduce duplication.
   - Recommend bundling related configuration values and small secrets into environment groups for easier management.
   - Advise teams to use a single consolidated environment when sensible, and to avoid exploding the backlog into hundreds of tiny config items.

6. **Warn against common anti-patterns**
   - Warn against using ESC as a free-form secret dump with no structure or ownership.
   - Warn against creating huge environments that are hard to reason about; keep environments focused on meaningful deployment contexts.
   - Warn against bypassing RBAC or audit controls by putting sensitive values into local files or unchecked config values.

---

## Decision points and guidance

- **Is the config value environment-specific?** If yes, store it in an ESC environment rather than hard-coding it.
- **Is the value sensitive?** If yes, mark it as a secret and use provider-managed secret retrieval when possible.
- **Does this deployment need multiple cloud or tool contexts?** If yes, use ESC to share environment values across Pulumi, Terraform, GitHub Actions, Docker, etc.
- **Is the backlog filled with tiny config items?** If yes, bundle similar values into higher-level environment stories or cards.
- **Do you need auditability and identity control?** If yes, use Pulumi Cloud identity and RBAC with ESC.

---

## Quality criteria

A strong ESC response should ensure the guidance is:

- **centralized:** environment config and secrets are managed in a coherent system,
- **secure:** sensitive values are protected and resolved through provider integrations,
- **auditable:** changes are traceable through identity and RBAC,
- **composable:** environments can be layered and reused,
- **portable:** values can be consumed by multiple tools and cloud contexts,
- **pragmatic:** avoid overly large or overly fragmented environment definitions.

---

## Example prompts

- "How do I manage Pulumi environments and secrets with ESC?"
- "What is the difference between `esc env get` and `esc env open`?"
- "How can I use Pulumi ESC with Azure Key Vault and GitHub Actions?"
- "When should I bundle config values into a single ESC environment?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-secrets
- dev-ops-infrastructure-as-code-cloud-security
- dev-ops-infrastructure-as-code-configuration-management
