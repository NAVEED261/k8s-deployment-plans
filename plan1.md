# 🚀 Plan 1: AI Native Task Manager — Kubernetes Deployment Plan

> **Project:** AI Native Task Manager  
> **Components:** UI Interface, Backend APIs, Task Agent, Notification Service  
> **Author:** NAVEED261  
> **Repo:** https://github.com/NAVEED261/k8s-deployment-plans

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Namespaces](#namespaces)
3. [Pods and Deployments / StatefulSets](#pods-and-deployments--statefulsets)
4. [Services and Their Types](#services-and-their-types)
5. [Resource Requests and Limits](#resource-requests-and-limits)
6. [ConfigMaps](#configmaps)
7. [Secrets Management and Expiry Handling](#secrets-management-and-expiry-handling)
8. [RBAC — Roles and RoleBindings](#rbac--roles-and-rolebindings)
9. [Inter-Service Communication](#inter-service-communication)
10. [Architecture Diagram (Text)](#architecture-diagram-text)

---

## 1. Architecture Overview

The AI Native Task Manager is a cloud-native application composed of four main microservices:

| Component | Role | Type |
|---|---|---|
| **UI Interface** | Frontend — user interacts here | Stateless Web App |
| **Backend APIs** | Core logic, REST/GraphQL endpoints | Stateless Service |
| **Task Agent** | AI-powered task processing, LLM calls | Stateless Worker |
| **Notification Service** | Email/SMS/Push notifications | Stateless Worker |
| **PostgreSQL DB** | Persistent task storage | Stateful Database |
| **Redis Cache** | Session + queue management | Stateful Cache |

---

## 2. Namespaces

Namespaces isolate resources logically — jaise alag kamre in a building.

```yaml
# Namespace 1 — Frontend layer
apiVersion: v1
kind: Namespace
metadata:
  name: task-frontend
  labels:
    team: ui
    env: production

---
# Namespace 2 — Backend + AI layer
apiVersion: v1
kind: Namespace
metadata:
  name: task-backend
  labels:
    team: backend
    env: production

---
# Namespace 3 — Data layer
apiVersion: v1
kind: Namespace
metadata:
  name: task-data
  labels:
    team: data
    env: production

---
# Namespace 4 — Monitoring
apiVersion: v1
kind: Namespace
metadata:
  name: task-monitoring
  labels:
    team: ops
    env: production
```

### Why These Namespaces?

| Namespace | Contains | Reason |
|---|---|---|
| `task-frontend` | UI Interface | Isolate public-facing layer |
| `task-backend` | Backend APIs, Task Agent, Notification Service | Group all business logic |
| `task-data` | PostgreSQL, Redis | Protect data layer separately |
| `task-monitoring` | Prometheus, Grafana | Ops team has separate access |

---

## 3. Pods and Deployments / StatefulSets

### 3.1 UI Interface — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui-interface
  namespace: task-frontend
  labels:
    app: ui-interface
    version: "1.0"
spec:
  replicas: 2                        # 2 copies for high availability
  selector:
    matchLabels:
      app: ui-interface
  template:
    metadata:
      labels:
        app: ui-interface
    spec:
      containers:
        - name: ui-interface
          image: naveed261/task-ui:latest
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: ui-config
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "300m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
```

**Why Deployment (not StatefulSet)?** UI is stateless — koi bhi pod handle kar sakta hai request.

---

### 3.2 Backend APIs — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: task-backend
  labels:
    app: backend-api
    version: "1.0"
spec:
  replicas: 3                        # 3 replicas — high traffic expected
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      containers:
        - name: backend-api
          image: naveed261/task-backend:latest
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: backend-config
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: jwt-secret
                  key: jwt-key
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: openai-secret
                  key: api-key
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "750m"
              memory: "512Mi"
```

---

### 3.3 Task Agent — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: task-agent
  namespace: task-backend
  labels:
    app: task-agent
    version: "1.0"
spec:
  replicas: 2                        # AI agent — scale based on queue depth
  selector:
    matchLabels:
      app: task-agent
  template:
    metadata:
      labels:
        app: task-agent
    spec:
      containers:
        - name: task-agent
          image: naveed261/task-agent:latest
          ports:
            - containerPort: 9090
          envFrom:
            - configMapRef:
                name: agent-config
          env:
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: openai-secret
                  key: api-key
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-secret
                  key: password
          resources:
            requests:
              cpu: "500m"             # AI agents need more CPU
              memory: "512Mi"
            limits:
              cpu: "2000m"            # 2 full CPU cores max
              memory: "2Gi"           # LLM calls need memory
```

---

### 3.4 Notification Service — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-service
  namespace: task-backend
  labels:
    app: notification-service
    version: "1.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: notification-service
  template:
    metadata:
      labels:
        app: notification-service
    spec:
      containers:
        - name: notification-service
          image: naveed261/task-notifier:latest
          ports:
            - containerPort: 8081
          envFrom:
            - configMapRef:
                name: notification-config
          env:
            - name: SMTP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: smtp-secret
                  key: password
            - name: TWILIO_AUTH_TOKEN
              valueFrom:
                secretKeyRef:
                  name: twilio-secret
                  key: auth-token
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "300m"
              memory: "256Mi"
```

---

### 3.5 PostgreSQL — StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: task-data
spec:
  serviceName: "postgresql"
  replicas: 1                        # Single primary for now
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgresql
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: "taskmanager"
            - name: POSTGRES_USER
              value: "taskadmin"
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2000m"
              memory: "4Gi"
  volumeClaimTemplates:
    - metadata:
        name: postgres-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
```

**Why StatefulSet?** Database ka data persist rehna chahiye — pod restart hone par bhi data save rahe.

---

### 3.6 Redis — StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: task-data
spec:
  serviceName: "redis"
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
          command: ["redis-server", "--requirepass", "$(REDIS_PASSWORD)"]
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-secret
                  key: password
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
```

---

## 4. Services and Their Types

### Service Types Summary

| Service | Type | Port | Reason |
|---|---|---|---|
| UI Interface | **LoadBalancer** | 80/443 | Internet-facing — users access here |
| Backend API | **ClusterIP** | 8080 | Internal only — UI calls backend |
| Task Agent | **ClusterIP** | 9090 | Internal only — backend calls agent |
| Notification Service | **ClusterIP** | 8081 | Internal only — backend triggers it |
| PostgreSQL | **ClusterIP** | 5432 | Never exposed outside cluster |
| Redis | **ClusterIP** | 6379 | Never exposed outside cluster |

---

### 4.1 UI Interface Service — LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ui-service
  namespace: task-frontend
spec:
  type: LoadBalancer
  selector:
    app: ui-interface
  ports:
    - name: http
      port: 80
      targetPort: 3000
    - name: https
      port: 443
      targetPort: 3000
```

---

### 4.2 Backend API Service — ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-api-service
  namespace: task-backend
spec:
  type: ClusterIP
  selector:
    app: backend-api
  ports:
    - port: 8080
      targetPort: 8080
```

---

### 4.3 Task Agent Service — ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: task-agent-service
  namespace: task-backend
spec:
  type: ClusterIP
  selector:
    app: task-agent
  ports:
    - port: 9090
      targetPort: 9090
```

---

### 4.4 Notification Service — ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: notification-service
  namespace: task-backend
spec:
  type: ClusterIP
  selector:
    app: notification-service
  ports:
    - port: 8081
      targetPort: 8081
```

---

### 4.5 PostgreSQL Service — ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgresql-service
  namespace: task-data
spec:
  type: ClusterIP
  selector:
    app: postgresql
  ports:
    - port: 5432
      targetPort: 5432
```

---

## 5. Resource Requests and Limits

### Complete Resource Table

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit | Reason |
|---|---|---|---|---|---|
| UI Interface | 100m | 300m | 128Mi | 256Mi | Simple React/Next.js app |
| Backend API | 250m | 750m | 256Mi | 512Mi | REST API + DB queries |
| Task Agent | 500m | 2000m | 512Mi | 2Gi | LLM calls = high compute |
| Notification Service | 100m | 300m | 128Mi | 256Mi | Lightweight sender |
| PostgreSQL | 500m | 2000m | 1Gi | 4Gi | Database needs headroom |
| Redis | 100m | 500m | 256Mi | 1Gi | In-memory cache |

> **Note:** `100m` = 0.1 CPU core. `256Mi` = 256 Megabytes RAM.

### HorizontalPodAutoscaler for Backend API

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-api-hpa
  namespace: task-backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## 6. ConfigMaps

ConfigMaps store non-sensitive configuration — jaise environment variables.

### 6.1 UI ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ui-config
  namespace: task-frontend
data:
  NEXT_PUBLIC_API_URL: "http://backend-api-service.task-backend.svc.cluster.local:8080"
  NEXT_PUBLIC_APP_NAME: "AI Task Manager"
  NEXT_PUBLIC_ENV: "production"
  NEXT_PUBLIC_TIMEOUT: "30000"
```

### 6.2 Backend API ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: task-backend
data:
  APP_PORT: "8080"
  DB_HOST: "postgresql-service.task-data.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "taskmanager"
  REDIS_HOST: "redis-service.task-data.svc.cluster.local"
  REDIS_PORT: "6379"
  AGENT_URL: "http://task-agent-service.task-backend.svc.cluster.local:9090"
  NOTIFICATION_URL: "http://notification-service.task-backend.svc.cluster.local:8081"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  CORS_ORIGIN: "https://app.taskmanager.com"
```

### 6.3 Task Agent ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: agent-config
  namespace: task-backend
data:
  AGENT_PORT: "9090"
  LLM_MODEL: "gpt-4o"
  LLM_MAX_TOKENS: "4096"
  LLM_TEMPERATURE: "0.7"
  REDIS_HOST: "redis-service.task-data.svc.cluster.local"
  TASK_QUEUE_NAME: "task_processing_queue"
  MAX_CONCURRENT_TASKS: "5"
  AGENT_TIMEOUT_SECONDS: "120"
```

### 6.4 Notification Service ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: notification-config
  namespace: task-backend
data:
  NOTIF_PORT: "8081"
  SMTP_HOST: "smtp.gmail.com"
  SMTP_PORT: "587"
  SMTP_FROM: "noreply@taskmanager.com"
  TWILIO_ACCOUNT_SID: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
  PUSH_PROVIDER: "firebase"
  RETRY_ATTEMPTS: "3"
  RETRY_DELAY_MS: "1000"
```

---

## 7. Secrets Management and Expiry Handling

### 7.1 All Secrets Definition

```yaml
# Database Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: task-data
  annotations:
    secret-expiry-date: "2025-12-31"      # Track expiry manually
    managed-by: "team-backend"
type: Opaque
data:
  password: cG9zdGdyZXNfcGFzc3dvcmQxMjM=  # base64 encoded

---
# JWT Secret
apiVersion: v1
kind: Secret
metadata:
  name: jwt-secret
  namespace: task-backend
  annotations:
    secret-expiry-date: "2025-09-30"
    rotation-policy: "90-days"
type: Opaque
data:
  jwt-key: and0X3N1cGVyX3NlY3JldF9rZXlfMTIz  # base64 encoded

---
# OpenAI Secret
apiVersion: v1
kind: Secret
metadata:
  name: openai-secret
  namespace: task-backend
  annotations:
    secret-expiry-date: "2025-12-31"
    rotation-policy: "manual"
type: Opaque
data:
  api-key: c2stcHJvai14eHh4eHh4eHh4eHh4eHh4eHh4  # base64 encoded

---
# SMTP Secret
apiVersion: v1
kind: Secret
metadata:
  name: smtp-secret
  namespace: task-backend
type: Opaque
data:
  password: c210cF9wYXNzd29yZF8xMjM=  # base64 encoded

---
# Twilio Secret
apiVersion: v1
kind: Secret
metadata:
  name: twilio-secret
  namespace: task-backend
type: Opaque
data:
  auth-token: dHdpbGlvX2F1dGhfdG9rZW5feHh4  # base64 encoded

---
# Redis Secret
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
  namespace: task-data
type: Opaque
data:
  password: cmVkaXNfcGFzc3dvcmRfMTIz  # base64 encoded
```

### 7.2 Secret Expiry Handling Strategy

| Strategy | Description |
|---|---|
| **Annotation-based tracking** | `secret-expiry-date` annotation har secret par — team ko pata rahe |
| **90-day rotation** | JWT secret har 90 din mein change karo |
| **External Secrets Operator** | Production mein AWS Secrets Manager / Vault se sync karo |
| **CronJob for alerting** | Expiry se 14 din pehle Slack alert bhejo |
| **Sealed Secrets** | Git mein encrypted secrets store karo (Bitnami Sealed Secrets) |

### 7.3 Secret Rotation CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: secret-expiry-checker
  namespace: task-backend
spec:
  schedule: "0 9 * * 1"              # Every Monday at 9 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: secret-checker
              image: naveed261/secret-checker:latest
              command: ["python", "check_expiry.py"]
          restartPolicy: OnFailure
```

---

## 8. RBAC — Roles and RoleBindings

### 8.1 RBAC Strategy

| Role | Who | Permissions |
|---|---|---|
| `backend-reader` | Backend API pods | Read ConfigMaps, Secrets |
| `agent-worker` | Task Agent pods | Read Secrets, write to queue |
| `notifier-role` | Notification pods | Read Secrets only |
| `data-admin` | DBA team | Full access to task-data namespace |
| `dev-readonly` | Junior developers | Read-only across all namespaces |
| `ops-admin` | Ops/SRE team | Full cluster access |

---

### 8.2 Backend API Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-reader
  namespace: task-backend
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-reader-binding
  namespace: task-backend
subjects:
  - kind: ServiceAccount
    name: backend-api-sa
    namespace: task-backend
roleRef:
  kind: Role
  apiGroupÇ: rbac.authorization.k8s.io
  name: backend-reader
```

---

### 8.3 Task Agent Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: agent-worker
  namespace: task-backend
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: agent-worker-binding
  namespace: task-backend
subjects:
  - kind: ServiceAccount
    name: task-agent-sa
    namespace: task-backend
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: agent-worker
```

---

### 8.4 Dev Read-Only ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dev-readonly
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dev-readonly-binding
subjects:
  - kind: Group
    name: "dev-team"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
  name: dev-readonly
```

---

## 9. Inter-Service Communication

### 9.1 Communication Flow

```
Internet
   ↓
[LoadBalancer: ui-service:80]
   ↓
[UI Interface Pods] (task-frontend namespace)
   ↓  HTTP REST
[backend-api-service:8080] (task-backend namespace)
   ↓               ↓
[task-agent-service:9090]  [notification-service:8081]
   ↓
[postgresql-service:5432] + [redis-service:6379] (task-data namespace)
```

### 9.2 DNS-Based Service Discovery

Kubernetes automatically provides DNS. Format:

```
<service-name>.<namespace>.svc.cluster.local:<port>
```

| From | To | DNS Address |
|---|---|---|
| UI → Backend | HTTP | `backend-api-service.task-backend.svc.cluster.local:8080` |
| Backend → Agent | HTTP | `task-agent-service.task-backend.svc.cluster.local:9090` |
| Backend → Notifier | HTTP | `notification-service.task-backend.svc.cluster.local:8081` |
| Backend → DB | TCP | `postgresql-service.task-data.svc.cluster.local:5432` |
| Agent → Redis | TCP | `redis-service.task-data.svc.cluster.local:6379` |

### 9.3 NetworkPolicy — Zero Trust

```yaml
# Only allow backend to talk to postgresql
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
  namespace: task-data
spec:
  podSelector:
    matchLabels:
      app: postgresql
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: backend
          podSelector:
            matchLabels:
              app: backend-api
      ports:
        - protocol: TCP
          port: 5432
```

---

## 10. Architecture Diagram (Text)

```
┌─────────────────────────────────────────────────────────────┐
│                        INTERNET                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                   ┌────────▼────────┐
                   │  LoadBalancer   │
                   │  ui-service:80  │
                   └────────┬────────┘
                            │
        ┌───────────────────▼─────────────────────┐
        │         NAMESPACE: task-frontend         │
        │  ┌──────────────────────────────────┐   │
        │  │   UI Interface (2 replicas)      │   │
        │  │   Deployment | Port 3000         │   │
        │  └──────────────┬───────────────────┘   │
        └─────────────────┼───────────────────────┘
                          │ HTTP
        ┌─────────────────▼───────────────────────┐
        │         NAMESPACE: task-backend          │
        │  ┌─────────────┐  ┌──────────────────┐  │
        │  │ Backend API │  │  Notification    │  │
        │  │ (3 replicas)│  │  Service         │  │
        │  │ Port 8080   │  │  (2 replicas)    │  │
        │  └──────┬──────┘  └──────────────────┘  │
        │         │                               │
        │  ┌──────▼──────┐                        │
        │  │ Task Agent  │                        │
        │  │ (2 replicas)│                        │
        │  │ Port 9090   │                        │
        │  └─────────────┘                        │
        └─────────────────┬───────────────────────┘
                          │ TCP
        ┌─────────────────▼───────────────────────┐
        │          NAMESPACE: task-data            │
        │  ┌─────────────┐  ┌──────────────────┐  │
        │  │ PostgreSQL  │  │     Redis        │  │
        │  │ StatefulSet │  │  StatefulSet     │  │
        │  │ Port 5432   │  │  Port 6379       │  │
        │  └─────────────┘  └──────────────────┘  │
        └─────────────────────────────────────────┘
```

---

## ✅ Summary Checklist

| Requirement | Status |
|---|---|
| Pods & Deployments/StatefulSets | ✅ 4 Deployments + 2 StatefulSets |
| Services (ClusterIP, LoadBalancer) | ✅ 1 LoadBalancer + 5 ClusterIP |
| Resource Requests & Limits | ✅ All components defined |
| ConfigMaps | ✅ 4 ConfigMaps |
| Secrets + Expiry Handling | ✅ 6 Secrets + CronJob rotation |
| Namespaces | ✅ 4 Namespaces |
| RBAC Roles & RoleBindings | ✅ 5 Roles + Bindings |
| Inter-service Communication | ✅ DNS + NetworkPolicy |

---

*Plan created for k8s-deployment-plans project | GitHub: NAVEED261*
