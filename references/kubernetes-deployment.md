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
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 5
  # delays liveness checks until app is ready; prevents restart loops on slow starts

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

## Manifest Best Practices

### Workload controllers
Never run naked Pods (Pods not owned by a controller). If a node dies, naked Pods are not rescheduled. Use a Deployment for every long-running workload:

```yaml
# BAD — naked Pod, not rescheduled on node failure
apiVersion: v1
kind: Pod
metadata:
  name: my-app

# GOOD — Deployment manages a ReplicaSet
apiVersion: apps/v1
kind: Deployment
```

Use a `Job` for one-off tasks (database migrations, batch processing) — it retries on failure and reports completion:

```yaml
apiVersion: batch/v1
kind: Job
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: my-app:1.2.3
          command: ["./migrate"]
```

### Service ordering
Create `Service` objects **before** the workloads that depend on them. Kubernetes injects service coordinates as environment variables at Pod start time:

```
FOO_SERVICE_HOST=10.0.0.1
FOO_SERVICE_PORT=8080
```

If the Service is created after the Pod starts, those variables will be missing. Prefer DNS (`foo.default.svc.cluster.local`) over env vars for resilience.

### Group related objects
Put the Deployment, Service, and ConfigMap for one application in a single manifest file. Apply an entire directory at once:

```bash
kubectl apply -f configs/my-app/
```

### Avoid hostPort and hostNetwork
`hostPort` and `hostNetwork: true` bind Pods to a specific node, breaking scheduling and scaling. Use them only when there is no alternative (e.g., CNI plugins, node-level daemons).

### Annotate resources
Use `kubernetes.io/description` to document why a resource exists:

```yaml
metadata:
  annotations:
    kubernetes.io/description: "Serves the public API; scaled by HPA on CPU and RPS."
```

### YAML boolean caution
Kubernetes YAML parses `yes`, `no`, `on`, `off` as booleans in some contexts. Quote them explicitly to avoid surprises:

```yaml
# BAD
value: yes

# GOOD
value: "yes"
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
- Missing startup probe on slow-starting containers (causes liveness restart loops)
- Single replica for production workloads
- Using `latest` image tag
- Storing secrets in ConfigMaps
- Naked Pods (not owned by a Deployment or StatefulSet)
- `hostNetwork: true` or `hostPort` without a strong reason

## Dev, Debug & Chaos Tools

| Tool | Category | Purpose |
|------|----------|---------|
| [K9s](https://k9scli.io/) | Terminal UI | Keyboard-driven cluster navigation, log tailing, exec |
| [stern](https://github.com/stern/stern) | Logging | Tail multiple pods/containers simultaneously with regex filtering |
| [kubectx + kubens](https://github.com/ahmetb/kubectx) | Context | Fast context and namespace switching |
| [Telepresence](https://www.telepresence.io/) | Local dev | Connect a local process into the cluster network for fast iteration |
| [Tilt](https://tilt.dev/) | Local dev | Hot-reload dev loop: rebuild and re-deploy on file save |
| [Chaos Mesh](https://chaos-mesh.org/) | Chaos | Cloud-native chaos engineering (pod kill, network delay, disk fault) |
| [Litmus](https://litmuschaos.io/) | Chaos | SRE-focused chaos orchestration with pre-built experiments |
| [kube-hunter](https://github.com/aquasecurity/kube-hunter) | Security | Active security scanning for cluster weaknesses |
| [Testkube](https://testkube.io/) | Testing | Kubernetes-native test execution framework |
| [Popeye](https://github.com/derailed/popeye) | Hygiene | Scans cluster for misconfigured resources and potential issues |

## Further Reading

- [Kubernetes The Hard Way (v1.33)](https://github.com/kaluzaaa/hard-way-1-33) — step-by-step cluster bootstrap from scratch; teaches every component
- [awesome-k8s-resources](https://github.com/tomhuang12/awesome-k8s-resources) — curated list of tools across all Kubernetes domains
