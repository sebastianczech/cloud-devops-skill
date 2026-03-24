# Azure Cloud Patterns

## Core Services for DevOps

| Service | Purpose |
|---------|---------|
| AKS | Managed Kubernetes |
| ACR | Container Registry |
| Azure DevOps / GitHub Actions | CI/CD pipelines |
| Key Vault | Secrets, certificates, keys |
| Azure Monitor / Log Analytics | Observability |
| Entra ID (AAD) | Identity and RBAC |
| Azure Policy | Compliance guardrails |

## AKS Best Practices

- Enable managed identity (system-assigned or user-assigned) — no service principal secrets
- Use Azure CNI for advanced networking (required for network policies)
- Enable workload identity for pod-level Azure RBAC
- Use node pools: system pool (reserved for system pods) + user pool(s) for workloads
- Enable cluster auto-scaler per node pool

### AKS Workload Identity
```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  workload_identity_enabled = true
  oidc_issuer_enabled       = true
}
```

## Networking

```
Hub VNet
├── Azure Firewall / NVA
├── VPN / ExpressRoute Gateway
└── Spoke VNets (peered)
    ├── AKS VNet
    ├── App VNet
    └── Data VNet
```

- Use Private Endpoints for PaaS services (Storage, Key Vault, ACR)
- Restrict AKS API server with authorized IP ranges or private cluster

## Resource Naming Convention

Pattern: `<abbreviation>-<project>-<environment>-<region>-<suffix>`

```
rg-myapp-prod-eastus-001
aks-myapp-prod-eastus-001
cr-myapp-prod-001          (no hyphens — ACR doesn't allow them; use 'cr' prefix)
kv-myapp-prod-eastus-001
```

### Official Azure Resource Abbreviations

Source (authoritative and complete list): [Azure CAF Resource Abbreviations](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations)

The full set of recommended abbreviations is maintained by Microsoft in the link above.
The table below is a **small, non-exhaustive sample** included only to illustrate how
the naming pattern is applied; refer to the official documentation for all services.

| Resource (example)              | Abbreviation (example) |
|---------------------------------|-------------------------|
| Resource group                  | `rg`                    |
| Virtual network                 | `vnet`                  |
| Network security group          | `nsg`                   |
| Azure Kubernetes Service        | `aks`                   |
| Azure Container Registry        | `cr`                    |
| Key Vault                       | `kv`                    |
| Custom vision (training) | `cstvt` |
| Document intelligence | `di` |
| Face API | `face` |
| Health Insights | `hi` |
| Immersive reader | `ir` |
| Language service | `lang` |
| Speech service | `spch` |
| Translator | `trsl` |

#### Analytics and IoT

| Resource | Abbreviation |
|----------|-------------|
| Azure Analysis Services server | `as` |
| Azure Databricks Access Connector | `dbac` |
| Azure Databricks workspace | `dbw` |
| Azure Data Explorer cluster | `dec` |
| Azure Data Explorer cluster database | `dedb` |
| Azure Data Factory | `adf` |
| Azure Digital Twin instance | `dt` |
| Azure Stream Analytics | `asa` |
| Azure Synapse Analytics private link hub | `synplh` |
| Azure Synapse Analytics SQL Dedicated Pool | `syndp` |
| Azure Synapse Analytics Spark Pool | `synsp` |
| Azure Synapse Analytics workspaces | `synw` |
| Data Lake Store account | `dls` |
| Event Hubs namespace | `evhns` |
| Event hub | `evh` |
| Event Grid domain | `evgd` |
| Event Grid namespace | `evgns` |
| Event Grid subscriptions | `evgs` |
| Event Grid topic | `evgt` |
| Event Grid system topic | `egst` |
| Fabric Capacity | `fc` |
| HDInsight - Hadoop cluster | `hadoop` |
| HDInsight - HBase cluster | `hbase` |
| HDInsight - Kafka cluster | `kafka` |
| HDInsight - Spark cluster | `spark` |
| HDInsight - Storm cluster | `storm` |
| HDInsight - ML Services cluster | `mls` |
| IoT hub | `iot` |
| Provisioning services | `provs` |
| Provisioning services certificate | `pcert` |
| Power BI Embedded | `pbi` |
| Time Series Insights environment | `tsi` |

#### Compute and Web

| Resource | Abbreviation |
|----------|-------------|
| App Service environment | `ase` |
| App Service plan | `asp` |
| Azure Load Testing instance | `lt` |
| Availability set | `avail` |
| Azure Arc enabled server | `arcs` |
| Azure Arc enabled Kubernetes cluster | `arck` |
| Azure Arc private link scope | `pls` |
| Azure Arc gateway | `arcgw` |
| Batch accounts | `ba` |
| Cloud service | `cld` |
| Communication Services | `acs` |
| Disk encryption set | `des` |
| Function app | `func` |
| Gallery | `gal` |
| Hosting environment | `host` |
| Image template | `it` |
| Managed disk (OS) | `osdisk` |
| Managed disk (data) | `disk` |
| Notification Hubs | `ntf` |
| Notification Hubs namespace | `ntfns` |
| Proximity placement group | `ppg` |
| Restore point collection | `rpc` |
| Snapshot | `snap` |
| Static web app | `stapp` |
| Virtual machine | `vm` |
| Virtual machine scale set | `vmss` |
| VM storage account | `stvm` |
| Web app | `app` |

#### Containers

| Resource | Abbreviation |
|----------|-------------|
| AKS cluster | `aks` |
| AKS system node pool | `npsystem` |
| AKS user node pool | `np` |
| Container apps | `ca` |
| Container apps environment | `cae` |
| Container apps job | `caj` |
| Container registry | `cr` |
| Container instance | `ci` |
| Service Fabric cluster | `sf` |
| Service Fabric managed cluster | `sfmc` |

#### Databases

| Resource | Abbreviation |
|----------|-------------|
| Azure Cosmos DB database | `cosmos` |
| Azure Cosmos DB for Apache Cassandra account | `coscas` |
| Azure Cosmos DB for MongoDB account | `cosmon` |
| Azure Cosmos DB for NoSQL account | `cosno` |
| Azure Cosmos DB for Table account | `costab` |
| Azure Cosmos DB for Apache Gremlin account | `cosgrm` |
| Azure Cosmos DB PostgreSQL cluster | `cospos` |
| Azure Managed Redis | `amr` |
| Azure SQL Database server | `sql` |
| Azure SQL database | `sqldb` |
| Azure SQL Elastic Job agent | `sqlja` |
| Azure SQL Elastic Pool | `sqlep` |
| MySQL database | `mysql` |
| PostgreSQL database | `psql` |
| SQL Managed Instance | `sqlmi` |

#### Developer Tools

| Resource | Abbreviation |
|----------|-------------|
| App Configuration store | `appcs` |
| Maps account | `map` |
| SignalR | `sigr` |
| WebPubSub | `wps` |

#### DevOps

| Resource | Abbreviation |
|----------|-------------|
| Azure Managed Grafana | `amg` |
| Managed DevOps Pools | `mdp` |

#### Integration

| Resource | Abbreviation |
|----------|-------------|
| API Management service instance | `apim` |
| Integration account | `ia` |
| Logic app | `logic` |
| Service Bus namespace | `sbns` |
| Service Bus queue | `sbq` |
| Service Bus topic | `sbt` |
| Service Bus topic subscription | `sbts` |

#### Management and Governance

| Resource | Abbreviation |
|----------|-------------|
| Automation account | `aa` |
| Application Insights | `appi` |
| Azure Monitor action group | `ag` |
| Azure Monitor data collection rule | `dcr` |
| Azure Monitor alert processing rule | `apr` |
| Data collection endpoint | `dce` |
| Deployment scripts | `script` |
| Log Analytics workspace | `log` |
| Log Analytics query packs | `pack` |
| Management group | `mg` |
| Microsoft Purview instance | `pview` |
| Resource group | `rg` |
| Template specs | `ts` |

#### Migration

| Resource | Abbreviation |
|----------|-------------|
| Azure Migrate project | `migr` |
| Database Migration Service instance | `dms` |
| Recovery Services vault | `rsv` |

#### Networking

| Resource | Abbreviation |
|----------|-------------|
| Application gateway | `agw` |
| Application security group (ASG) | `asg` |
| CDN profile | `cdnp` |
| CDN endpoint | `cdne` |
| Connections | `con` |
| DNS forwarding ruleset | `dnsfrs` |
| DNS private resolver | `dnspr` |
| DNS private resolver inbound endpoint | `in` |
| DNS private resolver outbound endpoint | `out` |
| Firewall | `afw` |
| Firewall policy | `afwp` |
| ExpressRoute circuit | `erc` |
| ExpressRoute direct | `erd` |
| ExpressRoute gateway | `ergw` |
| Front Door (Standard/Premium) profile | `afd` |
| Front Door (Standard/Premium) endpoint | `fde` |
| Front Door firewall policy | `fdfp` |
| IP group | `ipg` |
| Load balancer (internal) | `lbi` |
| Load balancer (external) | `lbe` |
| Local network gateway | `lgw` |
| NAT gateway | `ng` |
| Network interface (NIC) | `nic` |
| Network security perimeter | `nsp` |
| Network security group (NSG) | `nsg` |
| Network Watcher | `nw` |
| Private Link | `pl` |
| Private endpoint | `pep` |
| Public IP address | `pip` |
| Public IP address prefix | `ippre` |
| Route filter | `rf` |
| Route server | `rtserv` |
| Route table | `rt` |
| Traffic Manager profile | `traf` |
| User defined route (UDR) | `udr` |
| Virtual network | `vnet` |
| Virtual network gateway | `vgw` |
| Virtual network manager | `vnm` |
| Virtual network peering | `peer` |
| Virtual network subnet | `snet` |
| Virtual WAN | `vwan` |
| Virtual WAN Hub | `vhub` |

#### Security

| Resource | Abbreviation |
|----------|-------------|
| Azure Bastion | `bas` |
| Key vault | `kv` |
| Key Vault Managed HSM | `kvmhsm` |
| Managed identity | `id` |
| SSH key | `sshkey` |
| VPN Gateway | `vpng` |
| VPN connection | `vcn` |
| VPN site | `vst` |
| Web Application Firewall (WAF) policy | `waf` |
| WAF policy rule group | `wafrg` |

#### Storage

| Resource | Abbreviation |
|----------|-------------|
| Backup Vault | `bvault` |
| Backup Vault policy | `bkpol` |
| File share | `share` |
| Storage account | `st` |
| Storage Sync Service | `sss` |

#### Virtual Desktop Infrastructure

| Resource | Abbreviation |
|----------|-------------|
| Virtual desktop host pool | `vdpool` |
| Virtual desktop application group | `vdag` |
| Virtual desktop workspace | `vdws` |
| Virtual desktop scaling plan | `vdscaling` |

## IaC for Azure

- Prefer **Terraform** for multi-cloud or team familiarity
- Use **Bicep** for Azure-native, simpler state management
- Use **Azure Verified Modules** (AVM) — Microsoft-maintained Terraform/Bicep modules

## Cost Management

- Tag all resources with `Environment`, `Project`, `Owner`
- Use Azure Cost Management budgets and alerts
- Enable auto-shutdown for dev/test VMs
- Use spot/preemptible node pools for batch workloads

## Security Checklist

- [ ] Enable Microsoft Defender for Cloud
- [ ] Enable Azure Policy initiatives (CIS, NIST)
- [ ] Enforce private endpoints for all PaaS
- [ ] Enable diagnostic settings → Log Analytics for all resources
- [ ] Use Privileged Identity Management (PIM) for elevated access

## AKS Checklist

Source: [the-aks-checklist.com](https://www.the-aks-checklist.com/) — last updated 2026-02-15

### Identity & Authorization

- [ ] Integrate authentication with AAD (managed integration) — use Azure AD tokens, not local accounts
- [ ] Integrate authorization with AAD RBAC — combine Kubernetes RBAC with Azure AD identities
- [ ] Use AKS + ACR integration without passwords — use Managed Identity instead of stored credentials
- [ ] Use managed identities instead of Service Principals
- [ ] Limit access to admin kubeconfig via Azure RBAC
- [ ] Use `kubelogin` for non-interactive logins (service principal sign-in)
- [ ] Disable AKS local accounts (`--disable-local-accounts`)
- [ ] Configure Just-in-Time (JIT) cluster access if required
- [ ] Use managed Kubelet Identity for finer control
- [ ] Configure AAD Conditional Access for AKS if required

### Cluster Security

- [ ] Use Azure Linux as host OS
- [ ] Configure cluster for regulated industries (FIPS-enabled nodes, CIS benchmarks)
- [ ] Ensure Kubernetes dashboard is not installed in production (disabled by default since v1.19)
- [ ] Encrypt ETCD at rest with your own key (KMS plugin + Azure Key Vault)
- [ ] Keep Kubernetes version up to date — support covers N-2 versions only
- [ ] Block deployment of vulnerable images via policy
- [ ] Use Azure Key Vault with Secrets Store CSI driver
- [ ] Enable Microsoft Defender for Containers
- [ ] Use ImageCleaner to remove stale/vulnerable images from nodes
- [ ] Use Azure Policy for Kubernetes (Gatekeeper) for compliance enforcement
- [ ] Separate applications from control plane — taint system node pool
- [ ] Refresh Service Principal credentials periodically (if not using managed identity)
- [ ] Use private registry (ACR) for all images
- [ ] Consider Confidential Compute for hardware-level isolation if required
- [ ] Define app separation strategy (namespace / node pool / cluster isolation)
- [ ] Consider Azure Dedicated Hosts for physical isolation if required

### Multi-Tenancy & Isolation

- [ ] Use logical isolation (namespaces, RBAC) before resorting to physical cluster separation
- [ ] Apply Azure tags on clusters and related resources (Environment, Project, Owner)

### Storage

- [ ] Choose the right storage type for the workload (SSD for performance, network-based for concurrent access)
- [ ] Size nodes for storage needs (disk count and bandwidth per SKU)
- [ ] Use dynamic volume provisioning with appropriate reclaim policies
- [ ] Secure and back up persistent data (Velero, Azure Backup for AKS)
- [ ] Keep application state outside the cluster — use Azure PaaS (Storage, SQL, Cosmos DB)
- [ ] Use ephemeral OS disks for lower latency and faster upgrades/scale
- [ ] Use Ultra Disks for high-IOPS stateful workloads
- [ ] Use ZRS disks for multi-zone node pools; LRS for single-zone
- [ ] Use high-IOPS OS disks for high-pod-density nodes

### Networking

- [ ] Choose the right CNI: Azure CNI (full VNet integration) or Azure CNI Overlay (recommended default)
- [ ] Size subnets appropriately for Azure CNI — account for pods, nodes, and upgrade surge IPs
- [ ] Use ingress controller (Application Routing add-on or ingress-nginx) instead of per-service LoadBalancers
- [ ] Protect exposed apps with WAF (Azure Application Gateway or Front Door)
- [ ] Restrict allowed ingress hostnames
- [ ] Default to internal ingress — do not expose load balancers publicly unless required
- [ ] Use network policies (Calico or Cilium) to control pod-to-pod traffic
- [ ] Configure deny-all default network policy per namespace, then add specific allows
- [ ] Filter egress with Azure Firewall or NVA if required (egress lockdown)
- [ ] Use Private Link for ACR — do not expose registry to the internet
- [ ] Block pod access to VMSS IMDS endpoint via Network Policy
- [ ] Use private cluster (private API server) if required; access via Bastion or jumpbox
- [ ] Restrict API server to authorized IP ranges if using public endpoint
- [ ] Use Azure NAT Gateway as outbound type for scalable egress
- [ ] Use dynamic IP allocation to avoid Azure CNI IP exhaustion
- [ ] Use Private Endpoints or VNet Service Endpoints for PaaS services
- [ ] Verify no CIDR overlaps: Service-CIDR, Pod-CIDR, Subnet-CIDR
- [ ] Consider a service mesh (Istio, Linkerd) for advanced traffic management and observability

### Resource Management

- [ ] Right-size worker nodes — balance blast radius vs management overhead
- [ ] Use ARM64 node pools where possible — better price/performance
- [ ] Enforce resource quotas per namespace
- [ ] Set LimitRange defaults per namespace
- [ ] Set CPU and memory requests and limits on all containers
- [ ] Configure Pod Disruption Budgets (PDBs) for all production workloads
- [ ] Use Kubecost (or similar) for namespace/team cost allocation
- [ ] Use NodePool Start/Stop for dev/test environments to reduce cost
- [ ] Use spot node pools for non-critical / fault-tolerant workloads
- [ ] Use scale-down mode (delete vs deallocate) intentionally
- [ ] Ensure sufficient subscription quota before scaling

### Cluster Operations

- [ ] Enable node image auto-upgrade (or upgrade weekly) — new image released weekly
- [ ] Set cluster auto-upgrade channel
- [ ] Enable cluster autoscaler (or Node Auto-Provisioning / Karpenter)
- [ ] Enable AKS auto-certificate rotation (e.g., every 90 days)
- [ ] Use GitOps (Flux v2 add-on or ArgoCD) to deploy workloads
- [ ] Implement CI/CD pipelines alongside GitOps
- [ ] Do not use the `default` namespace for applications
- [ ] Apply standard labels (technical, business, security) to all resources
- [ ] Send API/master logs to Azure Monitor (Log Analytics)
- [ ] Enable Container Insights (or Prometheus + Grafana) for metrics
- [ ] Enable ContainerLogV2 schema for improved log querying
- [ ] Configure distributed tracing (App Insights Zero Instrumentation or Codeless Attach)
- [ ] Configure alerts on critical metrics (Container Insights recommendations)
- [ ] Subscribe to resource health notifications for AKS
- [ ] Subscribe to EventGrid events for AKS automation
- [ ] Monitor SNAT port consumption if not using egress filtering
- [ ] Regularly check Azure Advisor recommendations
- [ ] Regularly scan cluster for issues (AKS Periscope, kube-bench)
- [ ] Consider Long-Term Support (LTS) Kubernetes versions for 24-month support lifecycle

### Business Continuity & Disaster Recovery

- [ ] Define SLAs, RTO, and RPO before going to production
- [ ] Use Availability Zones for all production clusters
- [ ] Plan for multi-region deployment — use Azure paired regions
- [ ] Use Azure Traffic Manager or Front Door for cross-region routing
- [ ] Create a storage migration / sync plan for multi-region clusters
- [ ] Use standard-tier AKS (Uptime SLA — 99.95% SLA)
- [ ] Use pod anti-affinity rules to spread replicas across nodes
- [ ] Configure ACR geo-replication across regions
- [ ] Enable ACR zone redundancy
- [ ] Move ACR to a dedicated resource group (prevents accidental deletion)
- [ ] Enable ACR soft-delete policy
- [ ] Use Azure Backup for AKS (cluster config + persistent volumes)
- [ ] Schedule and perform DR tests regularly (full automated redeployment test)

### Windows Node Pools (if applicable)

- [ ] Match container image OS version to host node OS version
- [ ] Handle abrupt container termination — `TerminationGracePeriod` not implemented on Windows
- [ ] Do not use privileged containers — not supported on Windows
- [ ] Monitor memory usage — no OOM eviction on Windows
- [ ] Use Azure CNI — Kubenet not supported for Windows node pools
- [ ] Manually upgrade Windows node pools for security patches
- [ ] Secure container traffic with an authentication layer (network policies unsupported on Windows)
- [ ] Enable Group Managed Service Accounts (GMSA)
- [ ] Taint Windows nodes and use node selectors to prevent cross-OS scheduling
- [ ] Use Calico Network Policies for Windows Server 2019/2022
- [ ] Enable Accelerated Networking for Windows workloads

### Application Deployment

- [ ] Implement liveness probes — detect broken pods and trigger restart
- [ ] Implement startup probes — delay liveness checks for slow-starting containers
- [ ] Implement readiness probes — signal when the app is ready to receive traffic
