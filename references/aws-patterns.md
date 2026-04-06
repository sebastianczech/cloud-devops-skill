# AWS Cloud Patterns

## Core Services for DevOps

| Service | Purpose |
|---------|---------|
| EKS | Managed Kubernetes |
| ECR | Container Registry |
| CodePipeline + CodeBuild | CI/CD pipelines |
| Secrets Manager / Parameter Store | Secrets management |
| CloudWatch | Observability |
| IAM | Identity and access management |
| AWS Config | Compliance and drift detection |

## EKS Best Practices

- Use managed node groups or Fargate for reduced operational overhead
- Enable IRSA (IAM Roles for Service Accounts) for pod-level AWS permissions
- Use EKS-optimized AMIs with automatic updates via managed node groups
- Enable control plane logging (api, audit, authenticator, controllerManager, scheduler)
- Use AWS Load Balancer Controller instead of in-tree cloud provider

### IRSA — How It Works

IRSA lets pods access AWS services using IAM permissions without storing credentials in the container. It works at the pod level, equivalent to EC2 instance profiles but scoped to a Kubernetes service account.

**Token flow:**
1. Kubernetes issues a `ProjectedServiceAccountToken` (OIDC JWT) to the pod
2. EKS hosts a public OIDC discovery endpoint with signing keys (keys rotate every 7 days)
3. The AWS SDK exchanges the token with STS via `AssumeRoleWithWebIdentity`
4. STS returns short-lived credentials scoped to the IAM role

**Setup — three steps:**

**1. Create the IAM OIDC provider** (once per cluster):
```bash
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve
```
> **DNS gotcha**: If the EKS VPC endpoint is enabled, run this from outside the VPC (e.g., AWS CloudShell) or add a Route 53 Resolver conditional forwarder for the OIDC issuer URL. Otherwise DNS resolves `oidc.eks.<region>.amazonaws.com` to NXDOMAIN.

**2. Create an IAM role with a trust policy** referencing the service account:
```hcl
module "irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name             = "my-app-role"
  attach_s3_read_policy = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["my-namespace:my-app-sa"]
    }
  }
}
```

The trust policy created by this module looks like:
```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/oidc.eks.REGION.amazonaws.com/id/CLUSTER_ID"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "oidc.eks.REGION.amazonaws.com/id/CLUSTER_ID:sub": "system:serviceaccount:my-namespace:my-app-sa"
    }
  }
}
```

**3. Annotate the Kubernetes service account:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-namespace
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/my-app-role
```

**Caveats:**
- Pods with `hostNetwork: true` always have IMDS access, but IRSA credentials take priority when the annotation is set
- If IMDS is unrestricted (hop limit > 1), containers can also access the node IAM role — restrict it with `--metadata-options` or a hop limit of 1
- External OIDC clients (e.g., third-party IdPs) must refresh signing keys before the 7-day rotation window expires

## Networking

```
VPC
├── Public Subnets (one per AZ) — Load Balancers, NAT Gateways
├── Private Subnets (one per AZ) — EKS nodes, Lambda
└── Isolated Subnets (one per AZ) — RDS, ElastiCache
```

- Enable VPC Flow Logs
- Use VPC Endpoints for S3, ECR, Secrets Manager to avoid NAT gateway costs
- Use AWS PrivateLink for third-party service access

## Resource Naming Convention

```
<project>-<environment>-<type>-<region>-<suffix>

Examples:
  myapp-prod-eks-us-east-1
  myapp-prod-ecr
  myapp-prod-secrets-db-password
```

## IaC for AWS

- **Terraform** + `terraform-aws-modules` — community-maintained, battle-tested
- **CDK** — TypeScript/Python, generates CloudFormation, good for developers
- **CloudFormation** — native, no state file, AWS-native integrations

## Cost Management

- Use Cost Allocation Tags for all resources
- Enable AWS Budgets with alerts at 80% and 100% of budget
- Use Spot Instances for fault-tolerant and batch workloads
- Enable Compute Savings Plans or Reserved Instances for predictable workloads
- Use S3 Intelligent-Tiering for variable access patterns

## Security Checklist

- [ ] Enable AWS Security Hub with CIS AWS Foundations benchmark
- [ ] Enable GuardDuty for threat detection
- [ ] Enable AWS Config with managed rules
- [ ] Enable CloudTrail in all regions
- [ ] Use SCPs (Service Control Policies) in AWS Organizations
- [ ] Enforce MFA for human IAM users
- [ ] Rotate access keys regularly or eliminate them with IRSA/OIDC

## EKS Best Practices Checklist

Source: [aws-samples/hardeneks](https://github.com/aws-samples/hardeneks) — rules map to the [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)

### IAM (Cluster-wide)

- [ ] Restrict wildcard verbs and resources in ClusterRoles
- [ ] Restrict ClusterRole bindings of anonymous or unauthenticated groups
- [ ] Restrict EKS cluster endpoint to private access (disable public endpoint)
- [ ] Use IRSA or EKS Pod Identity for the `aws-node` daemonset — do not use node instance profile
- [ ] Restrict pod access to the EC2 instance profile (block IMDS or require IMDSv2 hop limit of 1)

### IAM (Namespace-based)

- [ ] Restrict wildcard verbs and resources in Roles
- [ ] Restrict Role bindings of anonymous or unauthenticated groups
- [ ] Disable `automountServiceAccountToken` on the default service account
- [ ] Run pods as non-root user (`runAsNonRoot: true`)
- [ ] Use dedicated service accounts per Deployment, StatefulSet, and DaemonSet

### Multi-Tenancy

- [ ] Assign resource quotas to every namespace

### Detective Controls

- [ ] Enable EKS control plane logs (API, audit, authenticator, controller manager, scheduler)

### Network Security (Cluster-wide)

- [ ] Enable VPC Flow Logs
- [ ] Install AWS Private CA issuer for certificate management
- [ ] Configure default deny-all network policies for all namespaces

### Network Security (Namespace-based)

- [ ] Specify TLS/SSL certificate on all AWS Load Balancers

### Image Security

- [ ] Enable immutable image tags in ECR repositories

### Pod Security (Namespace-based)

- [ ] Disallow container socket mounts (Docker socket)
- [ ] Restrict `hostPath` volume mounts — if required, make them read-only
- [ ] Set CPU and memory requests and limits on all containers
- [ ] Set `allowPrivilegeEscalation: false` in pod spec
- [ ] Configure a read-only root filesystem (`readOnlyRootFilesystem: true`)

### Runtime Security

- [ ] Restrict Linux capabilities — drop `ALL`, add only what is explicitly required

### Reliability (Cluster-wide)

- [ ] Deploy Metrics Server (required for HPA)
- [ ] Deploy Vertical Pod Autoscaler (VPA) for right-sizing recommendations

### Reliability (Namespace-based)

- [ ] Run pods via Deployments — do not run singleton (bare) pods
- [ ] Run multiple replicas for each Deployment (minimum 2 for production)
- [ ] Spread replicas across AZs and nodes (topology spread constraints or pod anti-affinity)
- [ ] Configure Horizontal Pod Autoscaler (HPA) for all Deployments
- [ ] Define readiness probes for all pods
- [ ] Define liveness probes for all pods
