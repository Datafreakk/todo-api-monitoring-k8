# Azure Cloud Interview Prep — Platform & DevOps Engineer

---

## TABLE OF CONTENTS

1. [Azure Core Concepts](#1-azure-core-concepts)
2. [Azure Networking](#2-azure-networking)
3. [Azure Identity & Security](#3-azure-identity--security)
4. [Azure Compute](#4-azure-compute)
5. [Azure Kubernetes Service (AKS)](#5-azure-kubernetes-service-aks)
6. [Azure Container Registry (ACR)](#6-azure-container-registry-acr)
7. [Azure Storage](#7-azure-storage)
8. [Azure Key Vault](#8-azure-key-vault)
9. [Azure Monitor & Observability](#9-azure-monitor--observability)
10. [Azure DevOps & CI/CD](#10-azure-devops--cicd)
11. [High Availability & Disaster Recovery](#11-high-availability--disaster-recovery)
12. [Azure Cost Management](#12-azure-cost-management)
13. [Common Interview Questions & Answers](#13-common-interview-questions--answers)

---

## 1. Azure Core Concepts

### Management Hierarchy
Azure has a 4-level hierarchy to organize resources and apply policies/access at scale:

```
Tenant (Azure AD / Entra ID)
  └── Management Groups        ← group subscriptions, apply policy at scale
        └── Subscriptions      ← billing boundary + resource limit boundary
              └── Resource Groups  ← logical container for related resources
                    └── Resources  ← actual VMs, VNets, DBs, etc.
```

- **Tenant** — It’s an identity boundary in Microsoft Azure
One tenant can host multiple organizations, apps, or environments (B2B, multi-tenant SaaS).
It governs identity, auth, RBAC principals, not billing or resources directly.
You can have multiple tenants per company (common in M&A or isolation scenarios).

- **Management Group** — t’s a hierarchical governance layer above subscriptions. used to apply RBAC and Azure Policy across multiple subscriptions (e.g., one policy for all prod subscriptions).
- **Subscription** — billing unit. You're charged per subscription. Also a hard limit boundary (e.g., max 980 VNets per subscription).
- **Resource Group** — logical grouping. Resources in the same RG share a lifecycle — you delete the RG and everything in it goes.

> **Interview tip:** Resource Groups are not a security boundary — they're a management boundary. Use RBAC + Policy for security.

### Azure Regions and Availability Zones
- A **Region** is a geographic location with one or more datacenters (e.g., East US, West Europe).
- An **Availability Zone (AZ)** is a physically separate datacenter within a region with independent power, cooling, and networking.
- AZs protect against **datacenter-level failures** — not region-level.
- For region-level protection, you need **paired regions** (e.g., East US ↔ West US).

```
Region: East US
  ├── Zone 1  (datacenter A)
  ├── Zone 2  (datacenter B)
  └── Zone 3  (datacenter C)
```

---

## 2. Azure Networking

### Virtual Network (VNet)
A VNet is your private network in Azure. Resources inside a VNet can communicate with each other by default. Traffic to the internet or other VNets must be explicitly configured.

A VNet is a private, isolated IP network in Azure where you control addressing, segmentation, and traffic flow between resources and external networks.

```
VNet: 10.0.0.0/16
  ├── Subnet: web-subnet     10.0.1.0/24   (VMs, AKS nodes)
  ├── Subnet: db-subnet      10.0.2.0/24   (databases)
  └── Subnet: mgmt-subnet    10.0.3.0/24   (bastion, admin tools)
```

- A subnet is a range within a VNet — used to segment traffic and apply different NSG rules.
- **Azure reserves 5 IPs per subnet** (first 4 + last 1).

### Network Security Groups (NSG)
An NSG is a firewall at the subnet or NIC level. It contains inbound and outbound rules based on priority.

```
NSG Rule:
  Priority: 100
  Source: Internet
  Destination: web-subnet
  Port: 443
  Action: Allow
```

- Lower priority number = evaluated first.
- Default rules block all inbound from internet, allow all outbound.
- NSG can be attached to a **subnet** (applies to all resources in it) or a **NIC** (applies to one VM).

### VNet Peering
VNet Peering connects two VNets so they can communicate privately without going through the internet or a gateway. Traffic stays on the Azure backbone.

- **Regional peering** — same region, very low latency.
- **Global peering** — across regions, slightly higher latency.
- **Not transitive** — if VNet A peers with B, and B peers with C, A cannot talk to C automatically. You need A-C peering or a hub-spoke topology with a gateway.

### Private Endpoints
A Private Endpoint gives an Azure PaaS service (like Azure SQL, Storage, Key Vault) a private IP inside your VNet. Traffic never leaves Azure's network.

```
Without Private Endpoint:
  VM → internet → storage.blob.core.windows.net (public IP)

With Private Endpoint:
  VM → private IP (10.0.1.5) → Azure Storage (same backbone, no internet)
```

Use Private Endpoints for any PaaS service handling sensitive data — databases, Key Vault, ACR, storage.

### Azure Firewall vs NSG

| | NSG | Azure Firewall |
|---|---|---|
| Layer | Layer 4 (TCP/UDP ports) | Layer 4 + Layer 7 (FQDN, URL filtering) |
| Scope | Subnet or NIC | Entire VNet / hub |
| Cost | Free | Expensive (~$1/hour) |
| Use case | Basic traffic filtering | Centralized, advanced egress control |

### DNS in Azure
- **Azure-provided DNS** — automatic, works out of the box within a VNet.
- **Azure Private DNS Zones** — custom DNS names that resolve to private IPs inside a VNet.
- **Azure DNS** — host your public DNS zones in Azure.

Private Endpoints use Private DNS Zones to override the public DNS resolution of PaaS services.

### Hub-Spoke Topology
The recommended network architecture for enterprise Azure:

```
Hub VNet (shared services)
  ├── Azure Firewall
  ├── VPN / ExpressRoute Gateway
  └── Bastion

Spoke VNets (workloads)
  ├── dev-vnet  ──── peered to hub
  ├── prod-vnet ──── peered to hub
  └── data-vnet ──── peered to hub
```

- All traffic between spokes goes through the hub firewall.
- Shared services (DNS, monitoring, connectivity) live in hub.
- Each spoke is isolated — dev cannot talk to prod directly.

### Application Gateway
Application Gateway is a **regional Layer 7 (HTTP/HTTPS) load balancer**. It understands HTTP — it can route based on URL path, hostname, inspect headers, terminate SSL, and sit in front of your web apps or AKS services.

```
Internet
  └── Application Gateway (public IP)
        ├── Listener: HTTPS :443
        ├── SSL Termination  ← decrypts here, backend gets plain HTTP
        ├── Routing Rules:
        │     /api/*    → backend pool: AKS ingress (10.0.1.x)
        │     /static/* → backend pool: Storage Account
        └── Health Probes  ← removes unhealthy backends automatically
```

**Key features:**
- **URL-based routing** — `/api/*` goes to one backend, `/images/*` to another.
- **Host-based routing** — `api.myapp.com` → AKS, `admin.myapp.com` → App Service.
- **SSL termination** — Application Gateway holds the certificate, backends don't need to handle TLS.
- **Session affinity** — sticky sessions via cookie (useful for stateful apps).
- **Autoscaling** — scales gateway instances based on traffic (v2 SKU).
- **Health probes** — continuously checks backends, removes unhealthy ones from rotation.

**SKUs:**
| SKU | Features |
|---|---|
| Standard v2 | Load balancing, SSL termination, URL routing, autoscaling |
| WAF v2 | Everything in Standard + Web Application Firewall |

> Always use **v2 SKU** — v1 is legacy and being retired.

---

### WAF (Web Application Firewall)
WAF is a feature you enable on Application Gateway (or Azure Front Door). It inspects HTTP traffic and blocks common web attacks **before they reach your application**.

WAF is built on the **OWASP Core Rule Set (CRS)** — a standard set of rules that detect and block the most common web attack patterns.

```
Internet
  └── Application Gateway + WAF
        ├── WAF inspects every request
        │     ├── BLOCK: SQL injection attempt in query string
        │     ├── BLOCK: XSS payload in header
        │     ├── BLOCK: directory traversal (../../etc/passwd)
        │     └── ALLOW: normal request → forward to backend
        └── Backend: AKS / App Service
```

**What WAF protects against (OWASP Top 10):**
- SQL Injection — `' OR 1=1 --` in form fields
- Cross-Site Scripting (XSS) — malicious `<script>` in inputs
- Command injection — OS commands embedded in requests
- Path traversal — `../../etc/passwd` style attacks
- Protocol violations — malformed HTTP requests

**WAF Modes:**
| Mode | Behavior |
|---|---|
| Detection | Logs attacks but does NOT block — use this first to tune rules |
| Prevention | Actively blocks malicious requests — use in production |

**Custom Rules:**
Beyond OWASP, you can write your own rules — block traffic from specific IPs, countries, or request patterns:
```
Custom Rule: Block requests where IP is in range 1.2.3.0/24
Custom Rule: Block requests where URI contains "/admin" and source is not corporate IP
```

**WAF on Front Door vs Application Gateway:**
- **Application Gateway + WAF** — protects a single regional app. Traffic hits the gateway in your region.
- **Azure Front Door + WAF** — protects globally. Blocks at Azure's edge PoPs worldwide before traffic even enters your region. Better for DDoS-scale attacks.

**Interview tip:** The key distinction is that NSG/Firewall operates at network level (IPs, ports) — WAF operates at application level (HTTP content). A malicious SQL injection payload on port 443 looks like valid traffic to an NSG. WAF reads the payload and blocks it.

---

## 3. Azure Identity & Security

### Azure AD / Entra ID
Microsoft Entra ID (formerly Azure AD) is Azure's identity platform. It handles authentication (who are you?) for Azure resources, Microsoft 365, and custom apps.

- **User** — a human identity with username/password.
- **Group** — collection of users. Assign RBAC to groups, not individual users.
- **Service Principal** — non-human identity for apps/services/pipelines to authenticate to Azure.
- **Managed Identity** — a special Service Principal automatically managed by Azure. No credentials to handle yourself.

### RBAC (Role-Based Access Control)
RBAC controls what an identity can do on Azure resources. It's assigned at a scope.

```
Role Assignment = WHO + WHAT + WHERE

WHO:  Service Principal (pipeline identity)
WHAT: Contributor role
WHERE: Resource Group "my-app-rg"
```

**Built-in roles:**
| Role | What it can do |
|---|---|
| Owner | Everything including access management |
| Contributor | Create/manage resources, no access management |
| Reader | Read-only |
| User Access Administrator | Manage access only |

**Principle of Least Privilege** — give only the minimum permissions needed. Never assign Owner when Contributor works. Never assign subscription-level when RG-level is enough.

### Managed Identity vs Service Principal

A Service Principal is an identity you manually create and manage. When you register an app in Azure AD, a Service Principal is created. You then generate a client secret (password) or certificate and store it somewhere — your pipeline, your config, your Key Vault.

A Managed Identity is an identity Azure creates, manages, and rotates automatically. There is no secret you ever see or store. The workload (VM, AKS pod, App Service) gets a token by calling the Azure Instance Metadata Service (IMDS) endpoint at 169.254.169.254 — Azure returns a short-lived token backed by that identity.


| | Managed Identity | Service Principal |
|---|---|---|
| Credentials | None — Azure manages it | Client ID + Secret/Certificate |
| Rotation | Automatic | Manual (or automated) |
| Use case | Azure resources talking to Azure | External apps, CI/CD pipelines |
| Types | System-assigned, User-assigned | N/A |

**System-assigned** — tied to one resource, deleted when resource is deleted.  
**User-assigned** — standalone identity, can be assigned to multiple resources.

> **Interview tip:** Always prefer Managed Identity over Service Principal when the workload runs inside Azure. No secrets to rotate or leak.

### Azure Policy
Azure Policy enforces rules on resources — it prevents non-compliant resources from being created or flags existing ones.

```
Policy: "All resources must have an 'Environment' tag"
Effect: Deny  ← blocks resource creation without the tag
```

**Effects:**
- `Deny` — blocks the operation
- `Audit` — allows but logs non-compliance
- `DeployIfNotExists` — auto-deploys a related resource if missing (e.g., auto-install monitoring agent)

Policies are grouped into **Initiatives** (policy sets). Applied at Management Group, Subscription, or RG scope.

### Azure Key Vault
Stores secrets, keys, and certificates. Access is controlled via RBAC or Access Policies.

- **Secrets** — passwords, connection strings, API keys
- **Keys** — cryptographic keys for encryption (HSM-backed available)
- **Certificates** — TLS certificates with auto-renewal

Best practice: Use Managed Identity to access Key Vault — no credentials needed.

---

## 4. Azure Compute

### Virtual Machines (VMs)
A VM is an IaaS offering — you manage the OS and everything above it. Azure manages the physical hardware.

Key concepts:
- **VM Size** — defines CPU, RAM, disk (e.g., `Standard_D4s_v3` = 4 vCPU, 16GB RAM)
- **VM Image** — OS template (Ubuntu 22.04, Windows Server 2022, custom images)
- **Availability Set** — distributes VMs across fault domains (different racks) and update domains. Protects against hardware failure and planned maintenance.
- **Availability Zone** — distributes VMs across datacenters. Higher protection than Availability Sets.

### VM Scale Sets (VMSS)
VMSS lets you run multiple identical VMs that can autoscale based on metrics. You define a template and Azure creates/destroys instances from it.

```
VMSS: web-vmss
  Template: Ubuntu 22.04, Standard_B2s, custom startup script
  Min: 2, Max: 10
  Scale out: CPU > 70% for 5 min → add 2 instances
  Scale in:  CPU < 30% for 10 min → remove 1 instance
```

- All instances are identical and stateless.
- Use a Load Balancer in front to distribute traffic.
- Use `upgrade_mode = Rolling` to update instances without downtime.

### Azure App Service
PaaS for hosting web apps, APIs, and functions. You don't manage VMs — Azure handles OS, patching, scaling.

- Runs on an **App Service Plan** which defines the compute tier (F1 free → P3v3 premium).
- Supports .NET, Java, Node.js, Python, PHP, containers.
- Built-in features: custom domains, SSL, deployment slots, autoscaling.

**Deployment Slots** — staging environments within the same App Service. You deploy to staging, test it, then **swap** to production with zero downtime. If something breaks, swap back.

### Azure Container Apps (ACA)
Serverless container hosting built on Kubernetes. You don't manage nodes or clusters — just deploy containers and set scaling rules.

- Scales to zero when idle (cost efficient).
- Built-in DAPR support, HTTP/event-based scaling via KEDA.
- Good for microservices, background workers, APIs that don't need full AKS control.

---

## 5. Azure Kubernetes Service (AKS)

### What is AKS?
AKS is a managed Kubernetes service. Azure manages the control plane (API server, etcd, scheduler) for free. You pay only for the worker nodes (VMs).

```
AKS Cluster
  ├── Control Plane  (managed by Azure, free)
  │     ├── API Server
  │     ├── etcd
  │     └── Scheduler
  └── Node Pools     (you pay for these VMs)
        ├── system pool  (runs AKS system pods)
        └── user pool    (runs your workloads)
```

### Node Pools
A node pool is a group of VMs with the same size and configuration. You can have multiple node pools in one cluster.

- **System node pool** — required, runs AKS system components (coredns, metrics-server).
- **User node pool** — runs your application workloads.
- Use separate pools for different workload types (GPU nodes, high-memory nodes).

### AKS Networking Modes

| Mode | How it works |
|---|---|
| Kubenet | Nodes get Azure VNet IPs, pods get a separate private range. NAT between pod and VNet. |
| Azure CNI | Every pod gets a real VNet IP. More IPs needed but pods are directly reachable. |
| Azure CNI Overlay | Pods get overlay IPs, nodes get VNet IPs. Better IP efficiency than CNI. |

**Azure CNI** is recommended for production — pods are directly accessible from the VNet, no NAT issues.

### AKS Authentication & RBAC
- **Local accounts** (deprecated) — username/password built into the cluster.
- **Azure AD integration** — users log in with their Azure AD identity. `kubectl` uses their Azure AD token.
- **Azure RBAC for Kubernetes** — Azure RBAC controls who can do what inside the cluster (not just access the Azure resource).

```bash
# Assign Kubernetes reader role to a team
az role assignment create \
  --role "Azure Kubernetes Service Cluster User Role" \
  --assignee user@company.com \
  --scope /subscriptions/.../resourceGroups/.../providers/Microsoft.ContainerService/managedClusters/my-aks
```

### AKS Scaling
- **Manual scaling** — `kubectl scale` or set node count in node pool.
- **Cluster Autoscaler** — automatically adds/removes nodes based on pending pods.
- **HPA (Horizontal Pod Autoscaler)** — scales pods based on CPU/memory.
- **KEDA** — event-driven autoscaling (scale on queue length, HTTP requests, etc.).

### AKS Upgrades
AKS regularly releases new Kubernetes versions. Upgrades are rolling — nodes are cordoned, drained, replaced one by one.

```bash
# Check available versions
az aks get-upgrades --resource-group my-rg --name my-aks

# Upgrade
az aks upgrade --resource-group my-rg --name my-aks --kubernetes-version 1.29.0
```

Use `max_surge` in node pool config to add extra nodes during upgrade (faster, no capacity reduction).

### AKS + Private Cluster
In a private cluster, the API server has no public IP — only accessible from within the VNet or via private peering. Recommended for production.

```
Developer laptop → VPN/Bastion → Jump VM → private API server
CI/CD pipeline  → private agent running in VNet → private API server
```

---

## 6. Azure Container Registry (ACR)

### What is ACR?
ACR is a managed Docker container registry in Azure — like Docker Hub but private and Azure-integrated.

**Tiers:**
| Tier | Use case |
|---|---|
| Basic | Dev/test, low storage |
| Standard | Production workloads |
| Premium | Geo-replication, private endpoints, 500GB+ storage |

### ACR + AKS Integration
AKS can pull images from ACR without credentials by attaching the ACR to the cluster. AKS uses a Managed Identity to authenticate.

```bash
# Attach ACR to AKS (grants AcrPull role)
az aks update --resource-group my-rg --name my-aks --attach-acr my-registry
```

After this, pods can pull from `myregistry.azurecr.io` without any imagePullSecrets.

### ACR Tasks
ACR Tasks can build container images directly in Azure — triggered by a git commit, base image update, or on a schedule. No need for a separate build agent.

```bash
az acr task create \
  --registry myregistry \
  --name build-on-push \
  --image myapp:{{.Run.ID}} \
  --context https://github.com/myorg/myapp.git \
  --branch main \
  --file Dockerfile
```

### Image Security
- **Vulnerability scanning** — Azure Defender for Containers scans images in ACR and running in AKS.
- **Content trust** — sign images so AKS only pulls verified images.
- **Geo-replication** (Premium) — replicate registry to multiple regions, AKS pulls from nearest replica.

---

## 7. Azure Storage

### Storage Account Types

| Service | Use case |
|---|---|
| Blob Storage | Unstructured data — files, images, backups, logs |
| Azure Files | SMB/NFS file shares — mountable by VMs or AKS |
| Queue Storage | Message queues for async communication |
| Table Storage | NoSQL key-value store (mostly legacy, use Cosmos DB instead) |

### Blob Storage Tiers
Data has different access frequency — pay less for data you access rarely:

| Tier | Access frequency | Cost |
|---|---|---|
| Hot | Frequently accessed | Higher storage cost, low access cost |
| Cool | Infrequently accessed (30+ days) | Lower storage cost, higher access cost |
| Cold | Rarely accessed (90+ days) | Even lower storage cost |
| Archive | Almost never accessed | Cheapest storage, hours to rehydrate |

Use **Lifecycle Management Policies** to automatically move data between tiers:
```json
"rule": {
  "action": { "baseBlob": { "tierToCool": { "daysAfterModificationGreaterThan": 30 } } }
}
```

### Storage Security
- **Private Endpoint** — disable public access, access only via VNet.
- **Storage Firewall** — whitelist specific VNet subnets or IP ranges.
- **SAS tokens** — time-limited, scoped access URLs for sharing blobs without exposing keys.
- **Managed Identity** — preferred over access keys for Azure services accessing storage.
- **Customer-managed keys (CMK)** — bring your own Key Vault key for encryption instead of Microsoft-managed keys.

### Azure Files for AKS
Azure Files can be mounted as a persistent volume in AKS — useful when multiple pods need to share the same storage (ReadWriteMany).

```yaml
# Kubernetes PVC using Azure Files
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  accessModes: ["ReadWriteMany"]   # multiple pods can mount it
  storageClassName: azurefile
```

---

## 8. Azure Key Vault

### What it stores
- **Secrets** — passwords, connection strings, API keys
- **Keys** — RSA/EC keys for encryption, signing (HSM-backed for compliance)
- **Certificates** — TLS certs with auto-renewal via DigiCert or Let's Encrypt

### Access Models
**Access Policies (legacy):**
- Vault-level permissions — if you have Get on secrets, you can get ALL secrets.
- Less granular, being deprecated.

**RBAC (recommended):**
- Standard Azure RBAC — assign roles at vault or individual secret level.
- `Key Vault Secrets User` — read secrets.
- `Key Vault Secrets Officer` — read and write secrets.

### Key Vault + AKS (CSI Driver)
The Secret Store CSI Driver lets AKS pods mount Key Vault secrets as files or environment variables — without any code changes.

```yaml
# Mount Key Vault secret as a file in a pod
volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: my-vault-provider
```

The pod's Managed Identity fetches the secret from Key Vault at pod startup. No secrets in Kubernetes Secrets or environment variables in plain text.

### Soft Delete & Purge Protection
- **Soft delete** — deleted secrets/keys go to a recycle bin (90 days by default). Recoverable.
- **Purge protection** — prevents permanent deletion during soft delete period. Required for CMK encryption.

---

## 9. Azure Monitor & Observability

### The 3 Pillars in Azure

| Pillar | Azure Service |
|---|---|
| Metrics | Azure Monitor Metrics |
| Logs | Log Analytics Workspace |
| Traces | Application Insights |

### Azure Monitor
The umbrella platform that collects metrics and logs from all Azure resources. Everything feeds into Azure Monitor.

- **Metrics** — numerical time-series data (CPU %, request count). 93-day retention. Near real-time.
- **Logs** — text/structured data (diagnostic logs, events). Stored in Log Analytics. Queried with KQL.
- **Alerts** — trigger actions (email, webhook, auto-scale) when a metric/log condition is met.

### Log Analytics Workspace
Central store for logs from all Azure resources. You query with **KQL (Kusto Query Language)**.

```kusto
// Find all errors in the last 24 hours
ContainerLog
| where TimeGenerated > ago(24h)
| where LogEntry contains "ERROR"
| summarize count() by bin(TimeGenerated, 1h)
| render timechart
```

Key sources that send to Log Analytics:
- AKS diagnostics and container logs
- VM performance and event logs
- Azure Activity Log (who did what in Azure)
- Application Insights telemetry

### Application Insights
APM (Application Performance Monitoring) for your application code. Tracks:
- Request rates, response times, failure rates
- Dependency calls (DB, external APIs)
- Exceptions and stack traces
- Custom events and metrics

Integrates with code via SDK or auto-instrumentation (zero code change for some languages).

### AKS Monitoring
- **Container Insights** — Azure Monitor add-on for AKS. Collects node/pod CPU, memory, logs automatically.
- **Prometheus + Grafana** — open-source stack, commonly used alongside Container Insights for more control.
- **Managed Prometheus** — Azure-hosted Prometheus, no infrastructure to manage.

```bash
# Enable Container Insights on AKS
az aks enable-addons \
  --resource-group my-rg \
  --name my-aks \
  --addons monitoring \
  --workspace-resource-id /subscriptions/.../workspaces/my-workspace
```

### Alerts & Action Groups
An **Action Group** defines what happens when an alert fires — email, SMS, webhook, Logic App, Azure Function.

```
Alert Rule:
  Condition: AKS node CPU > 80% for 5 minutes
  Severity: 2 (Warning)
  Action Group: → send email to team + post to Slack webhook
```

---

## 10. Azure DevOps & CI/CD

### Azure DevOps vs GitHub Actions

| | Azure DevOps Pipelines | GitHub Actions |
|---|---|---|
| Repo | Azure Repos or GitHub | GitHub |
| Best for | Enterprise, complex pipelines | Open source, GitHub-native |
| Self-hosted agents | Azure DevOps Agent Pools | Self-hosted runners |
| YAML | Yes (`azure-pipelines.yml`) | Yes (`.github/workflows/`) |
| Secrets | Variable Groups + Key Vault link | GitHub Secrets / Environments |

### Azure DevOps Pipeline Structure
```yaml
trigger:
  branches:
    include: [main]

stages:
  - stage: Build
    jobs:
      - job: BuildAndTest
        steps:
          - task: Docker@2
            inputs:
              command: buildAndPush
              repository: myapp
              containerRegistry: my-acr-service-connection

  - stage: DeployDev
    dependsOn: Build
    environment: dev
    jobs:
      - deployment: DeployToAKS
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  inputs:
                    action: deploy
                    manifests: k8s/deployment.yaml
```

### Service Connections
Service Connections are how Azure DevOps authenticates to Azure. Use **Workload Identity Federation** (no secrets) instead of storing a Service Principal secret.

```
Azure DevOps → Workload Identity Federation → Azure RBAC → deploy to AKS
```

No client secret stored in Azure DevOps — Azure AD issues a short-lived token per pipeline run.

### Pipeline Agents
- **Microsoft-hosted agents** — fresh VMs per run, pre-installed tools, no maintenance. Limited for private network access.
- **Self-hosted agents** — your VMs/containers running in your VNet. Needed for private AKS clusters, private ACR, internal resources.

For private AKS, the agent must be in the same VNet (or peered VNet) as the cluster to reach the private API server.

### Environment & Approvals
In Azure DevOps, an **Environment** represents a deployment target (dev, staging, prod). You can add:
- **Manual approval checks** — a person must approve before prod deploy starts
- **Branch control** — only `main` branch can deploy to prod
- **Business hours** — deploys only allowed during certain times

---

## 11. High Availability & Disaster Recovery

### Key Metrics
- **RTO (Recovery Time Objective)** — maximum acceptable downtime. "We can be down for 4 hours max."
- **RPO (Recovery Point Objective)** — maximum acceptable data loss. "We can lose at most 1 hour of data."
- **SLA** — Azure's uptime guarantee per service. Single VM = 99.9%, VM with AZ = 99.99%.

### HA Patterns in Azure

**For VMs:**
- Availability Zones — spread VMs across 3 datacenters in a region (99.99% SLA)
- Load Balancer in front — routes traffic only to healthy VMs
- VMSS with autoscaling — automatically replaces failed instances

**For AKS:**
- Spread node pools across Availability Zones
- Use pod disruption budgets to prevent all pods going down during node upgrades
- Run minimum 3 nodes in system pool across 3 zones

**For databases:**
- Azure SQL: Zone Redundant + Geo-replication to secondary region
- Cosmos DB: Multi-region writes for active-active

### Backup and DR

**Azure Backup:**
- Backs up VMs, Azure SQL, Azure Files, AKS (volumes)
- Stored in Recovery Services Vault or Backup Vault
- Configure retention policies (daily/weekly/monthly/yearly)

**Azure Site Recovery (ASR):**
- Replicates entire VMs to a secondary region continuously
- Failover RTO: minutes
- Used for full DR scenarios — not databases (use native DB replication for those)

**Geo-redundant Storage (GRS):**
- Blob storage is automatically replicated to a paired region
- Read-access GRS (RA-GRS) — you can read from secondary even before failover

### Azure Traffic Manager vs Front Door vs Load Balancer

| Service | Layer | Use case |
|---|---|---|
| Load Balancer | L4 (TCP/UDP) | Internal/external, same region |
| Application Gateway | L7 (HTTP) | Web apps, WAF, SSL termination, same region |
| Azure Front Door | L7 global | Global HTTP, CDN, WAF, multi-region routing |
| Traffic Manager | DNS level | Multi-region failover, geographic routing |

---

## 12. Azure Cost Management

### Key Cost Drivers
- **Compute** — VMs, AKS nodes running 24/7 even when idle
- **Storage** — data stored and egress (traffic leaving Azure)
- **Networking** — VNet peering cross-region, public IP, egress bandwidth
- **PaaS services** — databases, Key Vault operations, API calls

### Cost Optimization Practices

**Right-sizing:**
- Review VM sizes — most VMs are overprovisioned
- Use Azure Advisor recommendations for right-sizing
- Use burstable VMs (B-series) for workloads with variable CPU (dev/test)

**Reserved Instances:**
- Commit to 1 or 3 years → 40-72% discount vs pay-as-you-go
- Good for stable, predictable workloads (prod databases, always-on services)

**Spot VMs:**
- Up to 90% cheaper but can be evicted with 2-minute notice
- Good for batch jobs, dev/test, fault-tolerant workloads
- In AKS, use a spot node pool for non-critical workloads

**Dev/Test environments:**
- Shut down VMs outside business hours (Azure Automation schedule)
- Use ACA (scales to zero) instead of always-on AKS for dev
- Use lower SKUs — no need for Zone Redundancy in dev

**Tags for cost allocation:**
```hcl
tags = {
  Environment = "prod"
  Team        = "platform"
  CostCenter  = "engineering-001"
}
```
Tag resources so Azure Cost Management can break down spend by team/project.

### Azure Cost Management Tools
- **Cost Analysis** — visualize spend by resource group, tag, service type
- **Budgets** — set spending limits and alert when approaching threshold
- **Azure Advisor** — recommendations for cost, performance, security, reliability

---

## 13. Common Interview Questions & Answers

---

**Q: What is the difference between a Service Principal and a Managed Identity?**

A Service Principal is a manual identity you create — it has a client ID and a secret or certificate that you must store, rotate, and manage. A Managed Identity is an identity Azure creates and manages automatically — there are no credentials to handle. The identity's token is fetched directly from the Azure metadata service at runtime. For anything running inside Azure (VMs, AKS pods, App Service), always use Managed Identity. Service Principals are for external systems like CI/CD pipelines running outside Azure.

---

**Q: How do you secure an AKS cluster?**

Several layers:
- **Control plane** — enable private cluster so API server has no public IP
- **Authentication** — integrate with Azure AD, disable local accounts
- **Authorization** — use Azure RBAC for Kubernetes, assign least privilege roles
- **Network** — use Azure CNI, NSGs on node subnet, Azure Policy to restrict pod-to-pod traffic
- **Image security** — pull only from private ACR, enable Defender for Containers for vulnerability scanning
- **Secrets** — use Key Vault CSI driver, no secrets hardcoded in manifests
- **Node security** — keep Kubernetes version updated, use node auto-upgrade

---

**Q: Explain VNet Peering and its limitations.**

VNet Peering connects two VNets so they communicate over the Azure private backbone — no internet, no gateway needed. The traffic is low latency and high bandwidth. The key limitation is that peering is **not transitive** — if A peers with B and B peers with C, A cannot reach C through B. For that you need either a direct A-C peering or a hub VNet with Azure Firewall or VPN Gateway acting as a transit. Also, peered VNets cannot have overlapping IP address spaces.

---

**Q: What is the difference between Azure Front Door and Application Gateway?**

Application Gateway is a regional Layer 7 load balancer — it handles HTTP/HTTPS traffic for apps within a single region, with WAF, SSL termination, and URL-based routing. Azure Front Door is a global Layer 7 service — it sits at Azure's edge PoPs worldwide, provides CDN, global load balancing, WAF, and can route traffic to backends across multiple regions with health-based failover. Use Application Gateway for single-region apps, Front Door when you need global distribution or multi-region routing.

---

**Q: How does AKS autoscaling work?**

AKS has two levels of autoscaling:
- **HPA (Horizontal Pod Autoscaler)** — scales the number of pod replicas based on CPU, memory, or custom metrics. Works within existing nodes.
- **Cluster Autoscaler** — when pods can't be scheduled because nodes are full, it adds new nodes. When nodes are underutilized, it removes them.

They work together: HPA scales pods up, Cluster Autoscaler adds nodes if needed to fit those pods. For event-driven scaling (queue depth, HTTP requests), KEDA extends HPA with external metrics.

---

**Q: What is the difference between Azure Policy and RBAC?**

RBAC controls **who can perform actions** on resources — it's about permissions. "This identity can create VMs in this resource group." Azure Policy controls **what configuration resources can have** — it's about compliance. "All VMs must have a specific tag" or "Storage accounts must disable public access." RBAC is about access management; Policy is about governance and compliance. You need both — RBAC so only authorized people can deploy, Policy so what they deploy meets your standards.

---

**Q: How do you handle secrets in a CI/CD pipeline deploying to AKS?**

Several approaches depending on the target:
- **Pipeline secrets** — store in Azure DevOps Variable Groups linked to Key Vault, or GitHub Secrets for GitHub Actions. Injected as environment variables at pipeline runtime, never stored in code.
- **At runtime in AKS** — use the Secret Store CSI Driver with Managed Identity. The pod's identity fetches secrets from Key Vault at startup and mounts them as files or syncs them to Kubernetes Secrets. No secrets in Docker images, manifests, or environment variables in plain text.
- **Key principle** — secrets should only exist in Key Vault as the single source of truth. Everything else fetches from there at runtime with zero hardcoding.

---

**Q: What happens during an AKS node upgrade?**

Azure cordons the node (marks it unschedulable for new pods), drains it (evicts all pods to other nodes gracefully), then deletes the old node and creates a new one with the upgraded OS/Kubernetes version. The new node starts up and joins the cluster. This happens node by node. If you have `max_surge` configured, Azure creates new upgraded nodes first before draining old ones — so cluster capacity never decreases during the upgrade. Pod Disruption Budgets (PDBs) are respected during drain to ensure minimum replicas stay running.

---

**Q: What is Azure Private Endpoint and when do you use it?**

A Private Endpoint creates a network interface with a private IP in your VNet that maps to a specific PaaS service instance. Traffic to that service goes over the VNet backbone, never the public internet. You use it whenever a PaaS service handles sensitive data or must not be publicly accessible — Azure SQL, Key Vault, ACR, Storage Accounts for Terraform state, Service Bus. Combined with disabling public network access on the service, it ensures zero exposure to the internet. You also need a Private DNS Zone so the service's public hostname resolves to the private IP inside the VNet.

---

**Q: How do you design for cost efficiency in a multi-environment Azure setup?**

For production: right-sized VMs with Reserved Instances for predictable workloads, Zone Redundancy only where SLA requires it, Spot node pools in AKS for non-critical workloads.
For dev/staging: auto-shutdown VMs on a schedule, use ACA (scales to zero) instead of always-on AKS, burstable B-series VMs, no Zone Redundancy needed. Tag everything by environment and team for cost allocation visibility in Cost Management. Set budget alerts so no environment silently overruns. Review Azure Advisor monthly for right-sizing recommendations.

---

*Last updated: April 2026 | Covers Azure for Platform & DevOps Engineers*
