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
| Kubernetes | Kube-bench, Kubesec, Polaris |
| Runtime | Falco, Microsoft Defender for Containers |
