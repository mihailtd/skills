---
name: dev-ops-infrastructure-as-code-pulumi-cicd
description: Guides teams to integrate Pulumi with CI/CD systems like GitHub Actions, GitLab CI/CD, and Jenkins, including deployment strategies, rollback, versioning, and best practices.
---

# Pulumi CI/CD Integration

This skill helps AI explain how Pulumi fits into modern CI/CD pipelines for infrastructure as code. It covers the role of CI/CD in infrastructure automation, integration with Jenkins/GitLab/GitHub Actions, pipeline examples, deployment strategies, rollback handling, versioning, testing, and security best practices.

---

## When to use this skill

Use this skill when you need to:

- explain how CI/CD accelerates Pulumi-based IaC delivery,
- show concrete pipeline examples for GitHub Actions, GitLab CI, and Jenkins,
- describe deployment strategies such as blue-green, canary, and rolling updates,
- explain rollback mechanisms and versioning in Pulumi CI/CD,
- highlight pipeline best practices for repeatable, secure infrastructure deployments.

---

## Outcome

Produce guidance that:

- defines the role of CI/CD in infrastructure automation,
- maps Pulumi commands to CI/CD stages,
- shows how to authenticate Pulumi and cloud providers in pipelines,
- includes sample YAML pipeline configurations,
- explains deployment strategy patterns and rollback controls,
- defines Git-based versioning and visibility best practices.

---

## Instructions for the AI

1. **Explain CI/CD for Pulumi IaC**
   - Describe CI/CD as automation for build/test/deploy stages, now extended to infrastructure.
   - Explain that Pulumi programs are source code and that CI/CD pipelines can validate, preview, and publish infrastructure changes.
   - Emphasize benefits: speed, consistency, traceability, and repeatable deployments.

2. **Describe the CI/CD role in infrastructure automation**
   - Explain that CI validates IaC changes before deployment.
   - Explain that CD automates safe deployment of infrastructure state changes.
   - Mention that infrastructure automation pipelines make provisioning more reliable than manual console actions.

3. **List Pulumi CI/CD benefits**
   - Speed and efficiency: automated pipeline execution and repeatable deployments.
   - Reliability and consistency: preview and policy checks catch issues early.
   - Visibility and traceability: pipeline logs, stack history, and Git commits create audit trails.
   - Scalability and agility: pipelines support multi-environment and multi-cloud workflows.

4. **Show GitLab CI/CD example**
   - Provide a complete `.gitlab-ci.yml` job for login, stack selection, preview, and up.
   - Explain secrets handling through CI/CD environment variables.

```yaml
stages:
  - validate
  - deploy

validate:
  stage: validate
  image: pulumi/pulumi:latest
  script:
    - pulumi login --cloud-url "$PULUMI_CLOUD_URL" --secrets-provider="passphrase"
    - pulumi stack select "$PULUMI_STACK"
    - pulumi preview --diff
  only:
    - merge_requests

deploy:
  stage: deploy
  image: pulumi/pulumi:latest
  script:
    - pulumi login --cloud-url "$PULUMI_CLOUD_URL" --secrets-provider="passphrase"
    - pulumi stack select "$PULUMI_STACK"
    - pulumi up --yes
  environment:
    name: production
  only:
    - main
```

5. **Show GitHub Actions example**
   - Provide a workflow with checkout, Pulumi login, preview, and apply.
   - Mention GitHub Secrets for credentials.

```yaml
name: pulumi-deploy
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pulumi/actions@v5
        with:
          command: preview
          stack-name: org/project/main
          work-dir: ./infra
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

  deploy:
    needs: preview
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: pulumi/actions@v5
        with:
          command: up
          stack-name: org/project/main
          work-dir: ./infra
          yes: true
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

6. **Show Jenkins example**
   - Provide a Declarative Jenkins pipeline snippet with Pulumi CLI and environment variables.

```groovy
pipeline {
  agent any
  environment {
    PULUMI_ACCESS_TOKEN = credentials('pulumi-token')
    AWS_ACCESS_KEY_ID = credentials('aws-access-key')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Pulumi Preview') {
      steps {
        sh 'pulumi login --cloud-url $PULUMI_CLOUD_URL --secrets-provider=passphrase'
        sh 'pulumi stack select $PULUMI_STACK'
        sh 'pulumi preview --diff'
      }
    }
    stage('Pulumi Deploy') {
      when {
        branch 'main'
      }
      steps {
        sh 'pulumi up --yes'
      }
    }
  }
}
```

7. **Explain authentication and secrets management**
   - Recommend storing `PULUMI_ACCESS_TOKEN`, cloud credentials, and stack secrets in CI/CD secret stores.
   - Explain that Pulumi secrets should be handled with `pulumi config set --secret` or secret providers.
   - Mention service principal and vault integration for cloud provider authentication.

8. **Describe deployment strategies**
   - Blue-green: use separate Pulumi stacks for blue and green environments, deploy the new stack, verify, and switch traffic.
   - Canary: deploy to a small subset or separate stack, monitor, then promote.
   - Rolling updates: use Pulumi-managed compute or orchestration features and incrementally update resources.

9. **Show blue-green deployment pattern**
   - Explain using two stacks or two sets of infrastructure resources.

```yaml
blue-deploy:
  script:
    - pulumi stack select org/project/blue
    - pulumi up --yes

green-deploy:
  script:
    - pulumi stack select org/project/green
    - pulumi up --yes
```

10. **Show rollback and versioning strategies**
    - Explain that rollback can be triggered by `pulumi stack export`/`pulumi stack import` or `pulumi destroy` as a last resort.
    - Mention using stack history and Git tags to track versions.
    - Example:

```bash
pulumi stack export --stack org/project/main > previous-state.json
# ... after failed deployment
pulumi stack import --stack org/project/main < previous-state.json
```

11. **Discuss validation and policy checks**
    - Recommend using `pulumi preview --diff` as a gating step.
    - Suggest Pulumi Policy as Code or `pulumi policy check` for compliance validation.
    - Mention unit testing of TypeScript infrastructure code and integration tests using the Pulumi Automation API.

12. **List CI/CD best practices**
    - Keep pipeline definitions in Git and version them with the infrastructure code.
    - Run preview stages on merge requests before applying.
    - Separate validation from deployment.
    - Use stack-specific variables for `dev`, `staging`, and `prod`.
    - Keep secrets out of source control.
    - Log outputs and export deployment artifacts when needed.
    - Use lightweight checks before expensive cloud actions.
    - Monitor deployment results and rollback fast if necessary.

13. **Explain governance and monitoring**
    - Explain how pipeline logs and Pulumi stack history provide traceability.
    - Recommend attaching notifications or alerts for failed deployments.
    - Advise making cloud drift detection part of periodic pipeline checks.

14. **Include a concise conclusion**
    - Summarize Pulumi as a strong IaC tool for CI/CD automation.
    - Reinforce that automated pipelines improve reliability, consistency, and collaboration.

---

## Decision points and guidance

- **Does your workflow need automated infrastructure validation?** Yes — add `preview` or `pulumi policy check` stages.
- **Are you deploying to multiple environments?** Yes — use separate stacks and pipeline branches.
- **Do you need rollback safety?** Yes — capture stack snapshots and tag releases.
- **Is the pipeline managing secrets?** Yes — use CI/CD secrets and Pulumi secret providers.
- **Is the deployment path manual or automated?** Prefer automation with gated previews and separate deploy stages.

---

## Quality criteria

A strong response should ensure the guidance is:

- **actionable:** includes concrete CI/CD pipeline examples,
- **secure:** avoids leaking credentials and uses Pulumi secrets,
- **repeatable:** encourages stack-specific, versioned deployments,
- **safe:** uses preview, validation, and rollback controls,
- **governed:** provides traceability, monitoring, and compliance checks.

---

## Example prompts

- "How can I deploy Pulumi infrastructure with GitHub Actions?"
- "What should a GitLab CI pipeline for Pulumi look like?"
- "How do I add rollback support to a Pulumi deployment pipeline?"
- "Should I use separate Pulumi stacks for blue-green deployments?"

---

## Related guidance

Use this tool alongside:

- dev-ops-infrastructure-as-code-pulumi-stack-configurations
- dev-ops-infrastructure-as-code-pulumi-testing
- dev-ops-infrastructure-as-code-pulumi-workspaces
- dev-ops-infrastructure-as-code-pulumi-secrets
