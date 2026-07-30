---
name: code-review-security
description: Guides AI reviewers to evaluate code for security posture, threat-resistant design, and secure development lifecycle concerns.
---

# Code Review: Security

This skill helps AI reviewers identify security risks in code and architecture by
applying security design principles, attack surface reduction, and secure
lifecycle discipline.

---

## When to use this skill

Use this skill when you need to:

- review code for security vulnerabilities and insecure design,
- validate access control, authentication, and authorization logic,
- evaluate secrecy, integrity, and availability trade-offs,
- ensure security is built in, not bolted on,
- combine automated findings with manual security reasoning.

---

## Outcome

Produce a security review that:

- identifies high-risk vulnerabilities and weak design choices,
- aligns results with CIA/PAIN security principles,
- recommends concrete mitigations for insecure behavior,
- confirms that security controls are consistently enforced,
- highlights where security design should be improved earlier in the lifecycle.

---

## Instructions for the AI

1. **Apply the CIA triad and PAIN model**
   - Confidentiality: ensure sensitive data is protected and not leaked.
   - Integrity: ensure data and actions cannot be tampered with.
   - Availability: ensure security controls do not unnecessarily block authorized use.
   - Privacy, Authentication/Availability, Integrity, Non-repudiation: use these to assess security controls holistically.

2. **Enforce security by design**
   - Treat security as a fundamental design requirement from the start.
   - Reject fixes that simply bolt security onto a completed design.
   - Prefer solutions that reduce attack surface and minimize privileged scope.

3. **Check core security principles**
   - Least Privilege: verify that each actor and component only has the access it needs.
   - Defense in Depth: recommend layered protections rather than single-point defenses.
   - Segregation of Duties: ensure critical capabilities are separated when appropriate.
   - Fail Safe: confirm failures default to a secure state.
   - Complete Mediation: ensure every access request is authorized consistently.
   - Least Common Mechanism: minimize shared paths that can leak information.
   - Weakest Link: identify the most vulnerable part of the chain and harden it.

4. **Balance automated and manual review**
   - Treat SAST as a first pass for common patterns like injection, XSS, or unsafe crypto.
   - Use manual review for logic flaws, IAM, session handling, and error-path security.
   - Provide context-aware guidance that automated tools may miss.

5. **Inspect security-critical areas**
   - Data handling: sensitive data in logs, leaks, insecure transport, and crash-state exposure.
   - Access and identity: authentication, authorization, session management, and privilege boundaries.
   - Cryptography: key management, cipher selection, certificate handling, and hashing.
   - Common vulnerabilities: hard-coded secrets, injection, insecure deserialization, and hidden backdoors.
   - Compliance and design: security-by-obscurity, attack surface minimization, and policy alignment.

6. **Require secure defaults**
   - Prefer deny-by-default access models.
   - Ensure configuration and deployment defaults do not expose services or data.
   - Validate that explicit opt-in is required for elevated or insecure behavior.

7. **Provide actionable security feedback**
   - Use clear risk categories: low, medium, high, critical.
   - Recommend concrete remediation steps, not just generic warnings.
   - Link findings to relevant security principles and why they matter.

---

## Decision points and guidance

- **Does the code leak sensitive data?** If yes, identify where and how to stop it.
- **Are access checks enforced everywhere?** If not, find the gaps.
- **Does the design rely on secrecy of implementation?** If yes, call it out as security by obscurity.
- **Is the weakest link a human or a technical dependency?** If yes, recommend hardening that link.
- **Is authentication and authorization separated and well-scoped?** If no, recommend clearer boundaries.
- **Are cryptographic choices current and correctly applied?** If no, suggest standard algorithms and proper key handling.
- **Are security controls compatible with availability and usability?** If not, recommend safer trade-offs.

---

## Quality criteria

A strong security review should confirm that the code is:

- **confidential:** sensitive data is protected in transit, at rest, and in logs,
- **integrity-preserving:** tampering is detectable and unauthorized changes are prevented,
- **available:** security controls do not unduly block legitimate users,
- **least-privileged:** privileges are scoped minimally,
- **layered:** defenses exist in multiple places,
- **fail-safe:** failures favor secure denial over unsafe continuation,
- **mediated:** every access decision is checked,
- **transparent:** security design is explicit and easy to reason about.

---

## Security review checklist

Use these questions during the review:

- [ ] Are sensitive values excluded from logs and error output?
- [ ] Is access control enforced consistently across all code paths?
- [ ] Are authentication and authorization separated and clearly defined?
- [ ] Are secrets and keys managed securely, not hard-coded?
- [ ] Is data protected during storage and transport with strong crypto?
- [ ] Are input validation and output encoding applied where needed?
- [ ] Does the code avoid security by obscurity?
- [ ] Are failure cases handled safely, not permissively?
- [ ] Is the attack surface minimized by removing unused code and endpoints?
- [ ] Does the review identify the weakest link and suggest hardening it?

---

## Example prompts

- "Review this code for security risks and tell me whether the authorization flow is safe."
- "Explain where this module fails to protect confidential data and how to fix it."
- "Evaluate whether the current authentication and access controls follow least privilege."

---

## Related guidance

This skill complements:

- code-review-code-structure
- code-review-data-structures
- code-review-detect-bad-design
- code-review-architecture
- code-review-concurrency-parallelism-performance