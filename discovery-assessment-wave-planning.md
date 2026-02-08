---
marp: true
theme: default
paginate: true
header: 'Application Discovery & Wave Planning'
footer: 'Cloud Migration Workshop | Azure → GCP'
---

# Application Discovery & Assessment
## Wave Planning for Cloud Migration
## + Workshop Facilitation Guide

*Interactive Workshop: From Inventory to Migration Waves*

---

# Agenda

**Part 1: Discovery**
1. Discovery methodologies & tools
2. Building the application inventory
3. Dependency mapping

**Part 2: Assessment**
4. The 6 Rs framework
5. Complexity & risk scoring
6. Technical fit analysis

**Part 3: Wave Planning**
7. Prioritization frameworks
8. Wave design principles
9. Timeline & resourcing

**Part 4: Workshop Facilitation**
10. Running effective discovery sessions
11. Capture templates & tools

---

<!-- _class: lead -->

# Part 1
## Application Discovery
### Finding What You Have

---

# Why Discovery Matters

## The Migration Iceberg

```
        What stakeholders tell you
             ┌─────────────┐
             │  10 apps    │  ← "We have about 10 apps to migrate"
             └─────────────┘
    ─────────────────────────────── waterline
             ┌─────────────┐
             │  Shadow IT  │
             ├─────────────┤
             │  Legacy     │
             ├─────────────┤
             │  Forgotten  │  ← What you actually find: 50+ apps
             ├─────────────┤
             │  Scripts    │
             ├─────────────┤
             │  Integrations│
             └─────────────┘
```

**Discovery prevents:** Scope creep, surprise dependencies, missed deadlines

---

# Discovery Approaches

| Approach | Pros | Cons | Best For |
|----------|------|------|----------|
| **Stakeholder Interviews** | Context, priorities | Incomplete, biased | Starting point |
| **CMDB/Asset Inventory** | Structured data | Often outdated | Cross-reference |
| **Network Scanning** | Finds unknown | No business context | Shadow IT |
| **APM/Agent-based** | Deep dependencies | Deployment required | Complex apps |
| **Cloud Provider Tools** | Native integration | Vendor-specific | Single-cloud source |

## 💬 Recommendation
Use **multiple approaches** — each catches what others miss

---

# Discovery Tools Landscape

## Azure → GCP Migration

| Tool | Type | What It Finds |
|------|------|---------------|
| **Azure Migrate** | Native | VMs, DBs, web apps, dependencies |
| **Azure Resource Graph** | Query | All Azure resources via KQL |
| **Stratozone** | GCP Native | Fit assessment, TCO |
| **CAST Highlight** | Code analysis | App complexity, cloud blockers |
| **Flexera/Lakeside** | Agent-based | Usage, dependencies |
| **Qualys/Rapid7** | Scanner | Vulnerabilities, inventory |
| **ServiceNow CMDB** | ITSM | Config items, relationships |

---

# Azure Migrate: Discovery

## What It Captures

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Azure Migrate Discovery                                                  │
├─────────────────────────────────────────────────────────────────────────┤
│ VMs:                          │ Dependencies:                           │
│  • OS type & version          │  • Process-to-process connections       │
│  • CPU, RAM, disk             │  • Port-level communication             │
│  • Installed software         │  • Inbound/outbound flows               │
│  • Performance metrics        │  • Visualization map                    │
│                               │                                         │
│ Databases:                    │ Web Apps:                               │
│  • SQL Server instances       │  • App Service inventory                │
│  • DB sizes and configs       │  • Function Apps                        │
│  • Azure SQL/PostgreSQL       │  • Container instances                  │
└─────────────────────────────────────────────────────────────────────────┘
```

**Dependency visualization:** Requires agent installation on VMs

---

# Azure Resource Graph

## Query Your Entire Estate

```kusto
// Get all resources with their types and locations
resources
| summarize count() by type, location
| order by count_ desc

// Find VMs and their sizes
resources
| where type == "microsoft.compute/virtualmachines"
| extend vmSize = properties.hardwareProfile.vmSize
| project name, resourceGroup, location, vmSize

// Find all databases
resources
| where type contains "database" or type contains "sql"
| project name, type, resourceGroup, location
```

**Export to CSV** → Foundation for inventory spreadsheet

---

# Building the Application Inventory

## Required Data Points

| Category | Fields | Source |
|----------|--------|--------|
| **Identity** | App name, ID, owner, team | Interviews, CMDB |
| **Business** | Criticality, revenue impact, SLA | Business owners |
| **Technical** | Tech stack, OS, language, DB | Discovery tools |
| **Infrastructure** | VMs, containers, PaaS services | Azure Migrate |
| **Dependencies** | Upstream, downstream, external | APM, network flows |
| **Data** | Volume, sensitivity, compliance | DBA, InfoSec |
| **Operations** | Backup, DR, monitoring | Ops team |

---

# Application Inventory Template

## Spreadsheet Structure

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ A: App ID        │ H: Dependencies    │ O: Cloud Fit      │ V: Wave        │
│ B: App Name      │ I: Data Class      │ P: Effort Est     │ W: Status      │
│ C: Owner         │ J: Compliance      │ Q: Risk Score     │ X: Notes       │
│ D: Team          │ K: Tech Stack      │ R: 6R Decision    │                │
│ E: Criticality   │ L: Current Infra   │ S: Target State   │                │
│ F: Business Fn   │ M: Performance     │ T: Priority       │                │
│ G: SLA           │ N: Last Updated    │ U: Blockers       │                │
└──────────────────────────────────────────────────────────────────────────────┘
```

**📋 Template available:** [Link to shared template]

---

# Dependency Mapping

## Why It's Critical

```
"We'll just migrate the Order Service this weekend"
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Order Service Dependencies (discovered after planning)                   │
├─────────────────────────────────────────────────────────────────────────┤
│ ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│ │ User Auth   │  │ Inventory   │  │ Payment     │  │ Shipping    │     │
│ │ Service     │  │ Service     │  │ Gateway     │  │ API         │     │
│ └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘     │
│        │                │                │                │            │
│        └────────────────┴────────────────┴────────────────┘            │
│                                   │                                     │
│                          ┌────────▼────────┐                           │
│                          │ Order Service   │ ← Cannot migrate alone!   │
│                          └────────┬────────┘                           │
│                                   │                                     │
│        ┌──────────────────────────┼──────────────────────────┐         │
│        ▼                          ▼                          ▼         │
│ ┌─────────────┐          ┌─────────────┐          ┌─────────────┐     │
│ │ SQL Server  │          │ Redis Cache │          │ Service Bus │     │
│ └─────────────┘          └─────────────┘          └─────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Dependency Types

## Map All Connection Types

| Type | Example | Discovery Method |
|------|---------|------------------|
| **Synchronous** | REST API calls | APM, network flows |
| **Asynchronous** | Message queues, events | Queue metrics, code |
| **Data** | Shared databases | DB connection strings |
| **Authentication** | SSO, LDAP, AD | Auth logs, config |
| **File** | Shared storage, NFS | Mount points, SMB |
| **External** | 3rd party APIs, SaaS | Firewall logs, code |
| **Human** | Manual processes | Interviews |

## 💬 Workshop Question
*"What breaks if this system goes down for 1 hour?"*

---

# Dependency Visualization

## Create Dependency Maps

```
                           ┌─────────────────┐
                           │  External APIs  │
                           └────────┬────────┘
                                    │
    ┌───────────────────────────────┼───────────────────────────────┐
    │                               │                               │
    ▼                               ▼                               ▼
┌─────────┐                   ┌─────────┐                   ┌─────────┐
│ Web App │                   │ Mobile  │                   │ Partner │
│ (React) │                   │ App     │                   │ Portal  │
└────┬────┘                   └────┬────┘                   └────┬────┘
     │                             │                             │
     └──────────────┬──────────────┴──────────────┬──────────────┘
                    │                             │
                    ▼                             ▼
            ┌──────────────┐              ┌──────────────┐
            │ API Gateway  │              │ Auth Service │
            └──────┬───────┘              └──────────────┘
                   │
     ┌─────────────┼─────────────┬─────────────┐
     ▼             ▼             ▼             ▼
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│ Orders  │  │ Products│  │ Users   │  │ Reports │
└────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘
     │            │            │            │
     └────────────┴─────┬──────┴────────────┘
                        ▼
                ┌──────────────┐
                │   Database   │
                └──────────────┘
```

**Tools:** Lucidchart, draw.io, Miro, Azure Migrate visualization

---

<!-- _class: lead -->

# Part 2
## Application Assessment
### Deciding What to Do

---

# The 6 Rs Framework

## Migration Strategies

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          MIGRATION STRATEGIES                            │
├─────────────┬─────────────┬─────────────┬─────────────┬────────────────┤
│   RETAIN    │  RETIRE     │  REHOST     │ REPLATFORM  │   REFACTOR     │
│   (Keep)    │  (Delete)   │  (Lift&Shift)│ (Optimize)  │   (Rebuild)    │
├─────────────┼─────────────┼─────────────┼─────────────┼────────────────┤
│ Stay on-prem│ Decommission│ VM → VM     │ VM → PaaS   │ Rearchitect    │
│ or Azure    │             │ Same config │ Managed svc │ Cloud-native   │
├─────────────┼─────────────┼─────────────┼─────────────┼────────────────┤
│ Low value   │ No business │ Quick win   │ Balance     │ Strategic      │
│ High risk   │ value       │ Low change  │ Cost/effort │ apps           │
├─────────────┼─────────────┼─────────────┼─────────────┼────────────────┤
│ Effort: 0   │ Effort: Low │ Effort: Low │ Effort: Med │ Effort: High   │
│ Risk: 0     │ Risk: Low   │ Risk: Low   │ Risk: Med   │ Risk: High     │
└─────────────┴─────────────┴─────────────┴─────────────┴────────────────┘
                                                        │
                                              ┌─────────┴─────────┐
                                              │    REPURCHASE     │
                                              │    (Replace)      │
                                              ├───────────────────┤
                                              │ Move to SaaS      │
                                              │ Buy vs build      │
                                              ├───────────────────┤
                                              │ Effort: Med       │
                                              │ Risk: Med         │
                                              └───────────────────┘
```

---

# 6 Rs Decision Tree

## Quick Classification

```
┌─────────────────────────────────────────────────────────────────────────┐
│ START: Is this application still needed?                                 │
│                     │                                                    │
│          ┌──────────┴──────────┐                                        │
│          ▼ No                  ▼ Yes                                    │
│    ┌──────────┐        Is it strategic/core business?                   │
│    │ RETIRE   │                │                                        │
│    └──────────┘     ┌──────────┴──────────┐                             │
│                     ▼ No                  ▼ Yes                         │
│              Can we buy SaaS?      Does it need rearchitecting?         │
│                     │                     │                             │
│          ┌─────────┴─────┐     ┌──────────┴──────────┐                  │
│          ▼ Yes           ▼ No  ▼ No                  ▼ Yes              │
│    ┌───────────┐    Is there   │              ┌───────────┐             │
│    │REPURCHASE │    time/budget?              │ REFACTOR  │             │
│    └───────────┘         │     │              └───────────┘             │
│                   ┌──────┴──┐  │                                        │
│                   ▼ No      ▼ Yes                                       │
│             ┌─────────┐  Can we optimize                                │
│             │ RETAIN  │  with managed svc?                              │
│             └─────────┘       │                                         │
│                        ┌──────┴──────┐                                  │
│                        ▼ Yes         ▼ No                               │
│                  ┌───────────┐  ┌─────────┐                             │
│                  │REPLATFORM │  │ REHOST  │                             │
│                  └───────────┘  └─────────┘                             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Rehost: Lift & Shift

## When to Choose

| ✅ Good Fit | ❌ Poor Fit |
|------------|-------------|
| Tight timeline | App needs modernization anyway |
| Stable, well-understood app | Performance-sensitive workload |
| Legacy OS supported | Uses unsupported features |
| Quick win needed | Cost optimization priority |
| Bridge to future refactor | Long-term strategic app |

## Azure → GCP Mapping

| Azure | GCP Rehost Target |
|-------|-------------------|
| Azure VM | Compute Engine |
| Azure SQL VM | Cloud SQL or CE |
| Azure Files | Filestore |
| Azure Blob | Cloud Storage |

---

# Replatform: Optimize

## When to Choose

| ✅ Good Fit | ❌ Poor Fit |
|------------|-------------|
| App can use managed services | Highly customized config |
| Want reduced ops burden | Needs specific OS/version |
| Database migrations | Tightly coupled architecture |
| Moderate timeline | No PaaS equivalent exists |

## Azure → GCP Mapping

| Azure | GCP Replatform Target |
|-------|----------------------|
| Azure SQL | Cloud SQL |
| Azure PostgreSQL | Cloud SQL / AlloyDB |
| App Service | Cloud Run / App Engine |
| AKS | GKE Autopilot |
| Azure Functions | Cloud Functions |

---

# Refactor: Rearchitect

## When to Choose

| ✅ Good Fit | ❌ Poor Fit |
|------------|-------------|
| Strategic, long-lived app | End-of-life application |
| Need cloud-native benefits | Tight timeline |
| Scalability requirements | Limited budget |
| Team has skills/time | Simple workload |

## Common Refactor Patterns

| From | To |
|------|-----|
| Monolith | Microservices on GKE |
| Batch jobs | Event-driven (Pub/Sub + Functions) |
| File-based integration | API-based / streaming |
| Stateful servers | Stateless + managed state |
| Manual scaling | Auto-scaling |

---

# Application Complexity Scoring

## Quantify Migration Difficulty

| Factor | Low (1) | Medium (2) | High (3) |
|--------|---------|------------|----------|
| **Dependencies** | 0-2 | 3-5 | 6+ |
| **Data Volume** | < 100 GB | 100 GB - 1 TB | > 1 TB |
| **Compliance** | None | Standard | PCI/HIPAA |
| **Availability SLA** | < 99% | 99-99.9% | > 99.9% |
| **Tech Stack Age** | < 3 years | 3-7 years | > 7 years |
| **Documentation** | Complete | Partial | None |
| **Team Knowledge** | High | Medium | Low |
| **Custom Code** | Minimal | Moderate | Heavy |

**Score Range:** 8-24 points
- **8-12:** Low complexity (good pilot candidate)
- **13-18:** Medium complexity
- **19-24:** High complexity (plan carefully)

---

# Risk Assessment Matrix

## Evaluate Migration Risk

```
                        BUSINESS IMPACT
                   Low         Medium        High
              ┌───────────┬───────────┬───────────┐
         High │   Medium  │   High    │  Critical │
              │   Risk    │   Risk    │   Risk    │
              ├───────────┼───────────┼───────────┤
COMPLEXITY   Med│   Low     │   Medium  │   High    │
              │   Risk    │   Risk    │   Risk    │
              ├───────────┼───────────┼───────────┤
         Low │   Minimal │   Low     │   Medium  │
              │   Risk    │   Risk    │   Risk    │
              └───────────┴───────────┴───────────┘
```

**Critical Risk:** Needs executive sponsorship, extended testing, detailed rollback
**High Risk:** Dedicated team, thorough planning, staged cutover
**Medium Risk:** Standard process with extra validation
**Low Risk:** Standard migration process

---

# Technical Fit Assessment

## GCP Readiness Checklist

| Category | Check | Pass/Fail |
|----------|-------|-----------|
| **OS** | Supported OS version | ☐ |
| **Licensing** | License portability (BYOL) | ☐ |
| **Network** | No hardcoded IPs | ☐ |
| **Storage** | Compatible storage type | ☐ |
| **Database** | Migration path exists | ☐ |
| **Auth** | IAM/SSO compatible | ☐ |
| **Compliance** | GCP region meets requirements | ☐ |
| **Performance** | Instance type available | ☐ |
| **Dependencies** | All deps can migrate | ☐ |

**Blockers:** Any "Fail" needs remediation plan before migration

---

<!-- _class: lead -->

# Part 3
## Wave Planning
### Sequencing the Migration

---

# Wave Planning Principles

## The Golden Rules

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. START SMALL, LEARN FAST                                               │
│    Wave 1 = 1-2 low-risk apps, prove the process                        │
├─────────────────────────────────────────────────────────────────────────┤
│ 2. DEPENDENCIES DICTATE SEQUENCE                                         │
│    Shared services before consumers, DBs before apps                    │
├─────────────────────────────────────────────────────────────────────────┤
│ 3. GROUP BY AFFINITY                                                     │
│    Apps that share infra/team/domain migrate together                   │
├─────────────────────────────────────────────────────────────────────────┤
│ 4. BALANCE EACH WAVE                                                     │
│    Mix complexity levels, don't stack all hard apps                     │
├─────────────────────────────────────────────────────────────────────────┤
│ 5. RESPECT BUSINESS CYCLES                                               │
│    No migrations during peak season, month-end, etc.                    │
├─────────────────────────────────────────────────────────────────────────┤
│ 6. BUILD IN BUFFER                                                       │
│    Plan for delays — things always take longer                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Wave Sizing

## How Many Apps Per Wave?

| Wave | Apps | Duration | Purpose |
|------|------|----------|---------|
| **Wave 0** | 0 | 2-4 weeks | Foundation (VPN, IAM, networking) |
| **Wave 1** | 1-2 | 2-3 weeks | Pilot — prove the process |
| **Wave 2** | 3-5 | 3-4 weeks | Expand — refine playbooks |
| **Wave 3+** | 5-10 | 4-6 weeks | Scale — parallel migrations |

## Velocity Curve
```
Apps/Wave  │
     10    │                              ████████
      8    │                    ████████████████████
      6    │          ██████████████████████████████
      4    │    ████████████████████████████████████
      2    │████████████████████████████████████████
      0    └────────────────────────────────────────
           Wave 0  Wave 1  Wave 2  Wave 3  Wave 4
```

---

# Wave 0: Foundation

## Before Any Apps Move

| Task | Description | Duration |
|------|-------------|----------|
| **Networking** | VPN/Interconnect, DNS, firewall rules | 1-2 weeks |
| **IAM** | Service accounts, roles, federation | 1 week |
| **Landing Zone** | Projects, folders, policies | 1 week |
| **Monitoring** | Cloud Monitoring, logging, alerting | 3-5 days |
| **Security** | Security Command Center, org policies | 1 week |
| **CI/CD** | Pipeline setup, artifact repos | 1 week |
| **Runbooks** | Migration playbooks, rollback docs | 1 week |

**Overlap possible** — but don't skip this phase!

---

# Wave 1: The Pilot

## Characteristics of a Good Pilot

| ✅ Choose | ❌ Avoid |
|----------|---------|
| Low business criticality | Mission-critical systems |
| Simple architecture | Complex dependencies |
| Representative tech stack | Unique/one-off technology |
| Supportive app team | Resistant stakeholders |
| Clear success metrics | Vague requirements |
| Tolerant of downtime | Zero-downtime requirement |

## Pilot Success Criteria
- [ ] Migration completed within planned window
- [ ] Application functional post-migration
- [ ] Performance within acceptable range
- [ ] Rollback tested and documented
- [ ] Lessons learned captured

---

# Prioritization Framework

## Scoring Model for Wave Placement

| Factor | Weight | Score (1-5) | Weighted |
|--------|--------|-------------|----------|
| Business Value | 30% | ? | ? |
| Technical Readiness | 25% | ? | ? |
| Risk Level (inverse) | 20% | ? | ? |
| Resource Availability | 15% | ? | ? |
| Dependencies Clear | 10% | ? | ? |
| **Total** | 100% | | **?** |

**Higher score = Earlier wave**

## 💬 Workshop Exercise
Score your top 10 applications using this model

---

# Dependency-Driven Sequencing

## Migrate in Correct Order

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        WAVE SEQUENCE                                     │
│                                                                         │
│ Wave 0: Foundation                                                      │
│ ────────────────────────────────────────────────────────────────────── │
│ │ VPN │ IAM │ Networking │ Monitoring │ Landing Zone │                 │
│                                                                         │
│ Wave 1: Shared Services                                                 │
│ ────────────────────────────────────────────────────────────────────── │
│ │ DNS │ AD/LDAP Proxy │ Shared DBs │ Message Queues │                  │
│                                                                         │
│ Wave 2: Backend Services                                                │
│ ────────────────────────────────────────────────────────────────────── │
│ │ Auth Service │ Core APIs │ Data Services │                           │
│                                                                         │
│ Wave 3: Frontend & Consumer Apps                                        │
│ ────────────────────────────────────────────────────────────────────── │
│ │ Web Apps │ Mobile Backends │ Partner APIs │                          │
│                                                                         │
│ Wave 4: Everything Else                                                 │
│ ────────────────────────────────────────────────────────────────────── │
│ │ Batch Jobs │ Reporting │ Dev/Test │ Legacy │                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Wave Planning Template

## Example Wave Structure

| Wave | Timeline | Applications | Key Dependencies | Risk |
|------|----------|--------------|------------------|------|
| **0** | Week 1-4 | Foundation | None | Low |
| **1** | Week 5-7 | Internal Tools App | Shared DB | Low |
| **2** | Week 8-11 | Customer Portal, API Gateway | Auth, DB | Med |
| **3** | Week 12-15 | Order System, Inventory | Portal, Gateway | Med |
| **4** | Week 16-19 | Payment, Shipping | Order, external | High |
| **5** | Week 20-23 | Reporting, Analytics | All data sources | Med |
| **6** | Week 24+ | Legacy, batch, cleanup | N/A | Low |

---

# Timeline Estimation

## Realistic Planning

| Phase | Per Application | Notes |
|-------|-----------------|-------|
| **Assessment** | 2-5 days | Depends on documentation |
| **Planning** | 3-5 days | Runbook, test plan |
| **Preparation** | 1-2 weeks | Target env, replication |
| **Testing** | 1-2 weeks | Functional, performance |
| **Cutover** | 1-4 hours | Plus buffer |
| **Validation** | 2-5 days | Burn-in period |
| **Cleanup** | 1 week | Decommission source |

## Buffer Recommendation
Add **30-50%** to all estimates

---

# Resource Planning

## Team Structure

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      MIGRATION TEAM STRUCTURE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────┐                                               │
│  │  Migration Lead     │ ← Overall coordination, stakeholder mgmt      │
│  └──────────┬──────────┘                                               │
│             │                                                           │
│  ┌──────────┴──────────┬──────────────────┬──────────────────┐         │
│  ▼                     ▼                  ▼                  ▼         │
│ ┌──────────┐    ┌──────────┐      ┌──────────┐      ┌──────────┐      │
│ │ Cloud    │    │ App      │      │ Database │      │ Network  │      │
│ │ Engineer │    │ Teams    │      │ Engineer │      │ Engineer │      │
│ │ (GCP)    │    │ (SMEs)   │      │          │      │          │      │
│ └──────────┘    └──────────┘      └──────────┘      └──────────┘      │
│                                                                         │
│ Support: Security │ Testing │ Operations │ Change Management           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Resource Loading

## Parallel Capacity

| Role | Apps in Parallel | Notes |
|------|------------------|-------|
| Cloud Engineer | 2-3 | Depends on complexity |
| App SME | 1-2 | Need deep knowledge |
| DBA | 2-4 | Can overlap prep/cutover |
| Network | 3-5 | Mostly config work |
| Testing | 2-3 | Automated helps |

## Constraint Planning
```
If you have 2 cloud engineers → max 4-6 apps per wave
If DBAs are bottleneck → don't stack DB-heavy apps
If app teams are part-time → extend wave duration
```

---

<!-- _class: lead -->

# Part 4
## Workshop Facilitation
### Running Effective Discovery Sessions

---

# Workshop Types

## Discovery Workshop Series

| Workshop | Duration | Attendees | Output |
|----------|----------|-----------|--------|
| **Kickoff** | 2 hours | Sponsors, leads | Goals, scope, timeline |
| **Discovery** | 4-8 hours | App owners, tech leads | Inventory, dependencies |
| **Assessment** | 4 hours | Tech leads, architects | 6R decisions, complexity |
| **Wave Planning** | 4 hours | All stakeholders | Wave assignments |
| **Deep Dives** | 2 hours each | Individual app teams | Detailed runbooks |

---

# Discovery Workshop Agenda

## Full-Day Session (8 hours)

| Time | Activity | Duration |
|------|----------|----------|
| 9:00 | Welcome & Objectives | 30 min |
| 9:30 | Current State Overview | 45 min |
| 10:15 | Break | 15 min |
| 10:30 | **Application Inventory Exercise** | 90 min |
| 12:00 | Lunch | 60 min |
| 1:00 | **Dependency Mapping Exercise** | 90 min |
| 2:30 | Break | 15 min |
| 2:45 | **Complexity Scoring Exercise** | 60 min |
| 3:45 | **6R Decision Exercise** | 60 min |
| 4:45 | Wrap-up & Next Steps | 15 min |

---

# Facilitation Best Practices

## Running Effective Sessions

| Do | Don't |
|----|-------|
| ✅ Send pre-work (app list draft) | ❌ Expect people to remember everything |
| ✅ Use visual templates (Miro/Mural) | ❌ Just talk through slides |
| ✅ Timebox discussions | ❌ Let one person dominate |
| ✅ Capture decisions in real-time | ❌ Promise to "write it up later" |
| ✅ Have a parking lot for off-topics | ❌ Go down rabbit holes |
| ✅ Assign homework for unknowns | ❌ Leave gaps unfilled |
| ✅ End with clear next steps | ❌ End without action items |

---

# Workshop Tools

## Recommended Platforms

| Tool | Best For | Features |
|------|----------|----------|
| **Miro** | Visual collaboration | Templates, voting, sticky notes |
| **Mural** | Structured workshops | Facilitation tools, timers |
| **Lucidchart** | Dependency diagrams | Shapes, connectors, layers |
| **Excel/Sheets** | Inventory tracking | Sorting, filtering, formulas |
| **Confluence** | Documentation | Integration, versioning |
| **Jira** | Task tracking | Workflow, assignments |

---

# Capture Template: Application Card

## Collect Per Application

```
┌─────────────────────────────────────────────────────────────────────────┐
│ APPLICATION CARD                                           App ID: ___  │
├─────────────────────────────────────────────────────────────────────────┤
│ Name: _________________________  Owner: ___________________________    │
│ Team: _________________________  Criticality: High / Med / Low         │
│                                                                         │
│ Business Function: _________________________________________________   │
│                                                                         │
│ Tech Stack: ________________________________________________________   │
│ Current Infra: _____________________________________________________   │
│                                                                         │
│ Dependencies (Upstream): ___________________________________________   │
│ Dependencies (Downstream): _________________________________________   │
│                                                                         │
│ Data Classification: Public / Internal / Confidential / Restricted     │
│ Compliance: None / SOC2 / PCI / HIPAA / Other: ______                  │
│                                                                         │
│ SLA: _____ %   RPO: _____   RTO: _____                                 │
│                                                                         │
│ 6R Decision: Retain / Retire / Rehost / Replatform / Refactor / Repurchase │
│ Rationale: _________________________________________________________   │
│                                                                         │
│ Complexity Score: _____/24   Risk: Low / Med / High / Critical         │
│ Proposed Wave: _____                                                    │
│                                                                         │
│ Blockers/Concerns: ________________________________________________    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Capture Template: Dependency Matrix

## Track All Connections

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DEPENDENCY MATRIX                                     │
├─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬────────────┤
│ From/To │ Orders  │ Inventory│ Payment │ Auth    │ SQL DB  │ External   │
├─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼────────────┤
│ Orders  │    -    │  SYNC   │  SYNC   │  SYNC   │  DATA   │   ASYNC    │
├─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼────────────┤
│Inventory│         │    -    │         │  SYNC   │  DATA   │            │
├─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼────────────┤
│ Payment │         │         │    -    │  SYNC   │  DATA   │   SYNC     │
├─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼────────────┤
│ Auth    │         │         │         │    -    │         │   SYNC     │
├─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼────────────┤
│ SQL DB  │         │         │         │         │    -    │            │
└─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴────────────┘

Legend: SYNC = Synchronous API   ASYNC = Queue/Event   DATA = Database
```

---

# Capture Template: Wave Planning Board

## Miro/Mural Layout

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         WAVE PLANNING BOARD                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  BACKLOG          WAVE 1         WAVE 2         WAVE 3         DONE    │
│ ┌─────────┐     ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌───────┐ │
│ │ App X   │     │ App A   │    │ App D   │    │ App G   │    │       │ │
│ │ Score:18│     │ Score:9 │    │ Score:12│    │ Score:15│    │       │ │
│ ├─────────┤     ├─────────┤    ├─────────┤    ├─────────┤    │       │ │
│ │ App Y   │     │ App B   │    │ App E   │    │ App H   │    │       │ │
│ │ Score:22│     │ Score:10│    │ Score:14│    │ Score:17│    │       │ │
│ ├─────────┤     ├─────────┤    ├─────────┤    └─────────┘    │       │ │
│ │ App Z   │     │ App C   │    │ App F   │                   │       │ │
│ │ Score:20│     │ Score:11│    │ Score:13│                   │       │ │
│ └─────────┘     └─────────┘    └─────────┘                   └───────┘ │
│                                                                         │
│ CONSTRAINTS: [ DBA availability: 2 ] [ Freeze: Dec 15-Jan 5 ]          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Pre-Workshop Checklist

## Before the Session

| Task | Owner | Status |
|------|-------|--------|
| Send calendar invites (2 weeks ahead) | PM | ☐ |
| Share pre-read materials | PM | ☐ |
| Request attendees bring app info | PM | ☐ |
| Prepare Miro/Mural board | Facilitator | ☐ |
| Test video conferencing | Facilitator | ☐ |
| Print/share templates | Facilitator | ☐ |
| Identify note-taker | PM | ☐ |
| Prepare parking lot | Facilitator | ☐ |
| Draft initial app list from discovery tools | Engineer | ☐ |

---

# Post-Workshop Actions

## Capture & Follow Through

| Action | Owner | Timeline |
|--------|-------|----------|
| Share workshop recording | PM | Same day |
| Publish notes & decisions | Facilitator | 24 hours |
| Update application inventory | Engineer | 48 hours |
| Create Jira items for follow-ups | PM | 48 hours |
| Schedule deep-dive sessions | PM | 1 week |
| Circulate draft wave plan | Lead | 1 week |
| Obtain stakeholder sign-off | PM | 2 weeks |

---

# Workshop Question Bank

## Questions to Ask Stakeholders

**Business Context:**
- What business process does this app support?
- What happens if this app is down for 1 hour? 1 day?
- Are there seasonal peaks? Blackout periods?

**Technical Details:**
- What databases does this connect to?
- What external APIs or services does it call?
- Is there custom code or is it COTS?

**Migration Concerns:**
- What keeps you up at night about migrating this?
- What's the minimum acceptable downtime?
- Who needs to be involved in the cutover?

**Future State:**
- If you could rebuild this app, what would you change?
- Is this app scheduled for replacement?

---

# Common Workshop Pitfalls

## Avoid These Mistakes

| Pitfall | Solution |
|---------|----------|
| **No decision-makers present** | Require sponsor attendance |
| **Incomplete attendee list** | Map app owners beforehand |
| **Scope creep discussions** | Strict timeboxing + parking lot |
| **Analysis paralysis** | "Good enough" decisions, iterate later |
| **Missing documentation** | Assign homework, follow-up sessions |
| **Optimistic estimates** | Add 30-50% buffer to all estimates |
| **Ignoring dependencies** | Explicit dependency mapping exercise |
| **Forgetting change management** | Include comms & training in plan |

---

# Presentation Tips

## Delivering These Slides

| Tip | Why |
|-----|-----|
| **Skip slides as needed** | Not every slide fits every audience |
| **Use the 💬 prompts** | Discussion points are built in |
| **Fill in examples** | Add client-specific apps/scenarios |
| **Print the templates** | Physical cards work better in person |
| **Take photos** | Capture whiteboard work |
| **Record the session** | Reference for absent stakeholders |
| **Have a scribe** | You can't facilitate AND take notes |

---

# Summary

## Key Takeaways

| Phase | Critical Success Factors |
|-------|--------------------------|
| **Discovery** | Multiple sources, don't trust just one |
| **Assessment** | Use consistent scoring, document rationale |
| **Wave Planning** | Dependencies first, balance each wave |
| **Workshops** | Prepare well, capture everything, follow up |

## Artifacts to Produce
1. ✅ Application Inventory (spreadsheet)
2. ✅ Dependency Map (diagram)
3. ✅ 6R Decisions (documented)
4. ✅ Wave Plan (timeline + assignments)
5. ✅ Risk Register (blockers + mitigations)

---

# Next Steps

1. 📋 **Complete Application Inventory** — Fill gaps from today
2. 🗺️ **Finalize Dependency Maps** — Validate with app teams
3. 📊 **Score All Applications** — Use complexity + risk model
4. 🌊 **Draft Wave Plan** — Sequence based on dependencies
5. 📅 **Schedule Deep Dives** — 2-hour sessions per wave
6. ✅ **Obtain Sign-off** — Stakeholder approval before execution

---

# Questions?

📧 Contact: [your-email]
📚 Resources:
- [Google Cloud Migration Center](https://cloud.google.com/migration-center)
- [Azure Migrate Documentation](https://docs.microsoft.com/azure/migrate)
- [AWS Migration Hub](https://aws.amazon.com/migration-hub/)
- [6 Rs of Migration](https://aws.amazon.com/blogs/enterprise-strategy/6-strategies-for-migrating-applications-to-the-cloud/)

---

# Appendix: Templates

The following slides contain printable templates for workshop use.

---

# Template: Application Card (Printable)

```
┌─────────────────────────────────────────────────────────────────────────┐
│ APPLICATION CARD                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│ App Name: ________________________________  App ID: _________           │
│                                                                         │
│ Owner: ___________________________________  Team: _______________       │
│                                                                         │
│ Business Function: _________________________________________________    │
│                                                                         │
│ Criticality:  ○ Low   ○ Medium   ○ High   ○ Critical                   │
│                                                                         │
│ Tech Stack: ________________________________________________________    │
│                                                                         │
│ Current Infrastructure:                                                 │
│ ____________________________________________________________________    │
│ ____________________________________________________________________    │
│                                                                         │
│ Depends On (Upstream):                                                  │
│ ____________________________________________________________________    │
│                                                                         │
│ Depended On By (Downstream):                                           │
│ ____________________________________________________________________    │
│                                                                         │
│ 6R Decision:  ○ Retain  ○ Retire  ○ Rehost  ○ Replatform              │
│               ○ Refactor  ○ Repurchase                                 │
│                                                                         │
│ Complexity Score: _____/24     Risk Level: ________________            │
│                                                                         │
│ Proposed Wave: _____    Notes: ____________________________________    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Template: Complexity Scoring Worksheet

```
┌─────────────────────────────────────────────────────────────────────────┐
│ COMPLEXITY SCORING                          App Name: ________________  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│ Factor                          │ Score (1-3) │ Notes                   │
│ ────────────────────────────────┼─────────────┼─────────────────────── │
│ Dependencies (1=few, 3=many)    │     ___     │ _____________________  │
│ ────────────────────────────────┼─────────────┼─────────────────────── │
│ Data Volume (1=<100G, 3=>1TB)   │     ___     │ _____________________  │
│ ────────────────────────────────┼─────────────┼─────────────────────── │
│ Compliance (1=none, 3=strict)   │     ___     │ _____________________  │
│ ────────────────────────────────┼─────────────┼─────────────────────── │
│ Availability SLA (1=low, 3=high)│     ___     │ _____________________  │
│ ────────────────────────────────┼─────────────┼─────────────────────── │
│ Tech Stack Age (1=new, 3=old)   │     ___     │ _____________________  │
│ ────────────────────────────────┼─────────────┼─────────────────────── │
│ Documentation (1=good, 3=none)  │     ___     │ _____________________  │
│ ────────────────────────────────┼─────────────┼─────────────────────── │
│ Team Knowledge (1=high, 3=low)  │     ___     │ _____________________  │
│ ────────────────────────────────┼─────────────┼─────────────────────── │
│ Custom Code (1=little, 3=heavy) │     ___     │ _____________________  │
│ ────────────────────────────────┼─────────────┼─────────────────────── │
│                                                                         │
│                           TOTAL │     ___/24  │ ○ Low  ○ Med  ○ High   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
