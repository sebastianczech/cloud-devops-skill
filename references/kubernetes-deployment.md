# Kubernetes Deployment Patterns

## Workload Best Practices

### Resource Requests and Limits
Always set both requests and limits to enable the scheduler to place pods correctly and prevent noisy-neighbor issues.

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

### Health Probes
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

## Deployment Strategies

### Rolling Update (default)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

### Blue/Green
- Deploy new version as separate Deployment
- Switch Service selector label to new version
- Keep old version running until validation passes

### Canary (with ingress-nginx or Flagger)
- Route a percentage of traffic to new version
- Gradually increase traffic weight after metrics validation

## Namespace Conventions

| Namespace | Purpose |
|-----------|---------|
| `default` | Avoid — use named namespaces |
| `kube-system` | Kubernetes system components |
| `monitoring` | Prometheus, Grafana, alerting |
| `ingress-nginx` | Ingress controller |
| `<app>-dev` / `<app>-prod` | Per-environment app namespaces |

## Security Hardening

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

## Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## GitOps with Flux/ArgoCD

- All manifests stored in Git — cluster state reconciled automatically
- Use Kustomize overlays or Helm values for per-environment customization
- Image update automation: detect new image tags, open PRs, auto-merge on passing checks

## Common Anti-Patterns

- Containers running as root
- Missing resource requests/limits
- Single replica for production workloads
- Using `latest` image tag
- Storing secrets in ConfigMaps
