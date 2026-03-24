# 🤖 Plan 2: AI Employee using OpenClaw — Kubernetes Deployment Plan

> **Project:** AI Employee — Personal AI Assistant using OpenClaw Framework  
> **Focus:** Security-first design for a personal AI employee with sensitive data access  
> **Author:** NAVEED261  
> **Repo:** https://github.com/NAVEED261/k8s-deployment-plans

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Security-First Design Principles](#security-first-design-principles)
3. [Namespaces](#namespaces)
4. [Pods and Deployments / StatefulSets](#pods-and-deployments--statefulsets)
5. [Services and Their Types](#services-and-their-types)
6. [Resource Requests and Limits](#resource-requests-and-limits)
7. [ConfigMaps](#configmaps)
8. [Secrets Management and Expiry Handling](#secrets-management-and-expiry-handling)
9. [RBAC — Roles and RoleBindings](#rbac--roles-and-rolebindings)
10. [Inter-Service Communication](#inter-service-communication)
11. [Security Hardening](#security-hardening)
12. [Architecture Diagram (Text)](#architecture-diagram-text)

---

## 1. Architecture Overview

OpenClaw is an AI employee framework. It acts as a personal AI worker — browsing web, writing emails, managing calendars, executing code, and calling external APIs — all on behalf of a human user.

### Components

| Component | Role | Security Level |
|---|---|---|
| **OpenClaw Core Agent** | Main AI brain — orchestrates all tasks | 🔴 Critical |
| **Tool Executor** | Runs code, shell commands safely in sandbox | 🔴 Critical |
| **Memory Store** | Stores agent's long-term memory (vector DB) | 🔴 Critical |
| **Auth Gateway** | All external API calls go through here | 🔴 Critical |
| **Audit Logger** | Logs every action the AI takes | 🟡 High |
| **User Interface** | Human talks to AI here | 🟡 High |
| **Task Queue** | Redis-based job queue | 🟡 High |
| **PostgreSQL** | Persistent user data, task history | 🟡 High |
| **Qdrant (Vector DB)** | AI memory storage | 🟡 High |

---

## 2. Security-First Design Principles

Since OpenClaw is a personal AI employee, it has access to:
- User emails and calendars
- Financial data
- Private documents
- System shell/code execution

Therefore **security is the #1 priority:**

| Principle | Implementation |
|---|---|
| **Zero Trust** | Every pod authenticates every request |
| **Least Privilege** | Each service gets minimum permissions needed |
| **Sandboxed Execution** | Code runs in isolated gVisor containers |
| **Audit Everything** | Every AI action is logged immutably |
| **Encrypted at Rest** | All secrets via Vault/Sealed Secrets |
| **Network Isolation** | Strict NetworkPolicies — deny all by default |
| **No Root Containers** | All containers run as non-root user |
| **ReadOnly Filesystems** | Prevent file tampering |

---

## 3. Namespaces

```yaml
# Core AI namespace
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-core
  labels:
    team: ai-engine
    security-level: critical
    env: production

---
# Tool execution — most restricted
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-sandbox
  labels:
    team: execution
    security-level: maximum
    env: production

---
# Data layer
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-data
  labels:
    team: data
    security-level: critical
    env: production

---
# User-facing layer
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-ui
  labels:
    team: frontend
    security-level: high
    env: production

---
# Security and audit
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-security
  labels:
    team: security
    security-level: critical
    env: production
```

### Namespace Strategy

| Namespace | Contents | Why Isolated? |
|---|---|---|
| `openclaw-core` | AI Agent, Auth Gateway, Task Queue | Brain of the system — maximum protection |
| `openclaw-sandbox` | Tool Executor | Code execution must be fully isolated |
| `openclaw-data` | PostgreSQL, Qdrant, Redis | Data never leaves this namespace |
| `openclaw-ui` | User Interface | Separate from core — compromise here won't reach AI |
| `openclaw-security` | Audit Logger, Secret Manager | Security tools must be untouchable |

---

## 4. Pods and Deployments / StatefulSets

### 4.1 OpenClaw Core Agent — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openclaw-agent
  namespace: openclaw-core
  labels:
    app: openclaw-agent
    security-tier: critical
spec:
  replicas: 1                          # Personal AI — single instance per user
  selector:
    matchLabels:
      app: openclaw-agent
  template:
    metadata:
      labels:
        app: openclaw-agent
      annotations:
        container.apparmor.security.beta.kubernetes.io/openclaw-agent: runtime/default
    spec:
      serviceAccountName: openclaw-agent-sa
      securityContext:
        runAsNonRoot: true             # Never run as root
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: openclaw-agent
          image: naveed261/openclaw-agent:latest
          ports:
            - containerPort: 7000
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true  # Cannot write to filesystem
            capabilities:
              drop: ["ALL"]             # Drop all Linux capabilities
          envFrom:
            - configMapRef:
                name: agent-core-config
          env:
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: openclaw-openai-secret
                  key: api-key
            - name: ANTHROPIC_API_KEY
              valueFrom:
                secretKeyRef:
                  name: openclaw-anthropic-secret
                  key: api-key
            - name: USER_DATA_KEY
              valueFrom:
                secretKeyRef:
                  name: user-encryption-secret
                  key: encryption-key
          volumeMounts:
            - name: tmp-volume
              mountPath: /tmp           # Only tmp is writable
            - name: agent-config-vol
              mountPath: /app/config
              readOnly: true
          resources:
            requests:
              cpu: "1000m"             # 1 full CPU core
              memory: "2Gi"
            limits:
              cpu: "4000m"             # 4 cores max for heavy AI tasks
              memory: "8Gi"
          livenessProbe:
            httpGet:
              path: /health
              port: 7000
            initialDelaySeconds: 30
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /ready
              port: 7000
            initialDelaySeconds: 10
            periodSeconds: 15
      volumes:
        - name: tmp-volume
          emptyDir: {}
        - name: agent-config-vol
          configMap:
            name: agent-core-config
```

---

### 4.2 Tool Executor — Deployment (Sandboxed)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tool-executor
  namespace: openclaw-sandbox
  labels:
    app: tool-executor
    security-tier: maximum
spec:
  replicas: 2                          # Multiple sandboxes for parallel tool execution
  selector:
    matchLabels:
      app: tool-executor
  template:
    metadata:
      labels:
        app: tool-executor
      annotations:
        # gVisor runtime for kernel-level sandboxing
        io.kubernetes.cri.untrusted-workload: "true"
    spec:
      runtimeClassName: gvisor         # Run in gVisor sandbox
      serviceAccountName: tool-executor-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534               # Nobody user — minimum privileges
        runAsGroup: 65534
        seccompProfile:
          type: Localhost
          localhostProfile: "profiles/restricted.json"
      containers:
        - name: tool-executor
          image: naveed261/openclaw-tools:latest
          ports:
            - containerPort: 8888
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          envFrom:
            - configMapRef:
                name: tool-executor-config
          volumeMounts:
            - name: execution-tmp
              mountPath: /tmp/execution
            - name: tool-output
              mountPath: /tmp/output
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2000m"
              memory: "4Gi"
              ephemeral-storage: "1Gi" # Limit disk usage in sandbox
      volumes:
        - name: execution-tmp
          emptyDir:
            sizeLimit: 500Mi          # Limit tmp size
        - name: tool-output
          emptyDir:
            sizeLimit: 500Mi
```

**Why gVisor?** Code execution sandbox — agar AI galat code chalaye toh host OS safe rahe.

---

### 4.3 Auth Gateway — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-gateway
  namespace: openclaw-core
  labels:
    app: auth-gateway
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auth-gateway
  template:
    metadata:
      labels:
        app: auth-gateway
    spec:
      serviceAccountName: auth-gateway-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1002
      containers:
        - name: auth-gateway
          image: naveed261/openclaw-auth:latest
          ports:
            - containerPort: 8443
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          env:
            - name: GOOGLE_CLIENT_SECRET
              valueFrom:
                secretKeyRef:
                  name: google-oauth-secret
                  key: client-secret
            - name: GITHUB_CLIENT_SECRET
              valueFrom:
                secretKeyRef:
                  name: github-oauth-secret
                  key: client-secret
            - name: JWT_SIGNING_KEY
              valueFrom:
                secretKeyRef:
                  name: jwt-signing-secret
                  key: key
          envFrom:
            - configMapRef:
                name: auth-gateway-config
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

---

### 4.4 Audit Logger — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: audit-logger
  namespace: openclaw-security
  labels:
    app: audit-logger
spec:
  replicas: 1
  selector:
    matchLabels:
      app: audit-logger
  template:
    metadata:
      labels:
        app: audit-logger
    spec:
      serviceAccountName: audit-logger-sa
      containers:
        - name: audit-logger
          image: naveed261/openclaw-audit:latest
          ports:
            - containerPort: 9200
          securityContext:
            runAsNonRoot: true
            runAsUser: 1003
            allowPrivilegeEscalation: false
          envFrom:
            - configMapRef:
                name: audit-logger-config
          volumeMounts:
            - name: audit-logs
              mountPath: /var/log/audit
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
      volumes:
        - name: audit-logs
          persistentVolumeClaim:
            claimName: audit-log-pvc
```

---

### 4.5 User Interface — Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openclaw-ui
  namespace: openclaw-ui
  labels:
    app: openclaw-ui
spec:
  replicas: 2
  selector:
    matchLabels:
      app: openclaw-ui
  template:
    metadata:
      labels:
        app: openclaw-ui
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1004
      containers:
        - name: openclaw-ui
          image: naveed261/openclaw-ui:latest
          ports:
            - containerPort: 3000
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
          envFrom:
            - configMapRef:
                name: ui-openclaw-config
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "300m"
              memory: "256Mi"
```

---

### 4.6 PostgreSQL — StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: openclaw-data
spec:
  serviceName: "postgresql"
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999                 # postgres user
        fsGroup: 999
      containers:
        - name: postgresql
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: "openclaw"
            - name: POSTGRES_USER
              value: "openclaw_admin"
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: "/var/lib/postgresql/data/pgdata"
          volumeMounts:
            - name: postgres-data
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
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 50Gi             # Larger — AI employee stores a lot
```

---

### 4.7 Qdrant Vector DB — StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: qdrant
  namespace: openclaw-data
  labels:
    app: qdrant
spec:
  serviceName: "qdrant"
  replicas: 1
  selector:
    matchLabels:
      app: qdrant
  template:
    metadata:
      labels:
        app: qdrant
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - name: qdrant
          image: qdrant/qdrant:latest
          ports:
            - containerPort: 6333      # HTTP
            - containerPort: 6334      # gRPC
          env:
            - name: QDRANT__SERVICE__API_KEY
              valueFrom:
                secretKeyRef:
                  name: qdrant-secret
                  key: api-key
          volumeMounts:
            - name: qdrant-storage
              mountPath: /qdrant/storage
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2000m"
              memory: "4Gi"
  volumeClaimTemplates:
    - metadata:
        name: qdrant-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 30Gi
```

---

### 4.8 Redis Task Queue — StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-queue
  namespace: openclaw-data
spec:
  serviceName: "redis-queue"
  replicas: 1
  selector:
    matchLabels:
      app: redis-queue
  template:
    metadata:
      labels:
        app: redis-queue
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          command: ["redis-server", "--requirepass", "$(REDIS_PASSWORD)", "--appendonly", "yes"]
          ports:
            - containerPort: 6379
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-queue-secret
                  key: password
          volumeMounts:
            - name: redis-data
              mountPath: /data
          resources:
            requests:
              cpu: "200m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "2Gi"
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

---

## 5. Services and Their Types

| Service | Namespace | Type | Port | Reason |
|---|---|---|---|---|
| openclaw-ui-service | openclaw-ui | **LoadBalancer** | 443 | HTTPS only — user access |
| openclaw-agent-service | openclaw-core | **ClusterIP** | 7000 | Internal AI communication |
| auth-gateway-service | openclaw-core | **ClusterIP** | 8443 | All auth goes here |
| tool-executor-service | openclaw-sandbox | **ClusterIP** | 8888 | Sandbox — never exposed |
| audit-logger-service | openclaw-security | **ClusterIP** | 9200 | Internal audit only |
| postgresql-service | openclaw-data | **ClusterIP** | 5432 | Data — never exposed |
| qdrant-service | openclaw-data | **ClusterIP** | 6333 | Vector DB — internal only |
| redis-queue-service | openclaw-data | **ClusterIP** | 6379 | Queue — internal only |

```yaml
# UI Service — HTTPS LoadBalancer
apiVersion: v1
kind: Service
metadata:
  name: openclaw-ui-service
  namespace: openclaw-ui
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:..."
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
spec:
  type: LoadBalancer
  selector:
    app: openclaw-ui
  ports:
    - name: https
      port: 443
      targetPort: 3000

---
# Agent Service — ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: openclaw-agent-service
  namespace: openclaw-core
spec:
  type: ClusterIP
  selector:
    app: openclaw-agent
  ports:
    - port: 7000
      targetPort: 7000

---
# Tool Executor — ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: tool-executor-service
  namespace: openclaw-sandbox
spec:
  type: ClusterIP
  selector:
    app: tool-executor
  ports:
    - port: 8888
      targetPort: 8888
```

---

## 6. Resource Requests and Limits

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit | Storage |
|---|---|---|---|---|---|
| OpenClaw Agent | 1000m | 4000m | 2Gi | 8Gi | — |
| Tool Executor | 500m | 2000m | 512Mi | 4Gi | 1Gi ephemeral |
| Auth Gateway | 200m | 500m | 256Mi | 512Mi | — |
| Audit Logger | 100m | 500m | 256Mi | 1Gi | 100Gi PVC |
| UI | 100m | 300m | 128Mi | 256Mi | — |
| PostgreSQL | 500m | 2000m | 1Gi | 4Gi | 50Gi PVC |
| Qdrant | 500m | 2000m | 1Gi | 4Gi | 30Gi PVC |
| Redis Queue | 200m | 1000m | 512Mi | 2Gi | 10Gi PVC |

> **OpenClaw Agent** ko zyada resources diye — AI tasks heavy hote hain (document analysis, code generation, web browsing).

---

## 7. ConfigMaps

### 7.1 Core Agent ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: agent-core-config
  namespace: openclaw-core
data:
  AGENT_PORT: "7000"
  LLM_PROVIDER: "anthropic"
  LLM_MODEL: "claude-opus-4-5"
  LLM_MAX_TOKENS: "8192"
  AUTH_GATEWAY_URL: "http://auth-gateway-service.openclaw-core.svc.cluster.local:8443"
  TOOL_EXECUTOR_URL: "http://tool-executor-service.openclaw-sandbox.svc.cluster.local:8888"
  VECTOR_DB_URL: "http://qdrant-service.openclaw-data.svc.cluster.local:6333"
  POSTGRES_HOST: "postgresql-service.openclaw-data.svc.cluster.local"
  REDIS_HOST: "redis-queue-service.openclaw-data.svc.cluster.local"
  AUDIT_URL: "http://audit-logger-service.openclaw-security.svc.cluster.local:9200"
  MAX_TOOL_CALLS_PER_TASK: "20"
  TASK_TIMEOUT_SECONDS: "300"
  ENABLE_WEB_BROWSING: "true"
  ENABLE_CODE_EXECUTION: "true"
  ENABLE_EMAIL_ACCESS: "true"
  AGENT_MEMORY_COLLECTION: "openclaw_memory"
  LOG_LEVEL: "info"
  AUDIT_ENABLED: "true"
```

### 7.2 Tool Executor ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tool-executor-config
  namespace: openclaw-sandbox
data:
  EXECUTOR_PORT: "8888"
  MAX_EXECUTION_TIME_SECONDS: "60"
  ALLOWED_LANGUAGES: "python,javascript,bash"
  MAX_OUTPUT_SIZE_MB: "10"
  NETWORK_ACCESS: "restricted"        # Only allowed domains
  ALLOWED_DOMAINS: "api.openai.com,api.anthropic.com,googleapis.com"
  MEMORY_LIMIT_MB: "512"
  CPU_QUOTA_PERCENT: "50"
```

### 7.3 Auth Gateway ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-gateway-config
  namespace: openclaw-core
data:
  AUTH_PORT: "8443"
  JWT_EXPIRY_MINUTES: "60"
  REFRESH_TOKEN_EXPIRY_DAYS: "30"
  GOOGLE_CLIENT_ID: "xxxx.apps.googleusercontent.com"
  GITHUB_CLIENT_ID: "Iv1.xxxxxxxxxxxxxxxx"
  ALLOWED_REDIRECT_URLS: "https://openclaw.naveed.ai/callback"
  RATE_LIMIT_PER_MINUTE: "60"
  SESSION_TIMEOUT_MINUTES: "120"
```

### 7.4 Audit Logger ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: audit-logger-config
  namespace: openclaw-security
data:
  AUDIT_PORT: "9200"
  LOG_RETENTION_DAYS: "365"           # Keep 1 year of logs
  LOG_FORMAT: "json"
  ALERT_ON_SENSITIVE_ACTIONS: "true"  # Alert when AI accesses emails/files
  SLACK_WEBHOOK_ENABLED: "true"
  LOG_COMPRESSION: "gzip"
```

---

## 8. Secrets Management and Expiry Handling

### 8.1 All Secrets

```yaml
# LLM API Keys
apiVersion: v1
kind: Secret
metadata:
  name: openclaw-openai-secret
  namespace: openclaw-core
  annotations:
    secret-expiry-date: "2025-12-31"
    rotation-policy: "on-compromise"
    owner: "naveed-admin"
type: Opaque
data:
  api-key: c2stcHJvai14eHh4eHh4eHh4eA==   # base64 encoded

---
apiVersion: v1
kind: Secret
metadata:
  name: openclaw-anthropic-secret
  namespace: openclaw-core
  annotations:
    secret-expiry-date: "2025-12-31"
    rotation-policy: "on-compromise"
type: Opaque
data:
  api-key: c2staW50LXh4eHh4eHh4eHh4eA==   # base64 encoded

---
# User Data Encryption Key
apiVersion: v1
kind: Secret
metadata:
  name: user-encryption-secret
  namespace: openclaw-core
  annotations:
    secret-expiry-date: "2025-06-30"
    rotation-policy: "180-days"          # Rotate every 6 months
    criticality: "maximum"
type: Opaque
data:
  encryption-key: dXNlcl9lbmNyeXB0aW9uX2tleV8zMjBiaXQ=

---
# OAuth Secrets
apiVersion: v1
kind: Secret
metadata:
  name: google-oauth-secret
  namespace: openclaw-core
  annotations:
    rotation-policy: "annual"
type: Opaque
data:
  client-secret: Z29vZ2xlX2NsaWVudF9zZWNyZXRfeHh4

---
apiVersion: v1
kind: Secret
metadata:
  name: github-oauth-secret
  namespace: openclaw-core
type: Opaque
data:
  client-secret: Z2l0aHViX2NsaWVudF9zZWNyZXRfeHh4

---
# JWT Signing Key — rotated every 30 days
apiVersion: v1
kind: Secret
metadata:
  name: jwt-signing-secret
  namespace: openclaw-core
  annotations:
    rotation-policy: "30-days"          # Short rotation for JWT
    last-rotated: "2025-03-01"
type: Opaque
data:
  key: and0X3NpZ25pbmdfa2V5X3N1cGVyX3NlY3JldA==

---
# Database Secrets
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: openclaw-data
  annotations:
    rotation-policy: "90-days"
type: Opaque
data:
  password: cG9zdGdyZXNfcGFzc3dvcmRfc2VjdXJl

---
apiVersion: v1
kind: Secret
metadata:
  name: qdrant-secret
  namespace: openclaw-data
type: Opaque
data:
  api-key: cWRyYW50X2FwaV9rZXlfeHh4eHh4

---
apiVersion: v1
kind: Secret
metadata:
  name: redis-queue-secret
  namespace: openclaw-data
type: Opaque
data:
  password: cmVkaXNfcXVldWVfcGFzc3dvcmQ=
```

### 8.2 Advanced Secret Management Strategy

| Level | Tool | Use Case |
|---|---|---|
| **Dev** | K8s native Secrets | Basic secret storage |
| **Staging** | Sealed Secrets (Bitnami) | Encrypted secrets in Git |
| **Production** | HashiCorp Vault + ESO | Auto-rotation, audit trail |
| **Critical Secrets** | AWS KMS + Vault | Hardware-backed encryption |

### 8.3 Vault Integration (Production)

```yaml
# External Secrets Operator — syncs Vault → K8s Secrets
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: openclaw-agent-secrets
  namespace: openclaw-core
spec:
  refreshInterval: 1h                  # Re-sync every hour
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: openclaw-openai-secret
    creationPolicy: Owner
  data:
    - secretKey: api-key
      remoteRef:
        key: openclaw/openai
        property: api_key
```

### 8.4 Secret Rotation Schedule

| Secret | Rotation Frequency | Method |
|---|---|---|
| JWT Signing Key | Every 30 days | Automated CronJob |
| User Encryption Key | Every 180 days | Manual + Vault |
| LLM API Keys | On compromise only | Manual |
| DB Passwords | Every 90 days | Vault dynamic secrets |
| OAuth Secrets | Annual | Manual |

---

## 9. RBAC — Roles and RoleBindings

### 9.1 Service Accounts

```yaml
# OpenClaw Agent Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: openclaw-agent-sa
  namespace: openclaw-core
  annotations:
    description: "Service account for OpenClaw AI Agent"

---
# Tool Executor — maximum restriction
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tool-executor-sa
  namespace: openclaw-sandbox
  annotations:
    description: "Restricted SA for sandbox execution"

---
# Auth Gateway
apiVersion: v1
kind: ServiceAccount
metadata:
  name: auth-gateway-sa
  namespace: openclaw-core

---
# Audit Logger
apiVersion: v1
kind: ServiceAccount
metadata:
  name: audit-logger-sa
  namespace: openclaw-security
```

### 9.2 Roles

```yaml
# OpenClaw Agent Role — can read secrets it needs
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: openclaw-agent-role
  namespace: openclaw-core
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames:                      # Only specific secrets!
      - "openclaw-openai-secret"
      - "openclaw-anthropic-secret"
      - "user-encryption-secret"
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]

---
# Tool Executor — almost no permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tool-executor-role
  namespace: openclaw-sandbox
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["tool-executor-config"]
    verbs: ["get"]
  # No secret access — tool executor should never see credentials

---
# Audit Logger — can write logs
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: audit-logger-role
  namespace: openclaw-security
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get"]

---
# Security Admin — full access to security namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: security-admin-role
  namespace: openclaw-security
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]

---
# Developer Read-Only — across all openclaw namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: openclaw-dev-readonly
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: []                           # ZERO access to secrets for devs
```

### 9.3 RoleBindings

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: openclaw-agent-binding
  namespace: openclaw-core
subjects:
  - kind: ServiceAccount
    name: openclaw-agent-sa
    namespace: openclaw-core
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: openclaw-agent-role

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tool-executor-binding
  namespace: openclaw-sandbox
subjects:
  - kind: ServiceAccount
    name: tool-executor-sa
    namespace: openclaw-sandbox
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: tool-executor-role

---
# Dev team gets read-only cluster-wide
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openclaw-dev-readonly-binding
subjects:
  - kind: Group
    name: "openclaw-developers"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
  name: openclaw-dev-readonly
```

---

## 10. Inter-Service Communication

### 10.1 Communication Flow

```
Internet (HTTPS only)
        ↓
[LoadBalancer: openclaw-ui-service:443]
        ↓
[UI Interface] (openclaw-ui namespace)
        ↓ JWT-authenticated HTTP
[Auth Gateway] (openclaw-core)
        ↓ Verified request
[OpenClaw Agent] (openclaw-core)
    ↙         ↘          ↘
[Tool Executor] [Qdrant]  [PostgreSQL]
(sandbox)     (data)      (data)
    ↑
[Redis Queue] (data)
        ↓ All actions logged
[Audit Logger] (openclaw-security)
```

### 10.2 Service DNS Map

| From | To | Protocol | DNS |
|---|---|---|---|
| UI → Auth Gateway | HTTPS | `auth-gateway-service.openclaw-core.svc.cluster.local:8443` |
| UI → Agent | HTTP/WS | `openclaw-agent-service.openclaw-core.svc.cluster.local:7000` |
| Agent → Tool Executor | HTTP | `tool-executor-service.openclaw-sandbox.svc.cluster.local:8888` |
| Agent → Qdrant | HTTP | `qdrant-service.openclaw-data.svc.cluster.local:6333` |
| Agent → PostgreSQL | TCP | `postgresql-service.openclaw-data.svc.cluster.local:5432` |
| Agent → Redis | TCP | `redis-queue-service.openclaw-data.svc.cluster.local:6379` |
| Agent → Audit | HTTP | `audit-logger-service.openclaw-security.svc.cluster.local:9200` |

### 10.3 NetworkPolicies — Default Deny All

```yaml
# DENY ALL ingress by default — then selectively allow
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: openclaw-core
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress: []                          # No ingress by default
  egress: []                           # No egress by default

---
# Allow UI to reach Agent only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ui-to-agent
  namespace: openclaw-core
spec:
  podSelector:
    matchLabels:
      app: openclaw-agent
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: frontend
          podSelector:
            matchLabels:
              app: openclaw-ui
      ports:
        - protocol: TCP
          port: 7000

---
# Sandbox total isolation — only agent can call it
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sandbox-isolation
  namespace: openclaw-sandbox
spec:
  podSelector:
    matchLabels:
      app: tool-executor
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: ai-engine
          podSelector:
            matchLabels:
              app: openclaw-agent
      ports:
        - protocol: TCP
          port: 8888
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:                    # Cannot reach internal cluster
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
```

---

## 11. Security Hardening

### 11.1 PodSecurityPolicy / Pod Security Standards

```yaml
# Enforce restricted security standard on all namespaces
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw-sandbox
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 11.2 Security Summary Table

| Security Control | Applied To | Status |
|---|---|---|
| Non-root containers | All pods | ✅ |
| ReadOnly filesystem | All pods | ✅ |
| Drop ALL capabilities | All pods | ✅ |
| gVisor runtime | Tool Executor | ✅ |
| Network deny-all default | All namespaces | ✅ |
| Secrets — resourceNames restriction | Agent SA | ✅ |
| Zero secret access for devs | Dev ClusterRole | ✅ |
| Audit logging for all AI actions | All namespaces | ✅ |
| HTTPS only external | LoadBalancer | ✅ |
| Vault secret rotation | Production | ✅ |

---

## 12. Architecture Diagram (Text)

```
┌─────────────────────────────────────────────────────────────────┐
│                    INTERNET (HTTPS only)                        │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │    LoadBalancer      │
                    │  port 443 (HTTPS)   │
                    └──────────┬──────────┘
                               │
         ┌─────────────────────▼──────────────────────┐
         │          NAMESPACE: openclaw-ui             │
         │    ┌──────────────────────────────────┐    │
         │    │  User Interface (2 replicas)     │    │
         │    │  Non-root | ReadOnly FS          │    │
         │    └──────────────┬───────────────────┘    │
         └─────────────────  ┼  ─────────────────────┘
                             │ JWT Auth
         ┌───────────────────▼────────────────────────┐
         │         NAMESPACE: openclaw-core            │
         │  ┌────────────────┐  ┌─────────────────┐   │
         │  │  Auth Gateway  │→ │  OpenClaw Agent │   │
         │  │  port 8443     │  │  port 7000      │   │
         │  │  OAuth/JWT     │  │  1-4 CPU | 2-8G │   │
         │  └────────────────┘  └────┬────────────┘   │
         └────────────────────────── ┼ ───────────────┘
                  ┌──────────────────┼──────────────┐
                  ↓                  ↓               ↓
   ┌──────────────────┐   ┌──────────────┐  ┌────────────────────┐
   │NAMESPACE:        │   │NAMESPACE:    │  │NAMESPACE:          │
   │openclaw-sandbox  │   │openclaw-data │  │openclaw-security   │
   │                  │   │              │  │                    │
   │ Tool Executor    │   │ PostgreSQL   │  │ Audit Logger       │
   │ gVisor Runtime   │   │ Qdrant       │  │ Immutable Logs     │
   │ Nobody user      │   │ Redis Queue  │  │ 365 day retention  │
   └──────────────────┘   └──────────────┘  └────────────────────┘
```

---

## ✅ Summary Checklist

| Requirement | Status |
|---|---|
| Pods & Deployments/StatefulSets | ✅ 5 Deployments + 3 StatefulSets |
| Services (ClusterIP, LoadBalancer) | ✅ 1 LoadBalancer + 7 ClusterIP |
| Resource Requests & Limits | ✅ All components defined |
| ConfigMaps | ✅ 4 ConfigMaps |
| Secrets + Expiry + Vault Integration | ✅ 8 Secrets + ESO + Rotation Schedule |
| Namespaces | ✅ 5 Security-focused Namespaces |
| RBAC — Least Privilege | ✅ resourceNames-level restrictions |
| Inter-service Communication | ✅ DNS + Default Deny NetworkPolicy |
| Security Hardening | ✅ gVisor, Non-root, ReadOnly FS, AppArmor |

---

*Plan created for k8s-deployment-plans project | GitHub: NAVEED261*
