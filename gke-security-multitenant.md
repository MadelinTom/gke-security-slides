---
marp: true
theme: default
paginate: true
header: 'GKE Security & Migration'
footer: 'Multi-Tenant Cluster Architecture | Azure → GCP Migration'
---

# GKE Security Best Practices
## Multi-Tenant Cluster Architecture
## + Azure → GCP Migration Strategy

*Workshop: Security, Networking, and Cutover Planning*

---

# Agenda

**Part 1: GKE Multi-Tenancy**
1. Namespace isolation & Workload Identity
2. Network architecture for 20 deployments

**Part 2: Cross-Cloud Networking**
3. Azure ↔ GCP connectivity options
4. DNS, Firewall rules, GCP service ranges

**Part 3: Migration & Cutover**
5. App/API cutover strategies
6. On-premises LDAP integration

---

<!-- _class: lead -->

# Part 1
## GKE Multi-Tenancy

---

# Multi-Tenancy Model

## Namespace-Per-App Architecture

```
cluster/
├── app-orders/           # Namespace
│   ├── deployment
│   ├── service
│   └── configmap
├── app-inventory/        # Namespace
├── app-payments/         # Namespace
└── ... (20 namespaces)
```

**Why namespaces?**
- Logical isolation without cluster overhead
- RBAC scoping per team
- Resource quota enforcement
- Network policy boundaries

---

# Identity: Workload Identity Federation

```
┌─────────────────────────────────────────────────────┐
│ GKE Cluster                                          │
│  ┌──────────────────┐                               │
│  │ Pod (app-orders) │                               │
│  │ SA: orders-sa    │ ──────┐                       │
│  └──────────────────┘       │ WIF                   │
└───────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────┐
│ GCP IAM                                              │
│  orders-app@project.iam.gserviceaccount.com         │
│  Roles: Storage Object Viewer, Pub/Sub Publisher    │
└─────────────────────────────────────────────────────┘
```

**Per-app isolation:** K8s SA → GCP SA → Least privilege IAM

---

# Project Structure

```
org/
├── platform-gke-prod/          # Shared GKE cluster
├── app-orders-prod/            # Orders app resources
│   ├── Cloud SQL
│   ├── Cloud Storage
│   └── Pub/Sub topics
├── app-inventory-prod/         # Inventory app resources
└── shared-services-prod/       # Logging, monitoring
```

| Resource Type | Recommendation |
|---------------|----------------|
| GKE Cluster | Shared (platform project) |
| Cloud SQL | Dedicated per app |
| Secret Manager | Dedicated per app |
| VPC/Networking | Shared (host project) |

---

# Network: Single Ingress for 20 APIs

```
                    ┌──────────────────────┐
                    │   Cloud Load Balancer │
                    │   + Cloud Armor (WAF) │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │   GKE Gateway API     │
                    │   (path-based routing)│
                    └──────────┬───────────┘
        ┌──────────┬───────────┼───────────┬──────────┐
        ▼          ▼           ▼           ▼          ▼
   /orders    /inventory   /payments    /users    /app-20
```

**Cost:** 1 LB vs 20 = significant savings
**Recommendation:** Skip API Gateway unless you need monetization

---

<!-- _class: lead -->

# Part 2
## Cross-Cloud Networking
### Azure ↔ GCP Connectivity

---

# Connectivity Options

| Option | Latency | Bandwidth | Cost | Complexity |
|--------|---------|-----------|------|------------|
| **VPN (HA)** | ~50ms | 3 Gbps | $ | Low |
| **Dedicated Interconnect** | ~10ms | 10-100 Gbps | $$$ | High |
| **Partner Interconnect** | ~20ms | 50 Mbps-10 Gbps | $$ | Medium |
| **Internet (Public)** | Variable | N/A | $ | Low |

## 💬 Discussion Point
*What's your latency tolerance during migration? 
How much cross-cloud traffic do you expect?*

---

# Option 1: HA VPN (Recommended for Migration)

```
┌─────────────────────┐                    ┌─────────────────────┐
│      Azure VNet     │                    │       GCP VPC       │
│   10.1.0.0/16       │                    │    10.2.0.0/16      │
│                     │                    │                     │
│  ┌───────────────┐  │    IPsec Tunnels   │  ┌───────────────┐  │
│  │ VPN Gateway   │◄─┼────────────────────┼─►│ Cloud VPN     │  │
│  │ (Active-Active)│ │    (4 tunnels)     │  │ (HA Gateway)  │  │
│  └───────────────┘  │                    │  └───────────────┘  │
│                     │                    │                     │
│  Azure Apps         │                    │  GKE Cluster        │
└─────────────────────┘                    └─────────────────────┘
```

**Throughput:** 3 Gbps per tunnel (12 Gbps with 4 tunnels)
**SLA:** 99.99% with HA configuration

---

# VPN Configuration

## Azure Side
```bash
# Create VPN Gateway (takes ~45 mins)
az network vnet-gateway create \
  --name azure-vpn-gw \
  --resource-group rg-network \
  --vnet hub-vnet \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw2 \
  --generation Generation2
```

## GCP Side
```bash
# Create HA VPN Gateway
gcloud compute vpn-gateways create gcp-vpn-gw \
  --network=shared-vpc \
  --region=australia-southeast1
```

---

# Option 2: Dedicated Interconnect

```
┌──────────────┐     ┌─────────────────────┐     ┌──────────────┐
│    Azure     │     │   Colocation Facility │     │     GCP      │
│              │     │   (e.g., Equinix SY1) │     │              │
│  ExpressRoute├────►│                       │◄────┤ Interconnect │
│              │     │  Cross-connect fiber  │     │              │
└──────────────┘     └─────────────────────────┘     └──────────────┘
```

**Best for:** High-bandwidth, low-latency requirements
**Lead time:** 2-4 weeks for provisioning
**Cost:** $1,700/month per 10 Gbps port + colo fees

## 💬 Discussion Point
*Is the extra cost/complexity justified for your traffic patterns?*

---

# IP Address Planning

## Non-Overlapping CIDR Ranges

| Environment | Azure CIDR | GCP CIDR |
|-------------|------------|----------|
| Production | 10.1.0.0/16 | 10.2.0.0/16 |
| Non-Prod | 10.11.0.0/16 | 10.12.0.0/16 |
| Management | 10.100.0.0/24 | 10.100.1.0/24 |

## GKE-Specific Ranges
```
Pod CIDR:     10.2.0.0/14   (262,144 IPs)
Service CIDR: 10.6.0.0/20   (4,096 IPs)
Master CIDR:  172.16.0.0/28 (Private endpoint)
```

⚠️ **Critical:** Ensure no overlap with Azure or on-prem ranges

---

# DNS Architecture

## Hybrid DNS Resolution

```
┌────────────────────────────────────────────────────────────────┐
│                     On-Premises DNS                            │
│                   (Active Directory)                           │
│                  corp.example.com                               │
└───────────────────────────┬────────────────────────────────────┘
                            │ Conditional forwarding
        ┌───────────────────┴───────────────────┐
        ▼                                       ▼
┌───────────────────┐                 ┌───────────────────┐
│   Azure DNS       │                 │   GCP Cloud DNS   │
│ Private Zones     │                 │   Private Zones   │
│                   │                 │                   │
│ *.azure.internal  │◄───────────────►│ *.gcp.internal    │
│ *.database.azure  │   Forwarding    │ *.pkg.dev         │
└───────────────────┘                 └───────────────────┘
```

---

# DNS Configuration

## GCP Cloud DNS Private Zone
```bash
# Create private zone for GCP resources
gcloud dns managed-zones create gcp-internal \
  --dns-name="gcp.internal." \
  --visibility=private \
  --networks=shared-vpc

# Forwarding zone for Azure resolution
gcloud dns managed-zones create azure-forward \
  --dns-name="azure.internal." \
  --visibility=private \
  --networks=shared-vpc \
  --forwarding-targets="10.1.0.4,10.1.0.5"  # Azure DNS IPs
```

## Azure DNS Forwarding
```powershell
# Conditional forwarder for GCP
Add-DnsServerConditionalForwarderZone `
  -Name "gcp.internal" `
  -MasterServers 10.2.0.2  # GCP DNS inbound endpoint
```

---

# Firewall Rules: GCP Side

## Required Ingress Rules

| Priority | Source | Destination | Ports | Purpose |
|----------|--------|-------------|-------|---------|
| 1000 | Azure VNet (10.1.0.0/16) | GKE Nodes | 443, 8080 | API traffic |
| 1000 | Azure VNet | Cloud SQL | 5432, 3306 | Database |
| 1000 | On-prem (10.100.0.0/24) | All | 22 | SSH mgmt |
| 65534 | 0.0.0.0/0 | All | All | Deny all |

```bash
gcloud compute firewall-rules create allow-azure-to-gke \
  --network=shared-vpc \
  --allow=tcp:443,tcp:8080 \
  --source-ranges=10.1.0.0/16 \
  --target-tags=gke-node
```

---

# GCP Service Ranges to Allow

## Outbound from Azure to GCP Services

| Service | Range/Domain | Port |
|---------|--------------|------|
| GKE API | 142.250.0.0/15 or *.googleapis.com | 443 |
| Cloud SQL | Private IP in VPC | 5432/3306 |
| Cloud Storage | *.storage.googleapis.com | 443 |
| Artifact Registry | *.pkg.dev | 443 |
| Cloud Logging | logging.googleapis.com | 443 |
| IAM/Auth | *.googleapis.com | 443 |

## 💬 Discussion Point
*Private Google Access vs Internet egress for GCP APIs?*

---

# Private Google Access

## Keep Traffic Off Public Internet

```
┌─────────────────────────────────────────────────────────────┐
│ GCP VPC (Private Google Access Enabled)                     │
│                                                             │
│  GKE Pod ──► 199.36.153.8/30 ──► googleapis.com             │
│             (private.googleapis.com)                        │
└─────────────────────────────────────────────────────────────┘
```

```bash
# Enable Private Google Access on subnet
gcloud compute networks subnets update gke-subnet \
  --region=australia-southeast1 \
  --enable-private-ip-google-access
```

**Benefit:** No public IPs needed for GCP API access

---

<!-- _class: lead -->

# Part 3
## Migration & Cutover Strategy

---

# Cutover Patterns

## Option A: Big Bang 💥
- All apps cut over at once
- Single maintenance window
- High risk, high coordination

## Option B: Strangler Fig 🌿 (Recommended)
- Migrate one app at a time
- Route traffic progressively
- Rollback per-app if needed

## Option C: Blue-Green 🔵🟢
- Full parallel environment
- DNS switch at cutover
- Highest cost, lowest risk

---

# Strangler Fig Pattern

```
Phase 1: 10% Traffic          Phase 2: 50% Traffic         Phase 3: 100% Traffic
┌──────────────────┐          ┌──────────────────┐         ┌──────────────────┐
│   Load Balancer  │          │   Load Balancer  │         │   Load Balancer  │
└────────┬─────────┘          └────────┬─────────┘         └────────┬─────────┘
         │                             │                            │
    ┌────┴────┐                   ┌────┴────┐                       │
    ▼         ▼                   ▼         ▼                       ▼
┌──────┐  ┌──────┐           ┌──────┐  ┌──────┐              ┌──────────┐
│Azure │  │ GCP  │           │Azure │  │ GCP  │              │   GCP    │
│ 90%  │  │ 10%  │           │ 50%  │  │ 50%  │              │  100%    │
└──────┘  └──────┘           └──────┘  └──────┘              └──────────┘
```

**Tools:** Traffic Manager (Azure), Cloud Load Balancing, Weighted routing

---

# API Cutover Strategy

## Per-API Migration Runbook

| Step | Action | Rollback |
|------|--------|----------|
| 1 | Deploy to GKE (shadow mode) | Delete deployment |
| 2 | Synthetic traffic testing | N/A |
| 3 | 10% canary traffic | Route 100% Azure |
| 4 | Monitor error rates/latency | Route 100% Azure |
| 5 | 50% traffic split | Route 100% Azure |
| 6 | 100% to GCP | Route 100% Azure |
| 7 | Decommission Azure | N/A |

**Monitoring:** Error rate < 0.1%, P99 latency within SLO

---

# Traffic Routing Options

## Option 1: DNS-Based (Simple)

```
api.example.com
    │
    ▼
┌─────────────────────────────┐
│ Azure Traffic Manager       │
│ or                          │
│ Cloud DNS (weighted routing)│
└─────────────────────────────┘
    │
    ├──► Azure App Service (weight: 50)
    │
    └──► GCP Load Balancer (weight: 50)
```

**Pros:** Simple, works everywhere
**Cons:** DNS TTL delays, no request-level control

---

# Traffic Routing Options

## Option 2: Global Load Balancer (Recommended)

```
                    ┌────────────────────────┐
                    │ GCP Global LB          │
                    │ (External HTTP(S))     │
                    └───────────┬────────────┘
                                │
                    ┌───────────┴───────────┐
                    │   Traffic Director    │
                    │   (header/weight)     │
                    └───────────┬───────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │ Azure Backend   │ │ GKE Backend     │ │ GKE Backend     │
    │ (NEG)           │ │ (orders)        │ │ (inventory)     │
    └─────────────────┘ └─────────────────┘ └─────────────────┘
```

**Pros:** Request-level routing, header-based canary, single IP

---

# Cutover Checklist

## Pre-Cutover
- [ ] Synthetic tests passing on GCP
- [ ] Monitoring/alerting configured
- [ ] Runbook documented
- [ ] Rollback tested
- [ ] Stakeholder comms sent

## During Cutover
- [ ] Start with 10% traffic
- [ ] Monitor for 30 mins
- [ ] Increment to 50%, monitor
- [ ] Increment to 100%

## Post-Cutover
- [ ] Confirm all traffic on GCP
- [ ] Keep Azure running 48-72h
- [ ] Decommission Azure resources

---

# Database Cutover

## Options for Stateful Workloads

| Pattern | Downtime | Complexity | Data Loss Risk |
|---------|----------|------------|----------------|
| **Dump & Restore** | Hours | Low | Medium |
| **Replication** | Minutes | Medium | Low |
| **DMS (Database Migration Service)** | Minutes | Low | Low |
| **Dual-Write** | Zero | High | Medium |

## 💬 Discussion Point
*What's your acceptable downtime window?
Any active-active requirements?*

---

<!-- _class: lead -->

# Part 4
## On-Premises LDAP Integration

---

# LDAP Integration Options

## Option 1: Cloud Identity + LDAP Sync
```
On-Prem AD/LDAP ──► Google Cloud Directory Sync ──► Cloud Identity
                          (scheduled sync)
```

## Option 2: Workload Identity + Direct LDAP
```
GKE Pod ──► VPN ──► On-Prem LDAP (port 389/636)
```

## Option 3: LDAP Proxy in GCP
```
GKE Pod ──► LDAP Proxy (GCE) ──► VPN ──► On-Prem LDAP
```

---

# Option 1: Google Cloud Directory Sync

## Best for: User Authentication

```
┌──────────────────────────────────────────────────────────────┐
│                     On-Premises                              │
│  ┌─────────────┐      ┌─────────────────────────────────┐   │
│  │ AD / LDAP   │◄────►│ Google Cloud Directory Sync     │   │
│  │             │      │ (runs on Windows server)        │   │
│  └─────────────┘      └───────────────┬─────────────────┘   │
└───────────────────────────────────────┼─────────────────────┘
                                        │ HTTPS (443)
                                        ▼
                            ┌───────────────────────┐
                            │   Google Cloud        │
                            │   Identity            │
                            └───────────────────────┘
                                        │
                                        ▼
                            ┌───────────────────────┐
                            │   GKE / IAP /         │
                            │   Cloud Console       │
                            └───────────────────────┘
```

---

# GCDS Configuration

```yaml
# Example GCDS config
ldap:
  hostname: ldap.corp.example.com
  port: 636
  ssl: true
  baseDn: DC=corp,DC=example,DC=com
  
google:
  domain: example.com
  adminEmail: admin@example.com
  
sync:
  users:
    filter: "(&(objectClass=user)(memberOf=CN=GCP-Users,OU=Groups,DC=corp,DC=example,DC=com))"
    attributes:
      email: mail
      firstName: givenName
      lastName: sn
  groups:
    filter: "(objectClass=group)"
```

**Sync frequency:** Every 1-4 hours (configurable)

---

# Option 2: Direct LDAP from GKE

## For Application-Level Auth (e.g., API auth)

```yaml
# Kubernetes Secret for LDAP bind credentials
apiVersion: v1
kind: Secret
metadata:
  name: ldap-credentials
  namespace: app-orders
type: Opaque
stringData:
  bind-dn: "CN=svc-gke-ldap,OU=Service Accounts,DC=corp,DC=example,DC=com"
  bind-password: "your-password"
---
# Application config
apiVersion: v1
kind: ConfigMap
metadata:
  name: ldap-config
data:
  LDAP_HOST: "ldap.corp.example.com"
  LDAP_PORT: "636"
  LDAP_BASE_DN: "DC=corp,DC=example,DC=com"
  LDAP_USER_FILTER: "(sAMAccountName={0})"
```

---

# Firewall Rules for LDAP

## GCP to On-Prem (via VPN)

| Source | Destination | Port | Protocol |
|--------|-------------|------|----------|
| GKE Pod CIDR | LDAP Server | 389 | TCP (LDAP) |
| GKE Pod CIDR | LDAP Server | 636 | TCP (LDAPS) |
| GKE Pod CIDR | AD DC | 88 | TCP/UDP (Kerberos) |
| GKE Pod CIDR | AD DC | 464 | TCP/UDP (Kerberos pwd) |

```bash
# On-prem firewall (example)
az network nsg rule create \
  --name allow-gke-ldap \
  --nsg-name on-prem-nsg \
  --priority 100 \
  --source-address-prefixes 10.2.0.0/14 \
  --destination-port-ranges 636 \
  --protocol Tcp
```

---

# Option 3: LDAP Proxy (Recommended)

## Benefits of a Proxy Layer

```
┌────────────────────────────────────────────────────────────────┐
│ GCP VPC                                                        │
│                                                                │
│  ┌─────────────┐        ┌─────────────────┐                   │
│  │ GKE Pods    │───────►│ LDAP Proxy      │                   │
│  │             │        │ (GCE or GKE)    │                   │
│  └─────────────┘        │                 │                   │
│                         │ - Connection    │                   │
│                         │   pooling       │                   │
│                         │ - TLS termination│                   │
│                         │ - Caching       │                   │
│                         │ - Failover      │                   │
│                         └────────┬────────┘                   │
└──────────────────────────────────┼────────────────────────────┘
                                   │ VPN
                                   ▼
                         ┌─────────────────┐
                         │ On-Prem LDAP    │
                         └─────────────────┘
```

---

# LDAP Decision Matrix

| Requirement | GCDS | Direct LDAP | LDAP Proxy |
|-------------|------|-------------|------------|
| User SSO to GCP Console | ✅ | ❌ | ❌ |
| Application auth | ⚠️ | ✅ | ✅ |
| Real-time auth | ❌ | ✅ | ✅ |
| Connection pooling | N/A | ❌ | ✅ |
| High availability | ✅ | ⚠️ | ✅ |
| Complexity | Low | Low | Medium |

## 💬 Discussion Point
*What's your primary LDAP use case?
- User SSO to GCP services?
- Application-level authentication?
- Both?*

---

# Complete Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                              Internet                                      │
└─────────────────────────────────┬─────────────────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     GCP Global Load Balancer + Cloud Armor                   │
└─────────────────────────────────┬─────────────────────────────────────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ GKE Cluster (platform-gke-prod)              │ Azure (during migration)    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐        │  ┌─────────────────────┐     │
│  │app-orders│ │app-inv  │ │app-pay  │        │  │ Legacy Apps         │     │
│  │ WIF+SA  │ │ WIF+SA  │ │ WIF+SA  │        │  │ (traffic split)     │     │
│  └────┬────┘ └────┬────┘ └────┬────┘        │  └─────────────────────┘     │
└───────┼──────────────────────────────────────┼─────────────────────────────┘
        │ VPN (HA)                             │
        ▼                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           On-Premises                                        │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                  │
│   │ LDAP / AD    │    │  DNS         │    │  Databases   │                  │
│   └──────────────┘    └──────────────┘    └──────────────┘                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

<!-- _class: lead -->

# Part 5
## VM-Based Workloads
### Networking & Security

---

# VM Networking Architecture

## Shared VPC with Service Projects

```
┌─────────────────────────────────────────────────────────────────┐
│ Host Project: platform-network-prod                              │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Shared VPC                            │  │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────┐ │  │
│  │  │ web-subnet     │  │ app-subnet     │  │ db-subnet  │ │  │
│  │  │ 10.2.0.0/24    │  │ 10.2.1.0/24    │  │ 10.2.2.0/24│ │  │
│  │  └────────────────┘  └────────────────┘  └────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
         │                      │                    │
         ▼                      ▼                    ▼
   ┌──────────┐          ┌──────────┐         ┌──────────┐
   │ web-prod │          │ app-prod │         │ db-prod  │
   │ (service │          │ (service │         │ (service │
   │  project)│          │  project)│         │  project)│
   └──────────┘          └──────────┘         └──────────┘
```

---

# Subnet Design for VMs

## Recommended Tier Layout

| Subnet | CIDR | Purpose | Firewall Tag |
|--------|------|---------|--------------|
| web-tier | 10.2.0.0/24 | Web/Proxy VMs | `web-vm` |
| app-tier | 10.2.1.0/24 | App VMs | `app-vm` |
| db-tier | 10.2.2.0/24 | Database VMs | `db-vm` |
| mgmt | 10.2.10.0/24 | Bastion, monitoring | `mgmt-vm` |

```bash
gcloud compute networks subnets create app-tier \
  --network=shared-vpc \
  --region=australia-southeast1 \
  --range=10.2.1.0/24 \
  --enable-private-ip-google-access
```

---

# VM Firewall Rules

## Defense in Depth

```
Internet ──► Cloud Armor/WAF ──► LB ──► web-tier ──► app-tier ──► db-tier
                                           │            │            │
                                     port 80/443    port 8080    port 5432
```

```bash
# Web tier: Allow from LB only
gcloud compute firewall-rules create allow-lb-to-web \
  --network=shared-vpc \
  --allow=tcp:80,tcp:443 \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=web-vm

# App tier: Allow from web tier only
gcloud compute firewall-rules create allow-web-to-app \
  --network=shared-vpc \
  --allow=tcp:8080 \
  --source-tags=web-vm \
  --target-tags=app-vm
```

---

# Private Google Access for VMs

## Access GCP APIs Without Public IP

```
┌────────────────────────────────────────────────────────────┐
│ GCP VPC (Private Google Access Enabled)                    │
│                                                            │
│  VM (no external IP)                                       │
│    │                                                       │
│    └──► 199.36.153.8/30 ──► Cloud Storage                 │
│         (private.googleapis.com)                           │
│                          ──► Secret Manager                │
│                          ──► Artifact Registry             │
└────────────────────────────────────────────────────────────┘
```

**Enable on subnet:**
```bash
gcloud compute networks subnets update app-tier \
  --region=australia-southeast1 \
  --enable-private-ip-google-access
```

---

# Cloud NAT for Outbound

## VMs Without Public IPs Still Need Updates

```
┌─────────────────────────────────────────────────────────────┐
│ VPC                                                          │
│  ┌─────────────┐                                            │
│  │ VM (no ext IP)│──► Cloud NAT ──► Internet (egress only) │
│  └─────────────┘      │                                     │
│                       │                                     │
│  ┌─────────────┐      │                                     │
│  │ VM (no ext IP)│────┘                                     │
│  └─────────────┘                                            │
└─────────────────────────────────────────────────────────────┘
```

```bash
gcloud compute routers create nat-router \
  --network=shared-vpc --region=australia-southeast1

gcloud compute routers nats create nat-config \
  --router=nat-router --region=australia-southeast1 \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips
```

---

# Load Balancing for VMs

## Options Comparison

| LB Type | Use Case | Health Check | SSL | 
|---------|----------|--------------|-----|
| **External HTTP(S)** | Public web apps | HTTP/HTTPS | Yes, managed |
| **Internal HTTP(S)** | Internal services | HTTP/HTTPS | Yes |
| **TCP/UDP Network** | Non-HTTP (DBs, games) | TCP | Pass-through |
| **Internal TCP/UDP** | Internal non-HTTP | TCP/UDP | No |

**Recommendation:** External HTTP(S) LB for public, Internal for inter-tier

---

# VM Load Balancer Architecture

```
                     ┌─────────────────────────┐
                     │   Global External LB    │
                     │   + Cloud Armor (WAF)   │
                     └───────────┬─────────────┘
                                 │
                     ┌───────────┴───────────┐
                     │   Backend Service     │
                     │   (health checks)     │
                     └───────────┬───────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            ▼                    ▼                    ▼
     ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
     │  web-vm-1    │    │  web-vm-2    │    │  web-vm-3    │
     │  (MIG)       │    │  (MIG)       │    │  (MIG)       │
     └──────────────┘    └──────────────┘    └──────────────┘
```

**MIG:** Managed Instance Group (auto-scaling, auto-healing)

---

# VM to Azure Cross-Cloud

## Same VPN, Different Targets

```
┌─────────────────────┐                    ┌─────────────────────┐
│      Azure VNet     │                    │       GCP VPC       │
│                     │                    │                     │
│  Azure VMs          │    HA VPN          │  GCE VMs            │
│  10.1.1.0/24        │◄──────────────────►│  10.2.1.0/24        │
│                     │                    │                     │
│  Azure DBs          │                    │  Cloud SQL          │
│  10.1.2.0/24        │                    │  10.2.2.0/24        │
└─────────────────────┘                    └─────────────────────┘
```

**Same connectivity** as GKE - VPN/Interconnect works for all workload types

---

# VM Firewall for Cross-Cloud

## Azure VMs ↔ GCP VMs

```bash
# Allow Azure VMs to reach GCP app tier
gcloud compute firewall-rules create allow-azure-to-app-vms \
  --network=shared-vpc \
  --allow=tcp:8080,tcp:443 \
  --source-ranges=10.1.0.0/16 \
  --target-tags=app-vm

# Allow GCP VMs to reach Azure during migration
gcloud compute firewall-rules create allow-gcp-to-azure \
  --network=shared-vpc \
  --direction=EGRESS \
  --allow=tcp:443,tcp:8080 \
  --destination-ranges=10.1.0.0/16 \
  --target-tags=app-vm
```

---

<!-- _class: lead -->

# Part 6
## Database Architecture
### Cloud SQL, AlloyDB, and Self-Managed

---

# Database Options Overview

| Service | Type | Best For | HA | Max Storage |
|---------|------|----------|-----|-------------|
| **Cloud SQL** | Managed MySQL/PG/SQL Server | Traditional apps | Regional | 64 TB |
| **AlloyDB** | Managed PG-compatible | High-perf analytics | Regional | 128 TB |
| **GCE + DB** | Self-managed | Legacy, licensing | DIY | Unlimited |
| **Cloud Spanner** | Distributed SQL | Global scale | Multi-region | Unlimited |

---

# Multi-Tenant DB Strategy

## The Big Question: Instance vs Database Isolation

```
Option A: 1 Instance per Environment        Option B: Shared Instance, Multiple DBs
┌─────────────────────────────────────┐    ┌─────────────────────────────────────┐
│ sql-instance-prod                   │    │ sql-instance-prod                   │
│   └── orders-db                     │    │   ├── orders-db                     │
│                                     │    │   ├── inventory-db                  │
│ sql-instance-nonprod                │    │   ├── payments-db                   │
│   └── orders-db                     │    │   └── users-db                      │
│                                     │    │                                     │
│ sql-instance-dev                    │    │ sql-instance-nonprod                │
│   └── orders-db                     │    │   ├── orders-db                     │
└─────────────────────────────────────┘    │   └── ... (all apps)                │
                                           └─────────────────────────────────────┘
```

---

# Instance vs Database: Trade-offs

| Factor | 1 Instance/Env | Shared Instance |
|--------|----------------|-----------------|
| **Cost** | Higher (N instances) | Lower (fewer instances) |
| **Isolation** | Strong (CPU/memory) | Weak (noisy neighbor) |
| **Maintenance** | More windows | Fewer windows |
| **IAM** | Instance-level | Database-level (limited) |
| **Backups** | Per-instance | All DBs together |
| **Scaling** | Independent | Shared limits |

## 💬 Recommendation
**Prod:** Separate instances per app (or app group)
**Non-prod:** Shared instance, multiple databases

---

# Project Structure: Databases

## Dedicated DB Projects

```
org/
├── platform-db-prod/           # Shared DB infrastructure
│   ├── Cloud SQL instances
│   └── AlloyDB clusters
│
├── app-orders-prod/            # App-specific resources
│   └── (Cloud SQL in platform-db-prod,
│        accessed via Private Service Connect)
│
└── platform-network-prod/      # Networking (Shared VPC host)
    └── Private Service Connect endpoints
```

**Why shared DB project?**
- Centralized DBA administration
- Consistent backup policies
- Easier monitoring/alerting

---

# Cloud SQL Architecture

## Regional HA Configuration

```
┌─────────────────────────────────────────────────────────────────┐
│ Region: australia-southeast1                                     │
│                                                                 │
│  ┌───────────────────────────┐  ┌───────────────────────────┐  │
│  │ Zone A (primary)          │  │ Zone B (standby)          │  │
│  │  ┌─────────────────────┐  │  │  ┌─────────────────────┐  │  │
│  │  │ Cloud SQL Instance  │  │  │  │ Cloud SQL Replica   │  │  │
│  │  │ (read-write)        │◄─┼──┼─►│ (sync replication)  │  │  │
│  │  └─────────────────────┘  │  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  └───────────────────────────┘  │
│                                                                 │
│  Automatic failover: ~60 seconds                                │
└─────────────────────────────────────────────────────────────────┘
```

**HA costs ~2x** but provides automatic failover

---

# Cloud SQL Sizing

## Instance Types

| Tier | vCPUs | RAM | Use Case |
|------|-------|-----|----------|
| db-f1-micro | Shared | 0.6 GB | Dev only |
| db-n1-standard-4 | 4 | 15 GB | Small prod |
| db-n1-standard-16 | 16 | 60 GB | Medium prod |
| db-n1-highmem-32 | 32 | 208 GB | Large prod |

**Storage:** SSD (recommended) or HDD, auto-grow enabled

```bash
gcloud sql instances create orders-prod \
  --database-version=POSTGRES_15 \
  --tier=db-n1-standard-8 \
  --region=australia-southeast1 \
  --availability-type=REGIONAL \
  --storage-type=SSD \
  --storage-auto-increase
```

---

# Cloud SQL Networking

## Private IP (Recommended)

```
┌─────────────────────────────────────────────────────────────────┐
│ Shared VPC                                                       │
│                                                                 │
│  ┌─────────────┐        Private Service          ┌───────────┐ │
│  │ GKE Pod     │        Connection               │ Cloud SQL │ │
│  │ or GCE VM   │◄───────────────────────────────►│ (private) │ │
│  │             │        10.2.100.5               │           │ │
│  └─────────────┘                                 └───────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Allocate IP range for private services
gcloud compute addresses create google-managed-services \
  --global --purpose=VPC_PEERING --prefix-length=16 \
  --network=shared-vpc

# Create private connection
gcloud services vpc-peerings connect \
  --service=servicenetworking.googleapis.com \
  --network=shared-vpc \
  --ranges=google-managed-services
```

---

# Cloud SQL IAM Authentication

## No Passwords, Just IAM

```
┌────────────────────────────────────────────────────────────┐
│ GKE Pod with Workload Identity                              │
│   K8s SA: orders-sa                                        │
│      │                                                     │
│      └──► GCP SA: orders-app@project.iam                   │
│               │                                            │
│               └──► IAM Role: roles/cloudsql.client         │
│                       │                                    │
│                       └──► Cloud SQL (no password!)        │
└────────────────────────────────────────────────────────────┘
```

```bash
# Grant IAM database access
gcloud projects add-iam-policy-binding platform-db-prod \
  --member="serviceAccount:orders-app@app-orders-prod.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"
```

---

# AlloyDB Architecture

## PostgreSQL-Compatible, Enterprise Performance

```
┌─────────────────────────────────────────────────────────────────┐
│ AlloyDB Cluster                                                  │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐    │
│  │               Intelligent Storage Layer                 │    │
│  │          (Distributed, Auto-scaling, 99.99% SLA)       │    │
│  └────────────────────────────────────────────────────────┘    │
│         ▲                    ▲                    ▲             │
│         │                    │                    │             │
│  ┌──────┴─────┐       ┌──────┴─────┐       ┌──────┴─────┐      │
│  │  Primary   │       │  Read Pool │       │  Read Pool │      │
│  │  Instance  │       │  Instance 1│       │  Instance 2│      │
│  │ (R/W)      │       │  (RO)      │       │  (RO)      │      │
│  └────────────┘       └────────────┘       └────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

**4x faster** than standard Cloud SQL PostgreSQL for analytics

---

# AlloyDB vs Cloud SQL

| Feature | Cloud SQL | AlloyDB |
|---------|-----------|---------|
| **Compatibility** | MySQL, PG, SQL Server | PostgreSQL only |
| **Performance** | Standard | 4x faster (analytical) |
| **Read replicas** | Manual | Auto-scaling read pool |
| **Storage** | Attached disk | Distributed (like Spanner) |
| **Columnar engine** | No | Yes (analytics) |
| **Price** | $ | $$ |
| **Best for** | General OLTP | High-perf OLTP+OLAP |

## 💬 When to Choose AlloyDB
- Heavy read workloads (dashboards, reporting)
- Mixed OLTP/OLAP
- PostgreSQL apps needing scale

---

# AlloyDB Multi-Tenant

## Same Pattern, Different Config

```yaml
# Cluster per environment (recommended for prod)
alloydb-cluster-prod:
  primary: 8 vCPU, 64 GB
  read-pool: 2-8 instances (auto-scale)
  databases:
    - orders_db
    - inventory_db

alloydb-cluster-nonprod:
  primary: 4 vCPU, 32 GB
  read-pool: 1-2 instances
  databases:
    - orders_db
    - inventory_db
    - (shared for dev/staging)
```

**Read pool per database?** No — read pool serves entire cluster

---

# Self-Managed DBs on VMs

## When You Need Full Control

| Use Case | Why VMs? |
|----------|----------|
| BYOL licensing | SQL Server Enterprise, Oracle |
| Specific version | Cloud SQL doesn't support your version |
| Custom extensions | PostGIS, TimescaleDB with custom config |
| Regulatory | Specific compliance requirements |

```
┌──────────────────────────────────────────────────────────────┐
│ GCE Managed Instance Group                                    │
│                                                              │
│  ┌─────────────────┐      ┌─────────────────┐               │
│  │ db-primary      │      │ db-replica      │               │
│  │ (n2-highmem-16) │◄────►│ (n2-highmem-16) │               │
│  │ + local SSD     │      │ + local SSD     │               │
│  └─────────────────┘      └─────────────────┘               │
│                                                              │
│  You manage: Backups, HA, patching, replication             │
└──────────────────────────────────────────────────────────────┘
```

---

# VM Database Storage Options

| Storage Type | IOPS | Latency | Cost | Best For |
|--------------|------|---------|------|----------|
| **pd-standard** | 3 IOPS/GB | ~5ms | $ | Cold data |
| **pd-balanced** | 6 IOPS/GB | ~1ms | $$ | General |
| **pd-ssd** | 30 IOPS/GB | <1ms | $$$ | OLTP |
| **pd-extreme** | 120k+ IOPS | <1ms | $$$$ | Extreme IOPS |
| **Local SSD** | 680k IOPS | <0.1ms | $$ | Ephemeral, max perf |

```bash
gcloud compute instances create db-primary \
  --machine-type=n2-highmem-16 \
  --create-disk=size=500GB,type=pd-ssd,auto-delete=no \
  --local-ssd=interface=NVME \
  --local-ssd=interface=NVME
```

---

# VM Database HA Options

## DIY Replication Patterns

```
Option A: Streaming Replication (PG/MySQL)
┌─────────────────┐        ┌─────────────────┐
│ Primary         │───────►│ Standby         │
│ (Zone A)        │  async │ (Zone B)        │
└─────────────────┘   or   └─────────────────┘
                    sync

Option B: Shared Storage (DRBD + Pacemaker)
┌─────────────────┐        ┌─────────────────┐
│ Active          │◄──────►│ Passive         │
│ (Zone A)        │  DRBD  │ (Zone B)        │
└────────┬────────┘        └────────┬────────┘
         │                          │
         ▼                          ▼
   ┌──────────────────────────────────────┐
   │        Regional PD (shared)          │
   └──────────────────────────────────────┘
```

---

# Database Networking Summary

## Firewall Rules for DBs

```bash
# Cloud SQL / AlloyDB (via Private Service Connect)
# - No firewall rules needed (VPC peering handles it)

# Self-managed VM DBs
gcloud compute firewall-rules create allow-app-to-db \
  --network=shared-vpc \
  --allow=tcp:5432,tcp:3306 \
  --source-tags=app-vm,gke-node \
  --target-tags=db-vm

# Cross-cloud (Azure to GCP DB during migration)
gcloud compute firewall-rules create allow-azure-to-db \
  --network=shared-vpc \
  --allow=tcp:5432,tcp:3306 \
  --source-ranges=10.1.0.0/16 \
  --target-tags=db-vm
```

---

# Database Cutover Strategy

## Migration Path Options

| Source | Target | Method | Downtime |
|--------|--------|--------|----------|
| Azure SQL | Cloud SQL | DMS | Minutes |
| Azure PostgreSQL | Cloud SQL PG | DMS / pglogical | Minutes |
| Azure PostgreSQL | AlloyDB | DMS | Minutes |
| VM → Cloud SQL | Any | DMS | Minutes |
| Any → VM | N/A | Dump/restore or replication | Varies |

```bash
# Database Migration Service
gcloud database-migration migration-jobs create azure-to-cloudsql \
  --region=australia-southeast1 \
  --source=azure-postgres-conn \
  --destination=cloudsql-postgres \
  --type=CONTINUOUS
```

---

# Database Multi-Tenancy Decision

## Summary Recommendations

| Environment | Cloud SQL | AlloyDB | VM DBs |
|-------------|-----------|---------|--------|
| **Prod** | 1 instance per app (or app-group) | 1 cluster per app-group | 1 VM group per app |
| **Non-prod** | Shared instance, multiple DBs | Shared cluster | Shared VMs |
| **Isolation** | Instance-level | Cluster-level | VM-level |

**Cost optimization:**
- Non-prod: Smaller instances, shared where possible
- Prod: Right-sized, separated for blast radius
- Use Cloud SQL Insights for query-level cost analysis

---

# Summary & Recommendations

| Topic | Recommendation |
|-------|----------------|
| **Cross-Cloud** | HA VPN (start), Interconnect if >3 Gbps needed |
| **DNS** | Cloud DNS private zones + conditional forwarding |
| **Firewall** | Explicit allow, deny-all default |
| **Cutover** | Strangler Fig with weighted traffic routing |
| **LDAP** | GCDS for SSO + Proxy for app auth |
| **VM Networking** | Shared VPC, tiered subnets, Cloud NAT |
| **Cloud SQL** | Separate prod instances, shared non-prod |
| **AlloyDB** | Use for high-perf PG workloads |
| **DB on VMs** | Only for BYOL or specific requirements |

---

# Discussion Points

1. **Connectivity:** VPN vs Interconnect based on your traffic patterns?
2. **Cutover window:** What's acceptable downtime for database migration?
3. **LDAP scope:** SSO only, or application authentication too?
4. **Traffic routing:** DNS-based or Global LB for canary?
5. **Rollback SLA:** How quickly do you need to fail back to Azure?

---

# Next Steps

1. 📋 **Document current state** — Network topology, app inventory
2. 🔌 **Establish VPN connectivity** — Test throughput and latency
3. 🧪 **Pilot migration** — Single low-risk app end-to-end
4. 📊 **Define SLOs** — Error rates, latency thresholds for cutover
5. 📅 **Cutover schedule** — App-by-app timeline with stakeholders

---

# Questions?

📧 Contact: [your-email]
📚 Resources:
- [GKE Best Practices](https://cloud.google.com/kubernetes-engine/docs/best-practices)
- [Cloud VPN](https://cloud.google.com/network-connectivity/docs/vpn)
- [Cloud Directory Sync](https://support.google.com/a/answer/106368)
- [Database Migration Service](https://cloud.google.com/database-migration)
