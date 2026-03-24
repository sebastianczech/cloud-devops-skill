# Terraform Best Practices

Source: [terraform-best-practices.com](https://www.terraform-best-practices.com/) by Anton Babenko

## Key Concepts

Terraform organizes infrastructure into four hierarchical levels:

| Level | Description |
|-------|-------------|
| **Resource** | A single cloud object (`aws_vpc`, `azurerm_resource_group`) |
| **Resource module** | A group of related resources performing a common function (e.g., VPC module) |
| **Infrastructure module** | Combines resource modules for a single logical scope (region / account / project) |
| **Composition** | Combines infrastructure modules across multiple regions or accounts |

Use **data sources** and `terraform_remote_state` as glue between modules at different levels.

## Code Structure

### File Layout

Every module and root configuration should follow this layout:

```
module/
├── main.tf          # Resources, modules, and data sources
├── variables.tf     # Input variable declarations
├── outputs.tf       # Output values
├── versions.tf      # Required Terraform and provider versions
├── locals.tf        # Local computed values
└── README.md        # Auto-generated via terraform-docs
```

`terraform.tfvars` is only used at the **composition** (root) layer — never inside reusable modules.

### Structure by Project Size

| Size | When to use | Notes |
|------|-------------|-------|
| **Small** | < 20–30 resources, single account/region/env | Single state file; use `-target` as complexity grows |
| **Medium** | Multiple accounts and environments, off-the-shelf modules | Separate state per env; harder to keep envs in sync |
| **Large** | Many accounts, many regions, internal modules | Consider Terragrunt for orchestration |

### State Management

- Always use remote state from day one — never store `.tfstate` in version control
- Enable state locking to prevent concurrent modifications
- Use one state file per environment (dev / staging / prod)
- Use separate state files for separate concerns (network, compute, data)
- Never store secrets in state — mark sensitive outputs with `sensitive = true`

#### Azure Backend
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "st<project>tfstate"
    container_name       = "tfstate"
    key                  = "<project>/<env>/terraform.tfstate"
  }
}
```

#### AWS Backend
```hcl
terraform {
  backend "s3" {
    bucket         = "tf-state-<account-id>"
    key            = "<project>/<env>/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tf-state-lock"
  }
}
```

### Orchestration

For managing dependencies between root modules, prefer:
- **Terragrunt** — dedicated orchestration, eliminates repetition, scales well
- **Atlantis** — pull-request automation for `plan` and `apply`
- Native Terraform with scripting for simple cases

## Naming Conventions

### General Rules

- Use `_` (underscore) everywhere in Terraform identifiers — resource names, variable names, outputs, locals
- Use lowercase letters and numbers only
- Use singular nouns for resource names
- Use dashes (`-`) in actual argument values (DNS names, tags, resource name strings) — not in Terraform identifiers

### Resources and Data Sources

- Never repeat or partially include the resource type in the resource name
  - `resource "aws_route_table" "public"` ✓
  - `resource "aws_route_table" "public_route_table"` ✗
- Use `this` only when a module has a single resource of that type and no better name exists
- Use descriptive names (`private`, `public`, `database`) when multiple resources of the same type exist

### Argument Ordering

Order arguments inside a resource block:
1. `count` or `for_each` — first, followed by a blank line
2. Resource-specific arguments
3. `tags` — last configuration argument (if supported)
4. `depends_on` — after a blank line (if needed)
5. `lifecycle` — after a blank line (if needed)

Use boolean expressions in `count`/`for_each` conditions:
```hcl
# Preferred
count = var.create_public_subnets ? 1 : 0

# Avoid
count = length(var.public_subnets) > 0 ? 1 : 0
```

### Variables

- Always include `description` — even if it seems obvious
- Use plural names for `list(...)` and `map(...)` types
- Order variable block keys: `description`, `type`, `default`, `validation`
- Prefer simple types (`string`, `number`, `list(...)`, `map(...)`) over `object()` unless strict validation is required
- Use `any` to disable type validation at a given depth when supporting multiple types
- Set `nullable = false` for variables that must never be null
- Avoid double negatives — use `encryption_enabled` not `encryption_disabled`

```hcl
variable "allowed_cidrs" {
  description = "List of CIDR blocks allowed to access the resource"
  type        = list(string)
  default     = []
  nullable    = false

  validation {
    condition     = alltrue([for c in var.allowed_cidrs : can(cidrnetmask(c))])
    error_message = "All entries must be valid CIDR blocks."
  }
}
```

### Outputs

- Follow naming pattern: `{name}_{type}_{attribute}`
  - `name` = resource identifier (e.g., `private` for `data "aws_subnet" "private"`)
  - `type` = resource type without provider prefix (e.g., `subnet`)
  - `attribute` = returned property (e.g., `id`, `arn`)
- Always include `description` for every output
- Use plural names when returning lists
- Use `try()` instead of the legacy `element(concat(...))` pattern
- Apply `sensitive = true` only to outputs you fully control end-to-end

```hcl
output "private_subnet_id" {
  description = "ID of the private subnet"
  value       = aws_subnet.private.id
}
```

## Code Styling

- Run `terraform fmt` on every file — integrate with pre-commit hooks
- Use `terraform fmt -check` in CI/CD to fail on unformatted code
- Use `#` for comments — avoid `//` and block comments
- Use `.editorconfig` for consistent indentation (2 spaces for `.tf` files)
- Generate documentation automatically with `terraform-docs` via pre-commit hooks
- README links must be absolute paths (required for Terraform Registry display)

### Recommended Pre-commit Hooks

```yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_docs
      - id: terraform_tflint
      - id: terraform_checkov
```

## Writing Configurations

### Locals for Explicit Dependencies

Use `locals` to model explicit dependencies between resources when no implicit dependency exists (Terraform uses them to determine destroy order):

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
    Owner       = var.owner
  }
}
```

### Modern Optional Attributes (Terraform 1.3+)

Use `optional()` in object types instead of complex `dynamic` blocks:

```hcl
variable "website" {
  type = object({
    index_document = string
    error_document = optional(string, "error.html")
  })
}
```

### Secret Management

- Never commit sensitive values to version control
- Use external data sources to fetch secrets at runtime (AWS Secrets Manager, Azure Key Vault)
- Terraform 1.11+ supports write-only arguments for secrets that are never stored in state

### Environment Separation

- Prefer **separate directories** per environment for strict isolation and clear blast radius
- Use `terraform.workspace` only for simple cases — it does not provide state isolation between workspaces in the same backend

## Testing and Validation

| Tool | Purpose |
|------|---------|
| `terraform validate` | Syntax and internal consistency |
| `terraform plan` | Diff review — always review before apply |
| `tflint` | Linting and provider-specific rule checks |
| Checkov | Security and compliance scanning (IaC) |
| tfsec | Security scanning (alternative to Checkov) |
| Terratest | Go-based integration tests for modules |
| Infracost | Cost estimates in pull requests |

## Recommended Toolchain

| Tool | Purpose |
|------|---------|
| Terragrunt | Orchestration and DRY wrapper for Terraform |
| tflint | Code linter |
| tfenv / asdf | Terraform version manager |
| Atlantis | Pull request automation (`plan` / `apply`) |
| pre-commit-terraform | Git hooks for fmt, validate, docs, security |
| Infracost | Cost estimation in CI/CD |

## Anti-Patterns to Avoid

- Hardcoded credentials, account IDs, or region names in code
- Using `count` for resources that differ significantly — use `for_each` instead
- Large monolithic root modules — split by lifecycle and blast radius
- Skipping `terraform plan` review before `apply`
- Storing `.tfstate` files in version control
- Managing multiple environments with a single state file
- Using `latest` or unpinned provider versions in production
- Repeating the resource type in the resource name
- Double-negative variable names (`disable_x`, `no_x`)
