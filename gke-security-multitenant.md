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

# Summary & Recommendations

| Topic | Recommendation |
|-------|----------------|
| **Cross-Cloud** | HA VPN (start), Interconnect if >3 Gbps needed |
| **DNS** | Cloud DNS private zones + conditional forwarding |
| **Firewall** | Explicit allow, deny-all default |
| **Cutover** | Strangler Fig with weighted traffic routing |
| **LDAP** | GCDS for SSO + Proxy for app auth |

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
