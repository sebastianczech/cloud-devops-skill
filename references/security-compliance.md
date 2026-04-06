# Security & Compliance

## Secrets Management

- Never hardcode secrets in code, IaC, or pipeline YAML
- Use cloud-native secret stores: Azure Key Vault, AWS Secrets Manager
- Inject secrets at runtime via CSI driver (Kubernetes) or environment variables from vault
- Rotate secrets regularly; automate rotation where possible

## Least Privilege IAM

- Start with zero permissions, add only what is needed
- Use managed identities / service accounts instead of long-lived keys
- Scope IAM roles to specific resources, not wildcards
- Review and audit permissions quarterly

## Container Security

- Scan images with Trivy or Snyk in CI pipeline
- Block images with critical CVEs from being deployed
- Sign images with cosign and enforce signature verification in cluster
- Use non-root users in Dockerfiles
- Minimize installed packages — use distroless or Alpine base images

## Kubernetes Security

- Enable Pod Security Standards (Baseline or Restricted) per namespace
- Use Network Policies to restrict pod-to-pod communication
- Enable audit logging on the API server
- Use OPA/Gatekeeper or Kyverno for policy enforcement
- Scan manifests with Checkov or Kubesec before deployment

## Infrastructure Security

- Enable encryption at rest for all storage (default on most managed services)
- Enforce TLS 1.2+ for all in-transit communication
- Use private endpoints / VPC endpoints for cloud service access
- Restrict management plane access to known IP ranges or VPN

## Compliance Frameworks

| Framework | Applicable To |
|-----------|--------------|
| CIS Benchmarks | Kubernetes, cloud provider baselines |
| SOC 2 | SaaS products, data handling |
| PCI-DSS | Payment card processing |
| HIPAA | Healthcare data (US) |
| ISO 27001 | General information security |

## Shift-Left Security Tools

| Stage | Tool |
|-------|------|
| Code | SonarQube, Semgrep |
| IaC | Checkov, tfsec, tflint |
| Container | Trivy, Snyk, Grype |
| Kubernetes | kube-bench, Kubesec, Polaris, kube-hunter |
| Runtime | Falco, Cilium Tetragon, Microsoft Defender for Containers |

- **[kube-hunter](https://github.com/aquasecurity/kube-hunter)** — actively probes the cluster for exploitable weaknesses (exposed APIs, privilege escalation paths); run in "remote" mode against a target cluster IP
- **[Cilium Tetragon](https://tetragon.io/)** — eBPF-based runtime enforcement; observes and blocks syscalls, file access, and network activity at the kernel level with near-zero overhead

## Security Training

**[Kubernetes Goat](https://madhuakula.com/kubernetes-goat)** — intentionally vulnerable cluster environment for hands-on security practice. Deploy in an isolated cluster only.

Key attack scenarios and the defensive control each one teaches:

| Scenario | Attack Vector | Defensive Control |
|----------|--------------|-------------------|
| Sensitive keys in codebases | Secrets committed to Git | Secret scanning, Sealed Secrets |
| Container escape to host | Privileged container, hostPath mount | `securityContext`, restricted PSS |
| SSRF in Kubernetes | Pod reaching cloud metadata endpoint | Network Policy, IMDS hop limit |
| RBAC misconfiguration | Over-permissive ClusterRole | Least-privilege roles, audit logs |
| Kubernetes namespace bypass | Missing network isolation | Default-deny NetworkPolicy |
| NodePort exposed services | Cluster internals reachable from internet | No NodePort in prod; use ingress |
| Crypto miner container | Compromised image | Image signing, Kyverno policy |
| Falco runtime detection | Unexpected process in container | Falco rules, alert on shell exec |
| Kyverno policy engine | Policy bypass | Admission webhook enforcement |
| Cilium Tetragon eBPF | Kernel-level evasion | eBPF enforcement on syscalls |
