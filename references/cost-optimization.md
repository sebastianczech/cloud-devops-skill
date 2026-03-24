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
