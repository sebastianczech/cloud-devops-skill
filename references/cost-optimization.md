# Cost Optimization

## Principles

1. **Right-size** — match instance/node sizes to actual workload requirements
2. **Elasticity** — scale down when demand drops (HPA, cluster auto-scaler)
3. **Spot/Preemptible** — use for fault-tolerant and batch workloads (50-90% savings)
4. **Commitments** — use Reserved Instances or Savings Plans for predictable baseline

## Kubernetes Cost Controls

- Set resource requests and limits on all pods
- Use LimitRange per namespace to enforce defaults
- Enable Vertical Pod Autoscaler (VPA) recommendations to right-size requests
- Use Horizontal Pod Autoscaler to scale down during off-peak
- Schedule scale-down for non-production clusters outside business hours

### Namespace LimitRange Example
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
    - default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      type: Container
```

## Infrastructure Cost Controls

- Delete unused resources: unattached disks, idle load balancers, unused IPs
- Use lifecycle policies on object storage to tier or delete old data
- Enable auto-shutdown for dev/test environments (nights, weekends)
- Use cloud provider cost anomaly detection alerts

## Tools

| Tool | Purpose |
|------|---------|
| Azure Cost Management | Azure spend analysis and budgets |
| AWS Cost Explorer | AWS spend analysis and recommendations |
| Kubecost | Kubernetes cost allocation per namespace/team |
| Infracost | Terraform plan cost estimation in CI |
| CloudHealth / Apptio Cloudability | Multi-cloud FinOps platform |

## FinOps Practices

- Tag all resources with cost allocation tags (project, team, environment)
- Generate monthly cost reports per team/project
- Set budget alerts at 80% and 100% thresholds
- Review Reserved Instance / Savings Plan coverage quarterly

## Azure FinOps — IaC Checklist

### Budget Alerts

Always declare budget resources — no alert means cost spikes go unnoticed until the quarterly review.

```hcl
resource "azurerm_consumption_budget_resource_group" "main" {
  name              = "budget-${var.project}-${var.environment}"
  resource_group_id = azurerm_resource_group.main.id
  amount            = var.monthly_budget_usd
  time_grain        = "Monthly"

  time_period {
    start_date = formatdate("YYYY-MM-01'T'00:00:00'Z'", timestamp())
  }

  notification {
    enabled        = true
    threshold      = 80
    operator       = "GreaterThan"
    threshold_type = "Actual"
    contact_emails = var.budget_alert_emails
  }

  notification {
    enabled        = true
    threshold      = 100
    operator       = "GreaterThan"
    threshold_type = "Actual"
    contact_emails = var.budget_alert_emails
  }
}
```

### Tagging Enforcement via Azure Policy

Deny resource creation when required cost-allocation tags are missing.

```hcl
resource "azurerm_policy_definition" "require_tags" {
  name         = "require-cost-tags"
  policy_type  = "Custom"
  mode         = "Indexed"
  display_name = "Require cost allocation tags"

  policy_rule = jsonencode({
    if = {
      anyOf = [
        { field = "tags['project']", exists = "false" },
        { field = "tags['team']", exists = "false" },
        { field = "tags['environment']", exists = "false" }
      ]
    }
    then = { effect = "Deny" }
  })
}

resource "azurerm_resource_group_policy_assignment" "require_tags" {
  name                 = "require-cost-tags"
  resource_group_id    = azurerm_resource_group.main.id
  policy_definition_id = azurerm_policy_definition.require_tags.id
}
```

### VM Auto-Shutdown (dev/test)

Prevents 24/7 VM costs in non-production environments.

```hcl
resource "azurerm_dev_test_global_vm_shutdown_schedule" "main" {
  virtual_machine_id = azurerm_linux_virtual_machine.main.id
  location           = azurerm_resource_group.main.location
  enabled            = true

  daily_recurrence_time = "1900"
  timezone              = "UTC"

  notification_settings {
    enabled = false
  }
}
```

### Application Insights Sampling

Missing sampling config is a common source of unexpectedly large bills.

```hcl
resource "azurerm_application_insights" "main" {
  name                = "appi-${var.project}-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  application_type    = "web"
  workspace_id        = azurerm_log_analytics_workspace.main.id

  sampling_percentage = 10  # reduce for high-traffic services; never leave at 100 in prod
}
```

### Log Analytics — Workspace Separation and Basic Logs

Put Sentinel on a dedicated workspace — mixing it with operational logs inflates costs significantly. Use Basic Logs tier for high-volume, rarely queried tables (e.g. raw diagnostic logs).

```hcl
# Operational workspace — AKS, App Service diagnostics, etc.
resource "azurerm_log_analytics_workspace" "ops" {
  name                = "log-${var.project}-${var.environment}-ops"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# Sentinel on its own workspace to avoid cost bleed
resource "azurerm_log_analytics_workspace" "sentinel" {
  name                = "log-${var.project}-${var.environment}-sentinel"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "PerGB2018"
  retention_in_days   = 90
}

# Downgrade high-volume diagnostic tables to Basic Logs (significantly cheaper)
resource "azurerm_log_analytics_workspace_table" "aks_diagnostics" {
  workspace_id  = azurerm_log_analytics_workspace.ops.id
  name          = "ContainerLogV2"
  plan          = "Basic"  # ~8x cheaper than Analytics tier; supports only basic KQL
  retention_in_days = 8
}
```

### Diagnostic Settings

Not enabled by default — must be declared explicitly per resource. Omitting them means no logs flow to Log Analytics or Datadog.

```hcl
resource "azurerm_monitor_diagnostic_setting" "aks" {
  name                       = "diag-${azurerm_kubernetes_cluster.main.name}"
  target_resource_id         = azurerm_kubernetes_cluster.main.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.ops.id

  enabled_log {
    category = "kube-apiserver"
  }

  enabled_log {
    category = "kube-audit-admin"
  }

  metric {
    category = "AllMetrics"
    enabled  = false  # metrics via Azure Monitor are cheaper; enable only if needed here
  }
}
```
