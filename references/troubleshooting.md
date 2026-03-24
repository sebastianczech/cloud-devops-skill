# Troubleshooting Guide

## Terraform

### State Lock
```bash
# If apply/plan is stuck due to stale lock
terraform force-unlock <lock-id>
```

### State Drift
```bash
# Check for drift without applying
terraform plan -refresh-only

# Import existing resource into state
terraform import <resource.name> <resource-id>
```

### Provider Authentication Errors
- Azure: run `az login` or check `ARM_*` environment variables
- AWS: run `aws configure` or check `AWS_*` environment variables
- Ensure service principal / IAM role has required permissions

## Kubernetes

### Pod Not Starting
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
```

Common causes:
- `ImagePullBackOff` → registry credentials missing or image tag wrong
- `CrashLoopBackOff` → application error, check logs
- `Pending` → insufficient resources, check node capacity and requests

### Service Not Reachable
```bash
# Check endpoints are populated
kubectl get endpoints <service-name> -n <namespace>

# Check network policy is not blocking
kubectl get networkpolicy -n <namespace>
```

### Node Issues
```bash
kubectl describe node <node-name>
kubectl top nodes
kubectl top pods -A --sort-by=memory
```

## CI/CD

### Pipeline Fails on Docker Push
- Check registry authentication (ACR/ECR credentials or OIDC setup)
- Verify image name and tag format
- Check that the pipeline has network access to the registry

### Terraform Plan/Apply in Pipeline Fails
- Verify OIDC or service principal permissions
- Check backend storage access (Blob / S3)
- Ensure `terraform init` is run before `plan`/`apply`

## Azure

### AKS Nodes Not Joining
- Check node pool subnet has available IP addresses
- Verify NSG rules allow required traffic
- Check managed identity has `Network Contributor` on subnet

### Key Vault Access Denied
- Verify managed identity / service principal is in Key Vault access policy or has RBAC role
- Check if Key Vault firewall is blocking the caller IP/VNet

## AWS

### EKS Nodes Not Joining
- Check node IAM role has `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, `AmazonEC2ContainerRegistryReadOnly`
- Verify node group security group allows API server communication

### ECR Pull Errors from EKS
- Check node IAM role has `AmazonEC2ContainerRegistryReadOnly`
- For cross-account: verify ECR repository policy allows the node role

## General Debugging Checklist

- [ ] Check recent changes (git log, deployment history)
- [ ] Check cloud provider service health dashboard
- [ ] Check resource quotas and limits
- [ ] Verify network connectivity and firewall rules
- [ ] Review logs in centralized logging (Log Analytics / CloudWatch)
- [ ] Check for certificate expiry
