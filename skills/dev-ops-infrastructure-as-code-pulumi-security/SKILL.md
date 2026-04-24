---
name: dev-ops-infrastructure-as-code-pulumi-security
description: Guides teams to implement Secure by Design infrastructure using Pulumi and TypeScript, focusing on IAM, policy-as-code, resource controls, and secure deployment practices.
---

# Pulumi Security and IAM Best Practices

This skill helps AI explain how to build secure infrastructure with Pulumi by codifying IAM, access controls, least privilege, encryption, policy checks, and secure operational practices. It includes TypeScript examples for AWS IAM and policy enforcement.

---

## When to use this skill

Use this skill when you need to:

- explain Secure by Design principles in Pulumi IaC,
- define IAM roles, policies, RBAC, and ABAC in code,
- implement least-privilege access controls for cloud resources,
- configure secret handling, encryption, and MFA,
- enforce security policies and auditing through Pulumi.

---

## Outcome

Produce guidance that:

- describes how Pulumi codifies IAM and access controls,
- shows how to apply least-privilege and defence-in-depth,
- includes TypeScript examples of IAM roles and policies,
- explains the use of policy-as-code and security validation,
- highlights secure registry, secret, and key management.

---

## Instructions for the AI

1. **Explain Secure by Design**
   - Define Secure by Design as embedding security through architecture, deployment, and operations, not bolting it on after the fact.
   - Emphasize that Pulumi’s programmatic model supports security as code and repeatable enforcement.

2. **Describe IAM fundamentals**
   - Explain authentication vs authorization.
   - Define least privilege, role separation, and the need for narrow access scopes.
   - Mention MFA and access review as complementary controls.

3. **Provide an IAM example**
   - Show an AWS IAM role and inline policy in TypeScript.

```typescript
import * as aws from "@pulumi/aws";

const readOnlyPolicy = new aws.iam.Policy("readOnlyPolicy", {
    description: "Read-only access to S3 and CloudWatch",
    policy: JSON.stringify({
        Version: "2012-10-17",
        Statement: [
            {
                Action: ["s3:GetObject", "s3:ListBucket"],
                Effect: "Allow",
                Resource: ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
            },
            {
                Action: ["cloudwatch:GetMetricData", "cloudwatch:ListMetrics"],
                Effect: "Allow",
                Resource: "*",
            },
        ],
    }),
});

const role = new aws.iam.Role("readOnlyRole", {
    assumeRolePolicy: aws.iam.assumeRolePolicyForPrincipal({
        Service: "ec2.amazonaws.com",
    }),
});

new aws.iam.RolePolicyAttachment("readOnlyRoleAttachment", {
    role: role.name,
    policyArn: readOnlyPolicy.arn,
});
```

4. **Explain resource access controls**
   - Describe using security groups/NSGs and IAM together.
   - Explain how network controls protect resources even when IAM is correct.

5. **Show policy-as-code enforcement**
   - Mention Pulumi Policy Packs or built-in policy engines.
   - Show a simple TypeScript policy example.

```typescript
import * as policy from "@pulumi/policy";

new policy.PolicyPack("securityPack", {
    policies: [
        {
            name: "no-public-s3",
            description: "Disallow public S3 buckets",
            enforcementLevel: "mandatory",
            validateResource: policy.validateResourceOfType(aws.s3.Bucket, (bucket, args, reportViolation) => {
                if (bucket.acl === "public-read" || bucket.acl === "public-read-write") {
                    reportViolation("S3 bucket should not be public.");
                }
            }),
        },
    ],
});
```

6. **Cover secrets and encryption**
   - Explain `pulumi.Config().requireSecret()` for sensitive values.
   - Show how to use provider-managed keys or KMS-managed encryption.

```typescript
const config = new pulumi.Config();
const dbPassword = config.requireSecret("dbPassword");
const bucket = new aws.s3.Bucket("secureBucket", {
    serverSideEncryptionConfiguration: {
        rule: {
            applyServerSideEncryptionByDefault: {
                sseAlgorithm: "aws:kms",
            },
        },
    },
});
```

7. **Discuss auditing and review**
   - Recommend regular IAM policy reviews, access logs, and audit trails.
   - Recommend using cloud-native audit services like AWS CloudTrail or Azure Activity Log.

8. **Warn about common pitfalls**
   - Avoid broad `*` permissions.
   - Avoid hardcoding secrets in code and Docker images.
   - Avoid defining overly permissive security groups.

---

## Decision points and guidance

- **Does the resource need direct cloud access?** If yes, use narrowly scoped IAM roles and only the permissions required.
- **Should the role be reusable?** If yes, define a role and attach policies rather than inline permissions.
- **Is this a security-critical resource?** If yes, enforce additional controls like MFA, encryption, and auditing.
- **Can the policy be automated?** If yes, use policy-as-code to prevent insecure deployments.

---

## Quality criteria

A strong response should ensure the guidance is:

- **secure by default:** default to least privilege and explicit deny patterns,
- **code-driven:** show how IAM and policies are expressed in Pulumi TypeScript,
- **defense-in-depth:** combine IAM, network controls, and encryption,
- **auditable:** encourage logging, reviews, and policy enforcement,
- **operational:** practical for real cloud deployments.

---

## Example prompts

- "How do I codify AWS IAM roles and policies in Pulumi TypeScript?"
- "What does Secure by Design look like for Pulumi infrastructure?"
- "How do I enforce no-public-s3 bucket policies with Pulumi?"
- "How should I manage secrets and KMS encryption in Pulumi?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-secrets
- dev-ops-infrastructure-as-code-pulumi-stack-configurations
- dev-ops-infrastructure-as-code-pulumi-testing
