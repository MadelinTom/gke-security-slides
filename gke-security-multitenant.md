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

<!-- _class: lead -->

# Part 7
## VM Migration Strategy
### Migrate for Compute Engine (M4CE)

---

# VM Migration Approaches

| Approach | Downtime | Complexity | Best For |
|----------|----------|------------|----------|
| **Lift & Shift (M4CE)** | Minutes | Low | Most VMs |
| **Cold Migration** | Hours | Low | Dev/Test |
| **Replatform** | Days | Medium | Optimization |
| **Rebuild** | Weeks | High | Modernization |

## Decision Framework
```
Is the app containerizable?
├─ Yes → GKE migration (Part 3)
└─ No → Continue with VM migration
         ├─ Tight cutover window? → M4CE (warm)
         └─ Flexible timeline? → Cold or replatform
```

---

# Migrate for Compute Engine (M4CE)

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        GCP (Target)                                      │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │              Migrate for Compute Engine                          │  │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │  │
│  │   │ Migrate     │    │ Cloud       │    │ Target      │         │  │
│  │   │ Connector   │◄──►│ Extensions  │───►│ GCE VMs     │         │  │
│  │   │ (on Azure)  │    │             │    │             │         │  │
│  │   └─────────────┘    └─────────────┘    └─────────────┘         │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                    ▲
                    │ Replication
                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Azure (Source)                                    │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │ Source VMs                                                       │  │
│   └─────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# M4CE Migration Phases

## 5-Phase Migration Lifecycle

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ ASSESS  │───►│ PREPARE │───►│ MIGRATE │───►│ CUTOVER │───►│ OPTIMIZE│
└─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘
     │              │              │              │              │
     ▼              ▼              ▼              ▼              ▼
 Inventory     Set up M4CE    Continuous     Switch        Right-size
 Dependency    VPN/Peering    Replication    Traffic       Modernize
 Fit analysis  Test clone     Validation     Retire src    Cost opt
```

**Timeline per VM:**
- Assess: 1-2 days
- Prepare: 1-2 days  
- Migrate (replication): 1-7 days (depends on data size)
- Cutover: 15-60 minutes
- Optimize: Ongoing

---

# Phase 1: Assessment

## Pre-Migration Discovery

```bash
# Install assessment tools on source VMs
# Azure: Use M4CE assessment tool or manual inventory

# Key data to collect:
┌──────────────────────────────────────────────────────────────┐
│ VM Inventory                                                  │
├──────────────────────────────────────────────────────────────┤
│ • OS version and patch level                                 │
│ • CPU/RAM/Disk utilization (30-day average)                  │
│ • Disk count, size, and IOPS requirements                    │
│ • Network interfaces and IPs                                 │
│ • Installed software and licenses                            │
│ • Dependencies (DB connections, API calls)                   │
│ • Backup/DR requirements                                     │
│ • Compliance requirements (PCI, HIPAA, etc.)                 │
└──────────────────────────────────────────────────────────────┘
```

---

# Assessment: Compatibility Matrix

## Supported Source Platforms

| Source | OS | M4CE Support |
|--------|-----|--------------|
| Azure | Windows Server 2012 R2+ | ✅ |
| Azure | Windows Server 2008 R2 | ⚠️ Limited |
| Azure | Ubuntu 16.04+ | ✅ |
| Azure | RHEL/CentOS 7+ | ✅ |
| Azure | Debian 9+ | ✅ |
| Azure | SUSE 12+ | ✅ |

## Blockers to Identify
- [ ] 32-bit OS (not supported)
- [ ] Physical appliances  
- [ ] Encrypted disks (need keys)
- [ ] GPU workloads (special handling)
- [ ] Clustered apps (migrate together)

---

# Assessment: Dependency Mapping

## Critical for Wave Planning

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Application: Order Processing                                            │
│                                                                         │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐        │
│  │ Web VM  │─────►│ App VM  │─────►│ DB VM   │─────►│ Storage │        │
│  │         │      │         │      │         │      │ Account │        │
│  └─────────┘      └─────────┘      └─────────┘      └─────────┘        │
│       │                │                │                               │
│       │                │                └──────► LDAP (on-prem)        │
│       │                └──────────────────────► API Gateway            │
│       └───────────────────────────────────────► CDN                    │
│                                                                         │
│  Migration Unit: Web + App + DB (migrate together)                      │
└─────────────────────────────────────────────────────────────────────────┘
```

**Tool:** Use network flow logs or APM to discover dependencies

---

# Phase 2: Preparation

## Infrastructure Setup Checklist

| Task | Owner | Status |
|------|-------|--------|
| VPN/Interconnect established | Network | ☐ |
| Target VPC and subnets created | Network | ☐ |
| Firewall rules configured | Security | ☐ |
| M4CE deployed and configured | Migration | ☐ |
| Service accounts and IAM set up | Security | ☐ |
| Target machine types selected | Migration | ☐ |
| Disk types and sizes planned | Migration | ☐ |
| DNS strategy documented | Network | ☐ |
| Monitoring/alerting ready | Ops | ☐ |
| Rollback procedure documented | Migration | ☐ |

---

# M4CE Setup

## Deploy Migrate Connector

```bash
# 1. Create service account for M4CE
gcloud iam service-accounts create m4ce-connector \
  --display-name="M4CE Connector"

# 2. Grant required roles
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:m4ce-connector@PROJECT.iam.gserviceaccount.com" \
  --role="roles/vmmigration.admin"

# 3. Deploy connector in Azure (via Azure Marketplace or OVA)
# Configure with:
#   - GCP project ID
#   - Service account key
#   - VPN endpoint IPs
```

---

# Phase 3: Migration (Replication)

## Continuous Data Replication

```
┌────────────────────────────────────────────────────────────────────────┐
│ Timeline                                                                │
│                                                                        │
│  Day 1         Day 2         Day 3         Day 4         Day 5        │
│    │             │             │             │             │           │
│    ▼             ▼             ▼             ▼             ▼           │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░░│  │
│  │ Initial Sync (500 GB)                      │ Delta Sync        │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  Initial sync: ~100 GB/day over VPN (depends on bandwidth)            │
│  Delta sync: Only changed blocks (minimal bandwidth)                   │
└────────────────────────────────────────────────────────────────────────┘
```

**Best Practice:** Start replication 5-7 days before cutover

---

# Replication Monitoring

## Track Sync Progress

```bash
# Check replication status via API or Console
gcloud alpha migration vms replicating-vms describe VM_NAME \
  --region=australia-southeast1

# Key metrics to monitor:
┌──────────────────────────────────────────────────────────┐
│ Metric                      │ Healthy          │ Alert   │
├─────────────────────────────┼──────────────────┼─────────┤
│ Replication lag             │ < 1 hour         │ > 4 hrs │
│ Data transferred (GB/day)   │ On track         │ Behind  │
│ Replication errors          │ 0                │ > 0     │
│ Network throughput          │ Stable           │ Drops   │
└──────────────────────────────────────────────────────────┘
```

**Alert Setup:** Cloud Monitoring alerts for replication lag

---

# Test Clone (Pre-Cutover Validation)

## Validate Before Cutover

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Production (Azure)                 │ Test Clone (GCP)                   │
│                                    │                                    │
│  ┌─────────────┐                   │  ┌─────────────┐                   │
│  │ Source VM   │ ──────────────────┼─►│ Clone VM    │ (isolated VPC)   │
│  │ (running)   │    Snapshot       │  │ (test)      │                   │
│  └─────────────┘                   │  └─────────────┘                   │
│                                    │        │                           │
│                                    │        ▼                           │
│                                    │  Run validation tests              │
│                                    │  • App starts successfully         │
│                                    │  • Can connect to dependencies     │
│                                    │  • Performance baseline            │
│                                    │  • Delete after testing            │
└─────────────────────────────────────────────────────────────────────────┘
```

**Best Practice:** Create test clone 48-72 hours before cutover

---

# Phase 4: Cutover Execution

## Cutover Window Timeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Cutover Window: 60 minutes                                               │
├─────────────────────────────────────────────────────────────────────────┤
│ T-60   │ Pre-checks: Replication healthy, team ready                    │
│ T-30   │ Notify stakeholders, war room active                           │
│ T-15   │ Final delta sync in progress                                   │
│ T-0    │ STOP source VM                                                 │
│ T+5    │ Complete final sync (< 5 min for typical delta)                │
│ T+10   │ Start target VM in GCP                                         │
│ T+15   │ Validate: App health checks, connectivity                      │
│ T+20   │ Update DNS/routing to point to GCP                             │
│ T+30   │ Smoke tests: End-to-end transaction validation                 │
│ T+45   │ Monitor: Error rates, latency, user feedback                   │
│ T+60   │ Declare cutover SUCCESS or initiate ROLLBACK                   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Cutover Runbook Template

## Step-by-Step Execution

| Step | Action | Owner | Duration | Rollback |
|------|--------|-------|----------|----------|
| 1 | Announce maintenance window | PM | - | - |
| 2 | Verify replication lag < 15 min | Eng | 5 min | Delay |
| 3 | Stop source VM | Eng | 2 min | Restart VM |
| 4 | Wait for final sync | Eng | 5-10 min | - |
| 5 | Detach and finalize migration | Eng | 5 min | Abort |
| 6 | Start GCP VM | Eng | 2 min | Stop, restart source |
| 7 | Validate app health | Eng | 10 min | Rollback |
| 8 | Update DNS/LB | Eng | 5 min | Revert DNS |
| 9 | Run smoke tests | QA | 10 min | Rollback |
| 10 | Confirm success | PM | - | - |

---

# Cutover: DNS Strategy

## Options for Traffic Switchover

```
Option A: DNS Update (Simple)
┌─────────────┐         ┌─────────────┐
│ app.example │ ──TTL──►│ New IP      │
│ (update A)  │  (wait) │ (GCP VM)    │
└─────────────┘         └─────────────┘
Downtime: DNS TTL (set to 60s before cutover)

Option B: Load Balancer (Zero-downtime)
┌─────────────┐    ┌─────────────────────────────────┐
│ app.example │───►│ Global LB                       │
│ (no change) │    │  ├─ Azure backend (weight: 0)   │
└─────────────┘    │  └─ GCP backend (weight: 100)   │
                   └─────────────────────────────────┘
Downtime: Near-zero (health check failover)
```

**Best Practice:** Lower DNS TTL to 60-300s one week before cutover

---

# Rollback Procedure

## If Cutover Fails

```
┌─────────────────────────────────────────────────────────────────────────┐
│ ROLLBACK DECISION TREE                                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Cutover complete ──► App healthy? ──► Yes ──► SUCCESS ✓                │
│                             │                                           │
│                             ▼ No                                        │
│                       Fixable in 15 min?                                │
│                             │                                           │
│              ┌──────────────┴──────────────┐                            │
│              ▼ Yes                         ▼ No                         │
│         Fix and retry                  ROLLBACK                         │
│                                            │                            │
│                                            ▼                            │
│                              ┌──────────────────────────┐               │
│                              │ 1. Revert DNS/LB         │               │
│                              │ 2. Stop GCP VM           │               │
│                              │ 3. Restart Azure VM      │               │
│                              │ 4. Verify app healthy    │               │
│                              │ 5. Post-mortem           │               │
│                              └──────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Rollback: Key Steps

## Rapid Recovery Procedure

```bash
# 1. Revert DNS (if DNS-based cutover)
gcloud dns record-sets update app.example.com. \
  --zone=example-zone \
  --type=A \
  --rrdatas="OLD_AZURE_IP" \
  --ttl=60

# 2. Or revert Load Balancer weights
gcloud compute backend-services update app-backend \
  --global \
  --update-backend-group=azure-neg,weight=100 \
  --update-backend-group=gcp-mig,weight=0

# 3. Restart source VM (if stopped)
az vm start --name app-vm --resource-group rg-prod

# 4. Verify Azure app is responding
curl -I https://app.example.com/health
```

**Keep Azure running:** 48-72 hours post-cutover minimum

---

# Phase 5: Post-Migration

## Optimization Checklist

| Task | Timeline | Owner |
|------|----------|-------|
| Delete source VM (after validation period) | +7 days | Eng |
| Right-size GCP VM based on metrics | +14 days | Eng |
| Implement committed use discounts | +30 days | FinOps |
| Optimize disk types (pd-balanced → pd-ssd?) | +14 days | Eng |
| Set up Cloud Monitoring dashboards | +1 day | Ops |
| Configure backup policies | +1 day | Ops |
| Update CMDB/inventory | +1 day | Ops |
| Document lessons learned | +7 days | PM |

---

# VM Migration Wave Planning

## Group VMs for Coordinated Cutover

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Wave Planning Matrix                                                     │
├─────────────────────────────────────────────────────────────────────────┤
│ Wave 1: Low Risk (Pilot)           │ Wave 2: Medium Risk               │
│  • Dev/Test environments            │  • Non-critical production        │
│  • Standalone apps                  │  • Apps with loose dependencies   │
│  • 2-3 VMs max                      │  • 5-10 VMs                        │
│  • Timeline: Week 1-2               │  • Timeline: Week 3-4             │
├─────────────────────────────────────┼───────────────────────────────────┤
│ Wave 3: Higher Risk                 │ Wave 4: Critical Systems          │
│  • Production workloads             │  • Core business apps             │
│  • Apps with dependencies           │  • High-availability required     │
│  • 10-20 VMs                        │  • Tight cutover windows          │
│  • Timeline: Week 5-6               │  • Timeline: Week 7-8             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Wave Dependencies

## Migrate in Correct Order

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   Wave 1                Wave 2                Wave 3                    │
│   ┌─────┐              ┌─────┐              ┌─────┐                    │
│   │ DNS │──────────────│ DB  │──────────────│ App │                    │
│   │     │              │     │              │     │                    │
│   └─────┘              └─────┘              └─────┘                    │
│      │                    │                    │                        │
│      │    Must complete   │    Must complete   │                        │
│      │    before Wave 2   │    before Wave 3   │                        │
│      ▼                    ▼                    ▼                        │
│   Shared services      Databases           Applications                │
│   first!               before apps!        last!                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Common order:** Infra → DBs → Backend → Frontend

---

<!-- _class: lead -->

# Part 8
## Database Migration Strategy
### Cloud SQL, AlloyDB, and VM Databases

---

# Database Migration Patterns

| Pattern | Downtime | Complexity | Data Loss Risk |
|---------|----------|------------|----------------|
| **DMS Continuous** | Minutes | Low | Very Low |
| **Native Replication** | Minutes | Medium | Low |
| **Dump & Restore** | Hours | Low | Medium |
| **Dual-Write** | Zero | High | Medium |
| **Blue-Green DB** | Seconds | High | Low |

## Decision Tree
```
Can you tolerate any downtime?
├─ Yes (minutes OK) → DMS or Native Replication
└─ No (zero downtime) → Dual-Write or Blue-Green
         └─ Complex, error-prone — avoid if possible
```

---

# Database Migration Service (DMS)

## GCP's Managed Migration Tool

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Database Migration Service Architecture                                  │
│                                                                         │
│  ┌─────────────────┐                        ┌─────────────────┐        │
│  │ Source DB       │                        │ Target DB       │        │
│  │ (Azure SQL/PG)  │                        │ (Cloud SQL)     │        │
│  └────────┬────────┘                        └────────▲────────┘        │
│           │                                          │                  │
│           │  ┌─────────────────────────────────────┐ │                  │
│           └─►│ DMS Migration Job                   │─┘                  │
│              │  • Initial full load                │                    │
│              │  • Continuous CDC replication       │                    │
│              │  • Schema conversion                │                    │
│              └─────────────────────────────────────┘                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# DMS Supported Sources

## Migration Path Matrix

| Source | Target | Method | Notes |
|--------|--------|--------|-------|
| Azure SQL | Cloud SQL (SQL Server) | DMS | Full support |
| Azure PostgreSQL | Cloud SQL (PostgreSQL) | DMS | Full support |
| Azure PostgreSQL | AlloyDB | DMS | Full support |
| Azure MySQL | Cloud SQL (MySQL) | DMS | Full support |
| SQL Server on VM | Cloud SQL | DMS | Requires agent |
| PostgreSQL on VM | Cloud SQL/AlloyDB | DMS | Requires agent |
| Oracle | Cloud SQL (PostgreSQL) | Ora2Pg + DMS | Schema conversion |
| Oracle | Bare Metal | Oracle tools | Lift & shift |

---

# DMS Setup: Step by Step

## 1. Create Connection Profiles

```bash
# Source connection profile (Azure PostgreSQL)
gcloud database-migration connection-profiles create azure-postgres \
  --region=australia-southeast1 \
  --display-name="Azure PostgreSQL Source" \
  --postgresql-host=myserver.postgres.database.azure.com \
  --postgresql-port=5432 \
  --postgresql-username=admin@myserver \
  --postgresql-password-file=./password.txt \
  --postgresql-ssl-config-certificate-file=./azure-ca.pem

# Target connection profile (Cloud SQL)
gcloud database-migration connection-profiles create cloudsql-target \
  --region=australia-southeast1 \
  --display-name="Cloud SQL Target" \
  --cloudsql-instance=projects/PROJECT/instances/INSTANCE
```

---

# DMS Setup: Migration Job

## 2. Create and Start Migration

```bash
# Create migration job
gcloud database-migration migration-jobs create azure-to-gcp \
  --region=australia-southeast1 \
  --display-name="Azure PG to Cloud SQL" \
  --source=azure-postgres \
  --destination=cloudsql-target \
  --type=CONTINUOUS \
  --dump-parallel-level=MAX

# Start the migration (begins initial full load)
gcloud database-migration migration-jobs start azure-to-gcp \
  --region=australia-southeast1

# Monitor progress
gcloud database-migration migration-jobs describe azure-to-gcp \
  --region=australia-southeast1 \
  --format="table(name,state,phase,error)"
```

---

# DMS Migration Phases

## Timeline Visualization

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Migration Timeline                                                       │
│                                                                         │
│  ┌─────────────┐ ┌─────────────────────────────┐ ┌─────────┐ ┌───────┐ │
│  │ Full Dump   │ │ CDC Replication             │ │ Promote │ │ Done  │ │
│  │ (hours-days)│ │ (continuous, lag < 1 min)   │ │ (mins)  │ │       │ │
│  └─────────────┘ └─────────────────────────────┘ └─────────┘ └───────┘ │
│                                                                         │
│  Phase 1: FULL_DUMP        Phase 2: CDC           Phase 3: PROMOTE     │
│  • Schema creation         • Real-time changes    • Stop source writes │
│  • Initial data load       • Low latency sync     • Final sync         │
│  • Can take hours          • Monitor lag          • Promote replica    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Data Validation

## Verify Data Integrity

```bash
# Built-in DMS validation
gcloud database-migration migration-jobs verify azure-to-gcp \
  --region=australia-southeast1

# Manual validation queries
# Run on BOTH source and target, compare results:

# Row counts per table
SELECT table_name, 
       (xpath('/row/count/text()', 
        query_to_xml('SELECT COUNT(*) FROM '||table_name, false, true, '')))[1]::text::int AS row_count
FROM information_schema.tables 
WHERE table_schema = 'public';

# Checksum critical tables
SELECT md5(array_agg(t.*)::text) AS checksum 
FROM (SELECT * FROM orders ORDER BY id) t;
```

---

# Validation Checklist

## Pre-Cutover Data Verification

| Check | Method | Pass Criteria |
|-------|--------|---------------|
| Row counts match | SQL query | 100% match |
| Schema identical | pg_dump --schema-only | Diff = 0 |
| Primary keys | Query | All present |
| Foreign keys | Query | All valid |
| Indexes | Query | All created |
| Sequences | Query | Current values match |
| Triggers/Functions | Query | All migrated |
| Permissions | Query | Roles configured |
| Sample data spot-check | Manual | Correct |

---

# Database Cutover Procedure

## Promotion Runbook

```
┌─────────────────────────────────────────────────────────────────────────┐
│ DATABASE CUTOVER TIMELINE (30-45 minutes)                                │
├─────────────────────────────────────────────────────────────────────────┤
│ T-30  │ Verify replication lag < 1 minute                               │
│ T-20  │ Notify stakeholders, freeze deployments                         │
│ T-10  │ Run final validation checks                                     │
│ T-5   │ Prepare connection string updates                               │
│ T-0   │ STOP application writes to source DB                            │
│ T+2   │ Wait for replication to catch up (lag = 0)                      │
│ T+5   │ Promote DMS migration (makes target primary)                    │
│ T+10  │ Update application connection strings                           │
│ T+15  │ Restart applications with new connection                        │
│ T+20  │ Smoke test: Verify reads AND writes work                        │
│ T+30  │ Monitor error rates and query performance                       │
│ T+45  │ Declare SUCCESS or ROLLBACK                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Connection String Cutover

## Options for Switching Applications

```yaml
# Option A: Direct Update (requires app restart)
# Before:
DATABASE_URL: "postgresql://user:pass@azure-server.postgres.database.azure.com/orders"
# After:
DATABASE_URL: "postgresql://user:pass@10.2.100.5/orders"  # Cloud SQL private IP

# Option B: DNS CNAME (no app restart needed)
# db.internal.example.com → azure-server.postgres.database.azure.com
# Cutover: Update CNAME to Cloud SQL private IP
# db.internal.example.com → 10.2.100.5

# Option C: Connection Proxy (Cloud SQL Auth Proxy)
# App connects to localhost:5432 → Proxy → Cloud SQL
# Cutover: Proxy already points to Cloud SQL
```

**Recommendation:** Use internal DNS CNAMEs for zero-downtime switchover

---

# Database Rollback Strategy

## If Cutover Fails

```
┌─────────────────────────────────────────────────────────────────────────┐
│ ROLLBACK DECISION                                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ⚠️  CRITICAL: Once apps write to new DB, rollback is COMPLEX           │
│                                                                         │
│  Scenario 1: Cutover failed BEFORE any writes to new DB                 │
│  ────────────────────────────────────────────────────────────────────── │
│  → Simple: Revert connection strings, restart apps                      │
│  → Source DB unchanged, no data loss                                    │
│                                                                         │
│  Scenario 2: Cutover failed AFTER writes to new DB                      │
│  ────────────────────────────────────────────────────────────────────── │
│  → Complex: Must migrate delta data back to source                      │
│  → Options:                                                             │
│    a) Forward-fix in GCP (preferred)                                    │
│    b) Set up reverse replication (DMS target→source)                    │
│    c) Accept brief data loss, restore source from backup                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Minimizing Rollback Risk

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Risk Mitigation Strategies                                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│ 1. READ-ONLY PERIOD                                                     │
│    • Cut over reads first (app reads from GCP)                          │
│    • Keep writes on source for 24-48 hours                              │
│    • Then cut over writes when confident                                │
│                                                                         │
│ 2. FEATURE FLAGS                                                        │
│    • Use feature flags to control DB routing                            │
│    • Instant rollback without deployment                                │
│                                                                         │
│ 3. CANARY TRAFFIC                                                       │
│    • Route 5% of connections to new DB                                  │
│    • Monitor for errors before full cutover                             │
│                                                                         │
│ 4. BACKUP BEFORE PROMOTE                                                │
│    • Take final backup of source DB                                     │
│    • Point-in-time recovery option if needed                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Native Replication Alternative

## PostgreSQL: pglogical / Logical Replication

```bash
# Source (Azure PostgreSQL) - Enable logical replication
# Requires: azure.replication_support = logical

# On source:
CREATE PUBLICATION orders_pub FOR TABLE orders, customers, products;

# On target (Cloud SQL):
CREATE SUBSCRIPTION orders_sub
  CONNECTION 'host=azure-server.postgres.database.azure.com 
              port=5432 
              dbname=orders 
              user=replication_user'
  PUBLICATION orders_pub;

# Monitor replication lag
SELECT 
  slot_name,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
FROM pg_replication_slots;
```

---

# Native Replication: MySQL

## GTID-Based Replication

```sql
-- Source (Azure MySQL) - Enable GTID
-- Requires: gtid_mode = ON, enforce_gtid_consistency = ON

-- On target (Cloud SQL for MySQL):
CALL mysql.setupExternalReplication(
  'azure-server.mysql.database.azure.com',  -- host
  3306,                                       -- port
  'replication_user',                         -- user
  'password',                                 -- password
  '',                                         -- binlog file (empty for GTID)
  0,                                          -- binlog pos
  TRUE                                        -- use GTID
);

-- Start replication
CALL mysql.startReplication();

-- Check status
CALL mysql.showReplicationStatus();
```

---

# AlloyDB Migration

## DMS to AlloyDB

```bash
# Create AlloyDB cluster first
gcloud alloydb clusters create orders-cluster \
  --region=australia-southeast1 \
  --password=SECURE_PASSWORD \
  --network=shared-vpc

# Create primary instance
gcloud alloydb instances create orders-primary \
  --cluster=orders-cluster \
  --region=australia-southeast1 \
  --instance-type=PRIMARY \
  --cpu-count=8

# Create DMS migration job to AlloyDB
gcloud database-migration migration-jobs create azure-to-alloydb \
  --region=australia-southeast1 \
  --source=azure-postgres \
  --destination-alloydb-cluster=orders-cluster \
  --type=CONTINUOUS
```

---

# VM Database Migration

## Self-Managed DB Cutover

```
┌─────────────────────────────────────────────────────────────────────────┐
│ VM-to-VM Database Migration (PostgreSQL example)                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Option A: Streaming Replication                                        │
│  ────────────────────────────────────────────────────────────────────── │
│  1. Set up GCP VM with PostgreSQL                                       │
│  2. Configure as streaming replica of Azure source                      │
│  3. Cutover: Promote GCP replica to primary                             │
│  4. Downtime: ~1-2 minutes                                              │
│                                                                         │
│  Option B: pg_dump / pg_restore                                         │
│  ────────────────────────────────────────────────────────────────────── │
│  1. pg_dump on source (with --format=directory for parallelism)         │
│  2. Transfer dump to GCP (gsutil or direct copy)                        │
│  3. pg_restore on target                                                │
│  4. Downtime: Hours (depends on size)                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Streaming Replication Setup

## Azure VM → GCP VM (PostgreSQL)

```bash
# On Source (Azure VM) - postgresql.conf
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1GB

# On Source - pg_hba.conf
host replication replicator 10.2.0.0/16 scram-sha-256

# On Target (GCP VM) - Initial base backup
pg_basebackup -h azure-db-vm -U replicator \
  -D /var/lib/postgresql/15/main -Fp -Xs -P -R

# On Target - Start PostgreSQL (will follow source)
systemctl start postgresql

# Verify replication
psql -c "SELECT * FROM pg_stat_wal_receiver;"
```

---

# Cutover: Promote Replica

## Streaming Replication Promotion

```bash
# Pre-cutover checklist
psql -c "SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();"
# Ensure both LSNs match (replica is caught up)

# 1. Stop applications writing to source
systemctl stop myapp

# 2. Final sync check (wait for lag = 0)
watch -n 1 "psql -c \"SELECT pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS lag_bytes;\""

# 3. Promote replica to primary
pg_ctl promote -D /var/lib/postgresql/15/main
# Or: SELECT pg_promote();

# 4. Verify it's now primary
psql -c "SELECT pg_is_in_recovery();"  # Should return 'f' (false)

# 5. Update application connection strings to GCP VM
# 6. Start applications
```

---

# Large Database Considerations

## Strategies for Multi-TB Databases

| Strategy | Use Case | Notes |
|----------|----------|-------|
| **Parallel dump/restore** | < 1 TB, can tolerate hours | pg_dump -j 8 |
| **Streaming replication** | Any size, minutes downtime | Recommended |
| **DMS continuous** | Managed DBs, any size | Easiest |
| **Physical backup + ship** | Very large, limited bandwidth | Ship disks |
| **Incremental approach** | > 10 TB | Initial + catchup |

```bash
# Parallel pg_dump for faster export
pg_dump -h source -U admin -d orders \
  --format=directory \
  --jobs=8 \
  --file=/backup/orders_dump

# Parallel pg_restore on target
pg_restore -h target -U admin -d orders \
  --jobs=8 \
  /backup/orders_dump
```

---

# Database Migration Checklist

## Comprehensive Pre-Cutover List

| Category | Check | Status |
|----------|-------|--------|
| **Schema** | All tables migrated | ☐ |
| **Schema** | All indexes created | ☐ |
| **Schema** | All constraints valid | ☐ |
| **Schema** | Sequences at correct values | ☐ |
| **Data** | Row counts match | ☐ |
| **Data** | Checksums match (sample) | ☐ |
| **Replication** | Lag < 1 minute | ☐ |
| **Performance** | Query plans acceptable | ☐ |
| **Connectivity** | Apps can connect | ☐ |
| **Auth** | IAM/passwords configured | ☐ |
| **Backup** | Final source backup taken | ☐ |
| **Rollback** | Procedure documented | ☐ |

---

# Post-Migration Database Tasks

## Don't Forget These!

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Post-Cutover Checklist                                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│ Immediate (Day 1):                                                      │
│ ☐ Update ANALYZE statistics (for query planner)                         │
│ ☐ Verify automated backups configured                                   │
│ ☐ Set up Cloud Monitoring alerts                                        │
│ ☐ Verify point-in-time recovery works                                   │
│                                                                         │
│ Short-term (Week 1):                                                    │
│ ☐ Monitor query performance vs baseline                                 │
│ ☐ Right-size instance based on actual usage                             │
│ ☐ Delete source database (after validation period)                      │
│ ☐ Update documentation and runbooks                                     │
│                                                                         │
│ Long-term (Month 1):                                                    │
│ ☐ Review and optimize slow queries                                      │
│ ☐ Consider read replicas if needed                                      │
│ ☐ Evaluate committed use discounts                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

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
