---
marp: true
theme: default
paginate: true
header: 'GKE Security Best Practices'
footer: 'Multi-Tenant Cluster Architecture'
---

# GKE Security Best Practices
## Multi-Tenant Cluster Architecture

*Namespace isolation, Workload Identity, and Network Design for 20+ Deployments*

---

# Agenda

1. **Tenant Isolation** — Namespace-per-app model
2. **Identity & Access** — Service accounts + Workload Identity
3. **GCP Project Structure** — App-specific projects
4. **Network Architecture** — Ingress, LB, and API Gateway
5. **Security Controls** — Network policies, Pod Security
6. **Recommendations** — Decision matrix for 20 deployments

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

# Namespace Best Practices

| Practice | Implementation |
|----------|---------------|
| Naming convention | `app-{name}` or `{team}-{app}` |
| Labels | `app`, `team`, `env`, `cost-center` |
| Resource quotas | CPU/memory limits per namespace |
| Limit ranges | Default container limits |
| Network policies | Deny-all default, explicit allow |

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-orders
  labels:
    app: orders
    team: platform
    env: prod
```

---

# Identity: Service Account Per App

## The Problem with Default SA

❌ Shared `default` SA across apps = blast radius  
❌ Overprivileged access  
❌ No audit trail per app  

## The Solution

✅ **Dedicated K8s SA per app**  
✅ **Dedicated GCP SA per app**  
✅ **Workload Identity binding**  

---

# Workload Identity Federation (WIF)

## How It Works

```
┌─────────────────────────────────────────────────────┐
│ GKE Cluster                                          │
│  ┌──────────────────┐                               │
│  │ Pod (app-orders) │                               │
│  │ SA: orders-sa    │ ──────┐                       │
│  └──────────────────┘       │ WIF                   │
│                              ▼                       │
└───────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────┐
│ GCP IAM                                              │
│  orders-app@project.iam.gserviceaccount.com         │
│  Roles: Storage Object Viewer, Pub/Sub Publisher    │
└─────────────────────────────────────────────────────┘
```

---

# WIF Configuration

```yaml
# 1. K8s ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-sa
  namespace: app-orders
  annotations:
    iam.gke.io/gcp-service-account: orders-app@proj.iam.gserviceaccount.com
---
# 2. GCP IAM binding (Terraform)
resource "google_service_account_iam_member" "workload_identity" {
  service_account_id = google_service_account.orders.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:PROJECT.svc.id.goog[app-orders/orders-sa]"
}
```

---

# Project Structure for Dependencies

## App-Specific GCP Projects

```
org/
├── platform-gke-prod/          # Shared GKE cluster
├── app-orders-prod/            # Orders app resources
│   ├── Cloud SQL
│   ├── Cloud Storage
│   └── Pub/Sub topics
├── app-inventory-prod/         # Inventory app resources
├── app-payments-prod/          # Payments app resources
└── shared-services-prod/       # Shared infra (logging, etc.)
```

**Benefits:**
- Billing isolation
- IAM boundary per app
- Resource quota per project
- Separate audit logs

---

# Shared vs Dedicated Projects

| Resource Type | Recommendation |
|---------------|----------------|
| GKE Cluster | Shared (platform-gke) |
| Cloud SQL | Dedicated per app |
| Pub/Sub | Dedicated or shared topic project |
| Cloud Storage | Dedicated per app |
| Secret Manager | Dedicated per app |
| VPC/Networking | Shared (host project) |

**Rule of thumb:** Anything with app data = dedicated project

---

# Network Architecture: 20 Deployments

## The Challenge

- 20 backend API deployments
- All need external exposure
- Security + cost optimization
- Observability

## Options

1. **Single Ingress Controller** (recommended)
2. **Cloud Load Balancer per service** (expensive)
3. **API Gateway** (depends on needs)

---

# Option 1: Single Ingress (Recommended)

```
                    ┌──────────────────────┐
                    │   Cloud Load Balancer │
                    │   (single external IP) │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │   Ingress Controller  │
                    │   (GKE Ingress/NGINX) │
                    └──────────┬───────────┘
                               │
        ┌──────────┬───────────┼───────────┬──────────┐
        ▼          ▼           ▼           ▼          ▼
   app-orders  app-inventory  app-payments  ...   app-20
   ClusterIP   ClusterIP      ClusterIP          ClusterIP
```

---

# GKE Gateway API (Modern Approach)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: external-gateway
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: orders-route
  namespace: app-orders
spec:
  parentRefs:
  - name: external-gateway
  hostnames: ["api.example.com"]
  rules:
  - matches:
    - path: {type: PathPrefix, value: "/orders"}
    backendRefs:
    - name: orders-service
      port: 8080
```

---

# Do You Need API Gateway?

## Use GKE Ingress When:
- ✅ Simple path-based routing
- ✅ TLS termination
- ✅ Basic rate limiting (via annotations)
- ✅ 20 internal backend APIs

## Use API Gateway (Apigee/Cloud Endpoints) When:
- 🔶 API monetization/quotas per client
- 🔶 OAuth/API key management
- 🔶 Request/response transformation
- 🔶 Developer portal needed
- 🔶 Multi-cloud API facade

---

# Recommendation for 20 Backend APIs

## **Use GKE Gateway API + Internal Load Balancer**

```
Internet
    │
    ▼
┌───────────────────┐
│ Cloud Armor (WAF) │
└─────────┬─────────┘
          ▼
┌───────────────────┐
│ External Gateway  │ ← Single entry point
└─────────┬─────────┘
          ▼
┌───────────────────┐
│ 20 HTTPRoutes     │ ← Path-based routing
│ (one per app)     │
└───────────────────┘
```

**Cost:** 1 LB vs 20 LBs = significant savings

---

# Network Policies: Default Deny

```yaml
# Default deny all ingress in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: app-orders
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Allow from ingress controller only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
spec:
  podSelector:
    matchLabels:
      app: orders
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
```

---

# Cross-Namespace Communication

## Service-to-Service Policies

```yaml
# Allow orders to call inventory
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-orders
  namespace: app-inventory
spec:
  podSelector:
    matchLabels:
      app: inventory
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          app: orders
    ports:
    - port: 8080
```

**Principle:** Explicit allow, deny everything else

---

# Pod Security Standards

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-orders
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

## Restricted Profile Requires:
- Non-root user
- Read-only root filesystem
- No privilege escalation
- Dropped capabilities
- Seccomp profile

---

# Complete Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Internet                             │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ Cloud Armor (WAF) + Cloud CDN                               │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ GKE Gateway (External LB) - Single IP                       │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ GKE Cluster (platform-gke-prod)                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐     ┌─────────┐       │
│  │app-orders│ │app-inv  │ │app-pay  │ ... │app-20   │       │
│  │ NS + SA  │ │ NS + SA │ │ NS + SA │     │ NS + SA │       │
│  │ NetPol   │ │ NetPol  │ │ NetPol  │     │ NetPol  │       │
│  └────┬─────┘ └────┬────┘ └────┬────┘     └────┬────┘       │
│       │ WIF        │ WIF       │ WIF           │ WIF        │
└───────┼────────────┼───────────┼───────────────┼────────────┘
        ▼            ▼           ▼               ▼
┌─────────────┐ ┌─────────┐ ┌─────────┐    ┌─────────┐
│orders-proj  │ │inv-proj │ │pay-proj │    │app20-proj│
│ SQL, GCS    │ │ SQL     │ │ SQL     │    │ ...      │
└─────────────┘ └─────────┘ └─────────┘    └──────────┘
```

---

# Security Checklist

| Control | Implementation |
|---------|---------------|
| ✅ Namespace isolation | 1 namespace per app |
| ✅ RBAC | Team-scoped roles per namespace |
| ✅ Workload Identity | K8s SA → GCP SA per app |
| ✅ Network policies | Default deny + explicit allow |
| ✅ Pod Security | Restricted PSS enforcement |
| ✅ Secrets | External Secrets + Secret Manager |
| ✅ Ingress | Single Gateway, path routing |
| ✅ WAF | Cloud Armor on external LB |
| ✅ Audit | GKE audit logs → Cloud Logging |

---

# Decision Matrix: API Gateway vs Ingress

| Requirement | Ingress | API Gateway |
|-------------|---------|-------------|
| Path routing | ✅ | ✅ |
| TLS termination | ✅ | ✅ |
| Rate limiting | ⚠️ Basic | ✅ Advanced |
| API keys/OAuth | ❌ | ✅ |
| Request transform | ❌ | ✅ |
| Analytics/Monetization | ❌ | ✅ |
| Cost (20 APIs) | $ | $$$ |
| Complexity | Low | High |

**For 20 internal backend APIs: Use GKE Ingress/Gateway**

---

# Summary

1. **Namespace per app** — Logical isolation + RBAC boundary
2. **Service Account per app** — K8s SA + GCP SA via WIF
3. **Project per app** — Data isolation + billing clarity
4. **Single Ingress** — Cost effective for 20 deployments
5. **Network Policies** — Default deny, explicit allow
6. **Skip API Gateway** — Unless you need monetization/dev portal

---

# Questions?

📧 Contact: [your-email]
📚 Resources:
- [GKE Best Practices](https://cloud.google.com/kubernetes-engine/docs/best-practices)
- [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity)
- [Gateway API](https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api)
