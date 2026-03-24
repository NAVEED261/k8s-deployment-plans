# 🧠 K8 Planning Skill — Reusable Kubernetes Deployment Planning Agent

> **Skill Name:** K8 Planning Skill  
> **Purpose:** Generate complete, production-grade Kubernetes deployment plans for any project  
> **Author:** NAVEED261  
> **Repo:** https://github.com/NAVEED261/k8s-deployment-plans  
> **Version:** 1.0.0

---

## 📋 Table of Contents

1. [Skill Overview](#skill-overview)
2. [When to Use This Skill](#when-to-use-this-skill)
3. [Input Schema](#input-schema)
4. [Planning Framework](#planning-framework)
5. [Step-by-Step Planning Process](#step-by-step-planning-process)
6. [Decision Trees](#decision-trees)
7. [Resource Sizing Guide](#resource-sizing-guide)
8. [Security Checklist](#security-checklist)
9. [Output Template](#output-template)
10. [Example Usage](#example-usage)
11. [Skill Evaluation Criteria](#skill-evaluation-criteria)

---

## 1. Skill Overview

This skill enables any AI agent or developer to generate a **complete, professional Kubernetes deployment plan** for any software project — without writing actual Kubernetes YAML code.

### What This Skill Produces

Given a project description, this skill outputs a structured Markdown document covering:

| Output Section | Description |
|---|---|
| **Namespaces** | Logical isolation strategy |
| **Pods & Deployments** | Which workloads need Deployment vs StatefulSet |
| **Services** | ClusterIP, NodePort, or LoadBalancer decisions |
| **Resource Requests/Limits** | CPU and Memory sizing |
| **ConfigMaps** | Non-sensitive configuration |
| **Secrets** | Sensitive data management + rotation |
| **RBAC** | Roles, ServiceAccounts, RoleBindings |
| **Inter-service Communication** | DNS, NetworkPolicies |
| **Architecture Diagram** | Text-based visual |

---

## 2. When to Use This Skill

Use this skill when:

- ✅ Starting a new project and need K8s infrastructure planning
- ✅ Converting a monolith to microservices
- ✅ Planning deployment for an AI/ML application
- ✅ Designing a secure multi-tenant platform
- ✅ Creating documentation for existing K8s deployments
- ✅ Reviewing/auditing a K8s architecture

Do NOT use when:
- ❌ Need actual running YAML files (use K8s generators instead)
- ❌ Project has only 1 service (use simple Docker Compose instead)
- ❌ Need real-time cluster management

---

## 3. Input Schema

To use this skill, provide answers to these questions:

### Required Inputs

```
PROJECT_NAME: <string>
  Example: "E-commerce Platform"

COMPONENTS: <list of services>
  Example: ["Frontend", "Product API", "Order Service", "Payment Gateway", "Database"]

ENVIRONMENT: <dev | staging | production>
  Example: "production"

SECURITY_LEVEL: <standard | high | maximum>
  Example: "high"
  - standard: Basic secrets, ClusterIP, simple RBAC
  - high: Vault secrets, NetworkPolicies, non-root containers
  - maximum: gVisor, default-deny-all, hardware secrets, full audit

HAS_DATABASE: <yes | no>
  If yes: <postgres | mysql | mongodb | redis | elasticsearch | qdrant | other>

HAS_AI_COMPONENTS: <yes | no>
  If yes: <which LLM providers, what AI tasks>

EXPECTED_TRAFFIC: <low | medium | high | very-high>
  - low: < 1000 requests/day
  - medium: 1000-100,000 requests/day
  - high: 100,000-1M requests/day
  - very-high: > 1M requests/day

TEAM_SIZE: <small (1-5) | medium (5-20) | large (20+)>

PUBLIC_FACING: <yes | no>
  Which services are exposed to internet?
```

### Optional Inputs

```
CLOUD_PROVIDER: <aws | gcp | azure | on-prem | any>
COMPLIANCE_REQUIREMENTS: <hipaa | pci-dss | gdpr | soc2 | none>
EXISTING_TECH_STACK: <list of technologies already in use>
BUDGET_CONSTRAINTS: <low | medium | high>
```

---

## 4. Planning Framework

### The K8P Framework (Kubernetes Planning Process)

```
K8P = Namespace → Workload → Network → Data → Security → Ops
```

Every K8s plan follows this exact order:

```
STEP 1: NAMESPACE DESIGN
        ↓
STEP 2: WORKLOAD CLASSIFICATION
        (Deployment vs StatefulSet vs Job vs CronJob)
        ↓
STEP 3: SERVICE TYPE SELECTION
        (ClusterIP vs NodePort vs LoadBalancer vs Ingress)
        ↓
STEP 4: RESOURCE SIZING
        (CPU/Memory requests and limits)
        ↓
STEP 5: CONFIGURATION MANAGEMENT
        (ConfigMaps for non-secrets, Secrets for sensitive data)
        ↓
STEP 6: SECURITY DESIGN
        (RBAC, NetworkPolicies, PodSecurity)
        ↓
STEP 7: COMMUNICATION MAP
        (DNS-based service discovery, NetworkPolicies)
        ↓
STEP 8: OPERATIONS PLAN
        (HPA, PodDisruptionBudgets, monitoring)
```

---

## 5. Step-by-Step Planning Process

### STEP 1: Namespace Design

**Rule:** Group services by team ownership and security boundary.

```
NAMESPACE PATTERNS:

Pattern A — By Layer (simple apps):
├── <app>-frontend
├── <app>-backend
├── <app>-data
└── <app>-monitoring

Pattern B — By Team (large apps):
├── <app>-payments-team
├── <app>-catalog-team
├── <app>-auth-team
└── <app>-data

Pattern C — By Security Level (security-critical apps):
├── <app>-public        (internet-facing)
├── <app>-internal      (business logic)
├── <app>-restricted    (sensitive data + AI)
├── <app>-data          (databases)
└── <app>-security      (audit, secrets management)
```

**Decision Rule:**
- Small project (< 5 services) → Pattern A
- Medium project (5-15 services) → Pattern B
- Security-critical project → Pattern C

---

### STEP 2: Workload Classification

**For each component, ask these questions:**

```
Q1: Does this component store data that must survive pod restarts?
    YES → StatefulSet
    NO  → Deployment

Q2: Is this a one-time task?
    YES → Job
    NO  → Continue

Q3: Does this run on a schedule?
    YES → CronJob
    NO  → Deployment or StatefulSet
```

**Quick Reference:**

| Component Type | Kubernetes Object |
|---|---|
| Web frontend | Deployment |
| REST API | Deployment |
| AI Agent/Worker | Deployment |
| Message queue consumer | Deployment |
| PostgreSQL / MySQL | StatefulSet |
| MongoDB | StatefulSet |
| Redis (persistent) | StatefulSet |
| Elasticsearch | StatefulSet |
| Qdrant / Weaviate | StatefulSet |
| Data migration | Job |
| Report generator | CronJob |
| Secret expiry checker | CronJob |
| Backup job | CronJob |

---

### STEP 3: Service Type Selection

```
Is this service accessed from the internet?
    YES → LoadBalancer (or Ingress + ClusterIP)
    NO  → Continue

Is this service accessed from outside the cluster but inside VPC?
    YES → NodePort
    NO  → ClusterIP (default — most secure)

Is this a database or internal cache?
    YES → ClusterIP (NEVER expose databases publicly)
```

**Service Type Decision Table:**

| Use Case | Service Type | Notes |
|---|---|---|
| Public website | LoadBalancer | Add TLS termination |
| Public API | LoadBalancer or Ingress | Rate limiting recommended |
| Internal microservice | ClusterIP | Default choice |
| Database | ClusterIP | Never expose publicly |
| Cache (Redis) | ClusterIP | Never expose publicly |
| AI agent (internal) | ClusterIP | Internal communication only |
| Admin panel | NodePort or Ingress | Restrict by IP whitelist |
| Monitoring (Grafana) | NodePort | Internal VPN access only |

---

### STEP 4: Resource Sizing Guide

#### CPU Sizing

| Component Type | Request | Limit | Reason |
|---|---|---|---|
| Simple frontend (React/Next.js) | 50m–100m | 200m–500m | Low compute |
| REST API (Node/Go/Python) | 100m–250m | 500m–1000m | Moderate |
| AI Agent (LLM calls) | 500m–1000m | 2000m–4000m | Heavy processing |
| ML inference service | 1000m–2000m | 4000m–8000m | Very heavy |
| PostgreSQL | 250m–500m | 1000m–4000m | Burst capacity needed |
| Redis | 50m–100m | 500m–1000m | Memory-primary |
| Elasticsearch | 500m–1000m | 2000m–4000m | JVM overhead |
| Qdrant | 250m–500m | 2000m–4000m | Vector operations |

#### Memory Sizing

| Component Type | Request | Limit | Reason |
|---|---|---|---|
| Simple frontend | 64Mi–128Mi | 256Mi–512Mi | Static serving |
| REST API | 128Mi–256Mi | 512Mi–1Gi | Business logic |
| AI Agent | 512Mi–2Gi | 2Gi–8Gi | LLM context window |
| ML inference | 2Gi–8Gi | 8Gi–32Gi | Model weights |
| PostgreSQL | 512Mi–1Gi | 2Gi–8Gi | Buffer pool |
| Redis | 256Mi–512Mi | 1Gi–4Gi | Dataset size |
| Elasticsearch | 2Gi–4Gi | 4Gi–16Gi | JVM heap |
| Qdrant | 512Mi–1Gi | 2Gi–8Gi | Vector index |

#### Replica Count

| Traffic Level | Replicas | Notes |
|---|---|---|
| Low (dev/test) | 1 | No HA needed |
| Medium (staging) | 2 | Basic HA |
| High (production) | 3+ | Full HA with rolling updates |
| Very high (enterprise) | 5+ | Use HPA for auto-scaling |

---

### STEP 5: ConfigMap Design

**Rule:** ConfigMaps = anything non-secret that can change per environment.

```
ConfigMap should contain:
✅ Service URLs and ports
✅ Feature flags
✅ Timeout values
✅ Log levels
✅ External service endpoints (non-auth)
✅ Application settings
✅ Non-sensitive database names

ConfigMap should NOT contain:
❌ Passwords
❌ API keys
❌ Private keys
❌ Tokens
❌ Any credential
```

**ConfigMap Naming Convention:**
```
<app-name>-<component>-config

Examples:
  myapp-frontend-config
  myapp-backend-config
  myapp-agent-config
  myapp-notification-config
```

---

### STEP 6: Secrets Design

**Rule:** Never store secrets in plain text. Always use base64 minimum, Vault in production.

```
Secret Hierarchy:

LEVEL 1 — Development:
  K8s native Secrets (base64 encoded)

LEVEL 2 — Staging:
  Sealed Secrets (Bitnami) — encrypted in Git

LEVEL 3 — Production:
  External Secrets Operator + HashiCorp Vault
  OR AWS Secrets Manager / GCP Secret Manager

LEVEL 4 — Maximum Security:
  Vault + KMS + Hardware Security Module (HSM)
```

**Secret Rotation Policy:**

| Secret Type | Rotation Frequency | Method |
|---|---|---|
| JWT signing keys | 30 days | Automated CronJob |
| Session tokens | 24 hours | Auto-refresh |
| Database passwords | 90 days | Vault dynamic secrets |
| User encryption keys | 180 days | Manual + key versioning |
| LLM API keys | On compromise | Manual |
| OAuth client secrets | Annual | Manual |
| TLS certificates | 90 days (Let's Encrypt) | cert-manager auto |

**Secret Naming Convention:**
```
<app-name>-<service>-secret

Examples:
  myapp-db-secret
  myapp-jwt-secret
  myapp-openai-secret
  myapp-smtp-secret
```

---

### STEP 7: RBAC Design

**The 4-Role Model** — works for most projects:

```
ROLE 1: Admin
  → Full cluster access
  → Only senior DevOps/SRE team
  → Verbs: ["*"]

ROLE 2: Developer
  → Read-only access to pods, deployments, logs
  → NO secret access
  → Verbs: ["get", "list", "watch"]

ROLE 3: Service Account (per microservice)
  → Minimum access for that specific service
  → Only the secrets it needs (by name)
  → Verbs: ["get"] for secrets, ["get","list"] for configmaps

ROLE 4: CI/CD Pipeline
  → Can deploy (create/update deployments)
  → Cannot access secrets
  → Verbs: ["get", "list", "create", "update", "patch"] on deployments
```

**RBAC Rules:**

```
Rule 1: Always use ServiceAccounts, not user accounts, for pods
Rule 2: Use resourceNames to restrict which secrets a SA can access
Rule 3: Use Role (namespace-scoped) over ClusterRole when possible
Rule 4: Never give wildcard (*) verbs to application service accounts
Rule 5: Developers should have ZERO access to production secrets
```

---

### STEP 8: Communication Design

**DNS Format:**
```
<service-name>.<namespace>.svc.cluster.local:<port>
```

**NetworkPolicy Rules:**

```
Rule 1: Start with default-deny-all in every namespace
Rule 2: Then selectively allow what's needed
Rule 3: Databases should only accept traffic from their consumers
Rule 4: Sandbox/execution namespaces should deny all egress to internal cluster
Rule 5: Only LoadBalancer services should accept internet traffic
```

---

## 6. Decision Trees

### Decision Tree 1: Should I Use Deployment or StatefulSet?

```
                    ┌─────────────────┐
                    │  New Component  │
                    └────────┬────────┘
                             │
              ┌──────────────▼──────────────┐
              │ Does it store persistent    │
              │ data that must survive      │
              │ pod restarts?               │
              └──────┬──────────────┬───────┘
                     │              │
                    YES             NO
                     │              │
          ┌──────────▼──┐   ┌───────▼──────────┐
          │ StatefulSet │   │ Is it a one-time  │
          │             │   │ task or job?      │
          └─────────────┘   └──────┬────────────┘
                                   │
                        ┌──────────▼──────────┐
                        │       YES           NO│
                        │         │             │
                   ┌────▼──┐  ┌───▼────────┐
                   │  Job  │  │ Deployment │
                   └───────┘  └────────────┘
```

### Decision Tree 2: Which Service Type?

```
                  ┌────────────────────┐
                  │  Service Needed    │
                  └────────┬───────────┘
                           │
            ┌──────────────▼──────────────┐
            │ Accessed from internet?      │
            └──────┬──────────────┬───────┘
                   │              │
                  YES             NO
                   │              │
        ┌──────────▼──┐  ┌────────▼──────────┐
        │LoadBalancer │  │ Accessed from      │
        │ or Ingress  │  │ outside cluster?   │
        └─────────────┘  └──────┬─────────────┘
                                │
                     ┌──────────▼──────────┐
                     │     YES         NO  │
                     │      │           │  │
                ┌────▼───┐ ┌▼──────────┐│
                │NodePort│ │ ClusterIP ││
                └────────┘ └───────────┘│
```

---

## 7. Resource Sizing Guide

### Quick Sizing Formula

```
For any component, start here:

DEVELOPMENT:
  CPU Request  = Target CPU Limit × 0.1
  Memory Req   = Target Memory Limit × 0.5
  Replicas     = 1

STAGING:
  CPU Request  = Target CPU Limit × 0.25
  Memory Req   = Target Memory Limit × 0.5
  Replicas     = 2

PRODUCTION:
  CPU Request  = Target CPU Limit × 0.3
  Memory Req   = Target Memory Limit × 0.7
  Replicas     = 3+ (or use HPA)
```

### AI Component Special Sizing

```
AI agents need extra headroom because:
1. LLM API calls can spike CPU during streaming
2. Context windows consume large memory
3. Vector operations need burst capacity

Recommended for AI components:
  CPU Limit   = Base limit × 2
  Memory Limit = Base limit × 3
  Add HPA with CPU threshold at 60% (not 80%)
```

---

## 8. Security Checklist

Use this checklist for every plan you generate:

### Pod Security
- [ ] `runAsNonRoot: true` set on all pods
- [ ] `runAsUser` is not 0 (root)
- [ ] `allowPrivilegeEscalation: false` on all containers
- [ ] `readOnlyRootFilesystem: true` where possible
- [ ] `capabilities.drop: ["ALL"]` on all containers
- [ ] No `hostPath` volume mounts (except monitoring)
- [ ] No `hostNetwork: true`
- [ ] No `privileged: true`

### Network Security
- [ ] Default-deny NetworkPolicy in every namespace
- [ ] Only necessary ingress/egress rules added
- [ ] Databases have no external exposure
- [ ] Only LoadBalancer uses HTTPS (port 443)
- [ ] Sandbox namespaces deny egress to internal IPs

### Secret Security
- [ ] No secrets in ConfigMaps
- [ ] No secrets in environment variables as plain text
- [ ] All secrets use `secretKeyRef`
- [ ] ServiceAccounts use `resourceNames` to restrict secret access
- [ ] Rotation schedule defined for every secret
- [ ] Production uses Vault or cloud secret manager

### RBAC Security
- [ ] Every pod has a dedicated ServiceAccount
- [ ] No ServiceAccount uses `automountServiceAccountToken: true` unnecessarily
- [ ] Developers have zero secret access in production
- [ ] `ClusterRole` used only when truly cluster-wide access is needed
- [ ] No wildcard (`*`) verbs on application service accounts

---

## 9. Output Template

When using this skill, generate your plan using this template:

```markdown
# 🚀 [PROJECT_NAME] — Kubernetes Deployment Plan

> Environment: [ENV] | Security: [LEVEL] | Traffic: [TRAFFIC]

## 1. Architecture Overview
[Table of components and their roles]

## 2. Namespaces
[YAML + explanation table]

## 3. Pods and Deployments / StatefulSets
[YAML for each component with explanations]

## 4. Services and Their Types
[Summary table + YAML for each service]

## 5. Resource Requests and Limits
[Complete resource table for all components]

## 6. ConfigMaps
[YAML for each ConfigMap]

## 7. Secrets Management
[YAML for each secret + rotation schedule]

## 8. RBAC
[ServiceAccounts + Roles + RoleBindings]

## 9. Inter-Service Communication
[DNS table + NetworkPolicy YAML]

## 10. Architecture Diagram
[ASCII text diagram]

## ✅ Summary Checklist
[All requirements checked off]
```

---

## 10. Example Usage

### Example: Generate a plan for an E-commerce Platform

**Input:**
```
PROJECT_NAME: ShopEasy E-commerce
COMPONENTS: [Frontend, Product API, Order Service, Payment Service, User Auth, Email Service]
ENVIRONMENT: production
SECURITY_LEVEL: high (handles payment data)
HAS_DATABASE: yes (PostgreSQL, Redis)
HAS_AI_COMPONENTS: no
EXPECTED_TRAFFIC: high
TEAM_SIZE: medium
PUBLIC_FACING: [Frontend, Product API, User Auth]
CLOUD_PROVIDER: aws
COMPLIANCE_REQUIREMENTS: pci-dss
```

**Agent Thinking Process:**

```
1. NAMESPACE DESIGN:
   → 6 services → Pattern B (by team/function)
   → shopeasy-frontend, shopeasy-apis, shopeasy-auth, shopeasy-data, shopeasy-ops

2. WORKLOAD CLASSIFICATION:
   → Frontend: Deployment (stateless)
   → Product API: Deployment (stateless)
   → Order Service: Deployment (stateless)
   → Payment Service: Deployment (stateless, but extra security)
   → User Auth: Deployment (stateless)
   → Email Service: Deployment (stateless)
   → PostgreSQL: StatefulSet (persistent data)
   → Redis: StatefulSet (persistent sessions)

3. SERVICE TYPES:
   → Frontend: LoadBalancer (internet-facing)
   → Product API: LoadBalancer (internet-facing)
   → User Auth: LoadBalancer (internet-facing)
   → Order Service: ClusterIP (internal)
   → Payment Service: ClusterIP (internal, never expose)
   → Email Service: ClusterIP (internal)
   → PostgreSQL: ClusterIP (never expose)
   → Redis: ClusterIP (never expose)

4. SECURITY (PCI-DSS):
   → Payment Service gets its own isolated namespace
   → All secrets via AWS Secrets Manager
   → Audit logging for all payment operations
   → Network deny-all + selective allow
   → Non-root containers everywhere

5. RESOURCES (high traffic):
   → Frontend: 200m/500m CPU, 256Mi/512Mi RAM, 3 replicas + HPA
   → APIs: 250m/1000m CPU, 512Mi/1Gi RAM, 3 replicas + HPA
   → Payment: 250m/750m CPU, 256Mi/512Mi RAM, 2 replicas (stable)
   → PostgreSQL: 1000m/4000m CPU, 2Gi/8Gi RAM
```

---

## 11. Skill Evaluation Criteria

### How to Evaluate If a K8s Plan Is Complete (100 marks)

| Category | Max Marks | Evaluation Criteria |
|---|---|---|
| **Namespace Design** | 10 | Logical isolation, security boundaries defined |
| **Workload Classification** | 15 | Correct Deployment vs StatefulSet, replicas justified |
| **Service Types** | 10 | Correct ClusterIP vs LoadBalancer, no database exposure |
| **Resource Sizing** | 10 | Requests < Limits, sized appropriately for component type |
| **ConfigMaps** | 10 | No secrets in ConfigMaps, proper naming, all config covered |
| **Secrets Management** | 15 | Rotation defined, proper encoding, Vault plan for prod |
| **RBAC** | 15 | ServiceAccounts per pod, resourceNames used, dev has no secret access |
| **Inter-service Comm** | 10 | DNS addresses correct, NetworkPolicy deny-all present |
| **Architecture Diagram** | 5 | Clear visual of all components and connections |
| **Security Checklist** | Bonus +5 | All security controls applied |

### Scoring Rubric

| Score | Grade | Meaning |
|---|---|---|
| 90-100 | A+ | Production-ready plan |
| 80-89 | A | Minor improvements needed |
| 70-79 | B | Several gaps, review required |
| 60-69 | C | Significant missing sections |
| < 60 | F | Major redesign needed |

---

## 📝 Quick Reference Card

```
┌────────────────────────────────────────────────────────────────┐
│              K8 PLANNING SKILL — QUICK REFERENCE               │
├────────────────────────────────────────────────────────────────┤
│ WORKLOADS:                                                     │
│   Stateless service     → Deployment                          │
│   Database/persistent   → StatefulSet                         │
│   One-time task         → Job                                 │
│   Scheduled task        → CronJob                             │
├────────────────────────────────────────────────────────────────┤
│ SERVICES:                                                      │
│   Internet-facing       → LoadBalancer                        │
│   Internal only         → ClusterIP                           │
│   VPC-internal          → NodePort                            │
│   Never expose DB       → ClusterIP only                      │
├────────────────────────────────────────────────────────────────┤
│ SECURITY:                                                      │
│   runAsNonRoot: true     Always                               │
│   readOnlyRootFilesystem Always where possible                │
│   capabilities.drop ALL  Always                               │
│   Default deny network   Always                               │
│   Vault for prod secrets Always                               │
├────────────────────────────────────────────────────────────────┤
│ RESOURCES (Production):                                        │
│   Frontend    100m/300m CPU  | 128Mi/256Mi RAM                │
│   API         250m/750m CPU  | 256Mi/512Mi RAM                │
│   AI Agent    500m/4000m CPU | 512Mi/8Gi RAM                  │
│   Database    500m/2000m CPU | 1Gi/4Gi RAM                    │
├────────────────────────────────────────────────────────────────┤
│ NAMESPACE PATTERN:                                             │
│   <5 services   → By Layer (frontend/backend/data)            │
│   5-15 services → By Team                                     │
│   Security-crit → By Security Level                           │
└────────────────────────────────────────────────────────────────┘
```

---

*K8 Planning Skill v1.0 | Created by NAVEED261 | GitHub: k8s-deployment-plans*
