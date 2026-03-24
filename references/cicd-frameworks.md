# CI/CD Pipeline Frameworks

## Pipeline Stages (Standard)

1. **Source** — trigger on push/PR to relevant branches
2. **Lint & Validate** — fast feedback: linting, static analysis, IaC validation
3. **Test** — unit tests, integration tests
4. **Build** — container image build and push to registry
5. **Deploy to Dev/Staging** — automated on merge to main
6. **Integration / E2E Tests** — smoke tests against deployed environment
7. **Deploy to Production** — gated by approval or automated on passing tests

## GitHub Actions

### Reusable Workflow Pattern
```yaml
# .github/workflows/deploy.yml
on:
  push:
    branches: [main]

jobs:
  build:
    uses: ./.github/workflows/_build.yml
    with:
      environment: production
    secrets: inherit

  deploy:
    needs: build
    uses: ./.github/workflows/_deploy.yml
    with:
      environment: production
    secrets: inherit
```

### Environment Protection Rules
- Require reviewers for production deployments
- Add deployment branch policy (only `main`)
- Use environment secrets scoped to each environment

## Azure Pipelines

### Multi-Stage Pipeline
```yaml
stages:
  - stage: Build
    jobs:
      - job: BuildAndPush
        steps:
          - task: Docker@2
            inputs:
              command: buildAndPush

  - stage: DeployDev
    dependsOn: Build
    environment: dev

  - stage: DeployProd
    dependsOn: DeployDev
    environment: prod  # triggers approval gate
```

## AWS CodePipeline

- **Source** → CodeCommit / GitHub / S3
- **Build** → CodeBuild (runs buildspec.yml)
- **Deploy** → CodeDeploy / ECS / EKS / CloudFormation

## Container Image Best Practices

- Pin base image versions — avoid `latest`
- Multi-stage builds to minimize image size
- Scan images with Trivy or Snyk before push
- Sign images with cosign for supply chain security

## Deployment Environments

| Environment | Trigger | Approval |
|-------------|---------|---------|
| dev | Every PR merge | Automatic |
| staging | Tag / release branch | Automatic |
| production | Tag / release branch | Manual approval |

## Secrets Management

- GitHub Actions → GitHub Secrets or OIDC + cloud vault
- Azure Pipelines → Azure Key Vault task or variable groups
- AWS → AWS Secrets Manager + IAM role via OIDC
- Never store secrets in pipeline YAML files

## OIDC Federation (Keyless Auth)

Prefer OIDC over long-lived credentials:
- GitHub Actions → Azure: Workload Identity Federation
- GitHub Actions → AWS: IAM OIDC provider + assume role
