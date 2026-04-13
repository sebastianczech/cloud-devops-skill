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

> **Note:** `_build.yml` and `_deploy.yml` must each define `on: workflow_call` as their trigger to be callable from another workflow. Without that, the `uses:` reference will fail at runtime.

```yaml
# .github/workflows/_build.yml  (reusable workflow — must define on: workflow_call)
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # ... build steps
```

```yaml
# .github/workflows/deploy.yml  (caller workflow)
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

### PR Title Enforcement (Conventional Commits)

Enforce semantic PR titles on every pull request using `amannn/action-semantic-pull-request`. This is the prerequisite for automated changelog and version bump generation — every PR must be typed (`feat:`, `fix:`, `chore:`, etc.) before it can merge:

```yaml
# .github/workflows/lint-pr.yml
on:
  pull_request_target:
    types: [opened, edited, synchronize, reopened, ready_for_review]
permissions:
  pull-requests: read
jobs:
  main:
    runs-on: ubuntu-latest
    steps:
      - uses: amannn/action-semantic-pull-request@v5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Multi-Version Terraform Testing

For Terraform module repos, test against both the **minimum and maximum** versions declared in `versions.tf`. Use `clowdhaus/terraform-min-max` to detect these constraints automatically, then run pre-commit hooks (tflint, terraform-docs, Trivy, etc.) against each version in a matrix job. This catches regressions introduced at version boundaries:

```yaml
jobs:
  collectInputs:
    outputs:
      directories: ${{ steps.dirs.outputs.directories }}
    steps:
      - uses: actions/checkout@v4
      - id: dirs
        uses: clowdhaus/terraform-composite-actions/directories@v1.9.0

  preCommitMinVersions:
    needs: collectInputs
    strategy:
      matrix:
        directory: ${{ fromJson(needs.collectInputs.outputs.directories) }}
    steps:
      - uses: actions/checkout@v4
      - id: minMax
        uses: clowdhaus/terraform-min-max@v1.3.1
        with:
          directory: ${{ matrix.directory }}
      - uses: clowdhaus/terraform-composite-actions/pre-commit@v1.9.0
        with:
          terraform-version: ${{ steps.minMax.outputs.minVersion }}

  preCommitMaxVersion:
    needs: collectInputs
    steps:
      - uses: actions/checkout@v4
      - id: minMax
        uses: clowdhaus/terraform-min-max@v1.3.1
      - uses: clowdhaus/terraform-composite-actions/pre-commit@v1.9.0
        with:
          terraform-version: ${{ steps.minMax.outputs.maxVersion }}
```

### Super-Linter (Multi-Scanner in One Step)

`super-linter/super-linter` bundles many scanners into a single action. Toggle individual scanners via repository variables (`Settings → Variables → Actions`) so teams can enable/disable scanners without changing workflow YAML:

```yaml
- uses: super-linter/super-linter@v7.1.0
  env:
    VALIDATE_CHECKOV: ${{ vars.VALIDATE_CHECKOV }}      # IaC security
    VALIDATE_GITLEAKS: ${{ vars.VALIDATE_GITLEAKS }}    # secret detection
    VALIDATE_YAML: ${{ vars.VALIDATE_YAML }}            # YAML syntax
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Owner-Based Workflow Gating

For workflows that should only run in the upstream repo (not forks), gate on `github.repository_owner`. This prevents accidental releases or upgrades triggered from forked PRs:

```yaml
jobs:
  release:
    if: github.repository_owner == 'your-org-or-username'
    runs-on: ubuntu-latest
    # ...
```

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
    environment: prod  # can require approval if the 'prod' environment is configured with checks/approvals
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

## Automated Semantic Releases

Use `cycjimmy/semantic-release-action` together with conventional commits to fully automate versioning, changelog generation, and GitHub releases. Requires PR title enforcement (above) to be in place first:

```yaml
# .github/workflows/release.yml
on:
  workflow_dispatch:  # trigger manually or on merge to main

jobs:
  release:
    runs-on: ubuntu-latest
    if: github.repository_owner == 'your-org-or-username'
    permissions:
      contents: write
      issues: write
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
          fetch-depth: 0
      - uses: cycjimmy/semantic-release-action@v4
        with:
          semantic_version: 24.0.0
          extra_plugins: |
            @semantic-release/changelog@6.0.3
            @semantic-release/git@10.0.1
            @semantic-release/release-notes-generator@10.0.0
            conventional-changelog-conventionalcommits@8.0.0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Commit type → version bump mapping: `fix:` → patch, `feat:` → minor, `feat!:` / `BREAKING CHANGE:` → major.

## Automated Dependency Upgrades

Run pre-commit hooks across all files on a schedule or manual trigger. If any hook rewrites a file (e.g., terraform-docs regenerates a README), open a draft PR automatically for human review:

```yaml
# .github/workflows/upgrade-dependencies.yml
on:
  workflow_dispatch:

jobs:
  upgrade:
    runs-on: ubuntu-latest
    if: github.repository_owner == 'your-org-or-username'
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-python@v2
        with:
          python-version: '3.12'
      - run: pip install pre-commit

      # install tools pre-commit hooks rely on (tflint, terraform-docs, trivy, terrascan, etc.)

      - name: Run pre-commit
        id: precommit
        run: pre-commit run --all-files

      - name: Create PR if pre-commit made changes
        if: ${{ failure() && steps.precommit.conclusion == 'failure' }}
        uses: peter-evans/create-pull-request@v7
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          commit-message: "chore: Upgrade dependencies"
          title: "chore: Upgrade dependencies"
          branch: upgrade-dependencies
          base: main
          labels: dependencies
          draft: always-true
```

> **Note:** `pre-commit run --all-files` exits non-zero when it rewrites any file. The `create-pull-request` step only fires on that failure, so the PR is only created when there are actual changes.
