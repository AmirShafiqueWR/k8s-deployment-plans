# Plan 1 — Komplio: AI Native Task Manager
## Kubernetes Deployment Plan

**Project:** Komplio — Multi-tenant SaaS Task Manager  
**Scenario:** AI Native Task Manager (UI Interface, Backend APIs, Task Agent, Notification Service)  
**Author:** Class Project Submission  
**Version:** 1.0

---

## Table of Contents

1. [Application Overview](#1-application-overview)
2. [Environments](#2-environments)
3. [Namespaces](#3-namespaces)
4. [Pods and Deployments / StatefulSets](#4-pods-and-deployments--statefulsets)
5. [Services and Their Types](#5-services-and-their-types)
6. [Resource Requests and Limits](#6-resource-requests-and-limits)
7. [ConfigMaps](#7-configmaps)
8. [Secrets Management and Expiry Handling](#8-secrets-management-and-expiry-handling)
9. [RBAC — Roles and RoleBindings](#9-rbac--roles-and-rolebindings)
10. [Inter-Service Communication](#10-inter-service-communication)
11. [Architecture Diagram](#11-architecture-diagram)

---

## 1. Application Overview

Komplio is a commercial multi-tenant SaaS task management platform. It maps directly to the four components required by Scenario 1:

| Scenario Component | Komplio Component | Technology |
|---|---|---|
| UI Interface | Jinja2 web frontend | Python / HTML / JS |
| Backend APIs | FastAPI REST API | Python FastAPI |
| Task Agent | Whisper AI transcription agent | OpenAI Whisper API |
| Notification Service | WhatsApp + Email notification service | Meta Cloud API + Gmail SMTP |

### Key architectural characteristics
- **Multi-tenant** — each organisation is a fully isolated tenant with Row Level Security in Supabase
- **AI-native** — voice notes transcribed by Whisper AI, tasks managed by an intelligent agent layer
- **Event-driven notifications** — task events trigger WhatsApp and in-app notifications automatically
- **Stateless API, stateful data** — FastAPI is stateless; Supabase holds all persistent state

---

## 2. Environments

### 2.1 Development — Docker Desktop with Kubernetes enabled

```
Platform:   Docker Desktop (Mac / Windows) — Kubernetes enabled via Settings → Kubernetes
Cluster:    Single-node local cluster (docker-desktop context)
Registry:   Local images built with docker build, loaded directly into Docker Desktop K8s
Secrets:    kubectl create secret — real K8s Secrets, not .env files
Namespace:  komplio-dev
Command:    kubectl config use-context docker-desktop
```

**Why Docker Desktop K8s and not Docker Compose:**  
Docker Desktop Kubernetes runs real K8s — the same `kubectl apply` commands, Secrets, ConfigMaps, RBAC, Namespaces, livenessProbes, and PersistentVolumeClaims work identically to production. The same YAML manifests used locally are applied in production without modification. This achieves true dev-prod parity and eliminates "works on my machine" failures.

### 2.2 Staging

```
Platform:   DigitalOcean Kubernetes (DOKS) — small cluster
Nodes:      2 × 2vCPU / 4GB RAM ($48/mo)
Registry:   Docker Hub or GitHub Container Registry
Namespace:  komplio-staging
```

### 2.3 Production

```
Platform:   DigitalOcean Kubernetes (DOKS)
Nodes:      3 × 2vCPU / 4GB RAM nodes ($72/mo)
Registry:   GitHub Container Registry (ghcr.io)
Namespace:  komplio-prod (platform) + komplio-tenant-{id} (per tenant)
LoadBalancer: DigitalOcean managed load balancer ($12/mo)
```

---

## 3. Namespaces

Namespaces provide isolation between environments and between tenants.

### 3.1 Platform namespaces

```yaml
# Platform-level namespaces
apiVersion: v1
kind: Namespace
metadata:
  name: komplio-prod
  labels:
    environment: production
    app: komplio
---
apiVersion: v1
kind: Namespace
metadata:
  name: komplio-dev
  labels:
    environment: development
    app: komplio
---
apiVersion: v1
kind: Namespace
metadata:
  name: komplio-staging
  labels:
    environment: staging
    app: komplio
---
# Monitoring namespace (shared)
apiVersion: v1
kind: Namespace
metadata:
  name: komplio-monitoring
  labels:
    purpose: observability
```

### 3.2 Tenant namespaces (multi-tenancy)

Each organisation (tenant) gets its own namespace for complete isolation:

```yaml
# Example: tenant namespace for Al-Shifa Hospital
apiVersion: v1
kind: Namespace
metadata:
  name: komplio-tenant-alshifa
  labels:
    app: komplio
    tenant-id: "alshifa-001"
    plan: professional
```

**Isolation guarantees per namespace:**
- Separate resource quotas — one tenant cannot starve another
- Separate RBAC — tenant Super Admin can only access their namespace
- Separate Secrets — WhatsApp tokens, API keys scoped per tenant
- NetworkPolicy — tenant pods cannot communicate with other tenant pods

### 3.3 Resource Quotas per tenant namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: komplio-tenant-alshifa
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: "2Gi"
    limits.cpu: "4"
    limits.memory: "4Gi"
    persistentvolumeclaims: "3"
```

---

## 4. Pods and Deployments / StatefulSets

### 4.1 Component overview

| Component | K8s Kind | Replicas | Reason |
|---|---|---|---|
| Frontend (UI) | Deployment | 2 | Stateless — can scale horizontally |
| Backend API (FastAPI) | Deployment | 2 | Stateless — horizontal scaling safe |
| Task Agent (Whisper) | Deployment | 1–3 | Stateless — scale with voice note volume |
| Notification Service | Deployment | 1 | Stateless — WhatsApp + email sender |
| Admin Panel | Deployment | 1 | Low traffic — single replica sufficient |
| Redis (session cache) | StatefulSet | 1 | Stateful — needs persistent identity |

> **Note on Supabase:** The PostgreSQL database and file storage run on Supabase cloud — they are NOT deployed in Kubernetes. Kubernetes pods connect to Supabase via a connection string stored as a Secret.

---

### 4.2 Frontend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: komplio-frontend
  namespace: komplio-prod
  labels:
    app: komplio
    component: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: komplio
      component: frontend
  template:
    metadata:
      labels:
        app: komplio
        component: frontend
    spec:
      serviceAccountName: komplio-frontend-sa
      containers:
        - name: frontend
          image: ghcr.io/komplio/frontend:v1.0.0
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: komplio-app-config
          env:
            - name: APP_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: komplio-app-secrets
                  key: app-secret-key
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /ready
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
```

---

### 4.3 Backend API Deployment (FastAPI)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: komplio-api
  namespace: komplio-prod
  labels:
    app: komplio
    component: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: komplio
      component: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0        # zero downtime — never take both pods down
      maxSurge: 1              # spin up 1 extra during update
  template:
    metadata:
      labels:
        app: komplio
        component: api
    spec:
      serviceAccountName: komplio-api-sa
      containers:
        - name: fastapi
          image: ghcr.io/komplio/api:v1.0.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: komplio-app-config
          env:
            - name: SUPABASE_URL
              valueFrom:
                secretKeyRef:
                  name: komplio-db-secrets
                  key: supabase-url
            - name: SUPABASE_KEY
              valueFrom:
                secretKeyRef:
                  name: komplio-db-secrets
                  key: supabase-key
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: komplio-app-secrets
                  key: jwt-secret
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /docs
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
```

---

### 4.4 Task Agent Deployment (Whisper AI)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: komplio-task-agent
  namespace: komplio-prod
  labels:
    app: komplio
    component: task-agent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: komplio
      component: task-agent
  template:
    metadata:
      labels:
        app: komplio
        component: task-agent
    spec:
      serviceAccountName: komplio-agent-sa
      containers:
        - name: whisper-agent
          image: ghcr.io/komplio/task-agent:v1.0.0
          ports:
            - containerPort: 8081
          env:
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: komplio-ai-secrets
                  key: openai-api-key
            - name: SUPABASE_URL
              valueFrom:
                secretKeyRef:
                  name: komplio-db-secrets
                  key: supabase-url
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2000m"
              memory: "2Gi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8081
            initialDelaySeconds: 20
            periodSeconds: 30
```

---

### 4.5 Notification Service Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: komplio-notification-service
  namespace: komplio-prod
  labels:
    app: komplio
    component: notification-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: komplio
      component: notification-service
  template:
    metadata:
      labels:
        app: komplio
        component: notification-service
    spec:
      serviceAccountName: komplio-notification-sa
      containers:
        - name: notification-service
          image: ghcr.io/komplio/notification-service:v1.0.0
          ports:
            - containerPort: 8082
          env:
            - name: WHATSAPP_ACCESS_TOKEN
              valueFrom:
                secretKeyRef:
                  name: komplio-notification-secrets
                  key: whatsapp-access-token
            - name: WHATSAPP_PHONE_NUMBER_ID
              valueFrom:
                secretKeyRef:
                  name: komplio-notification-secrets
                  key: whatsapp-phone-number-id
            - name: GMAIL_USER
              valueFrom:
                secretKeyRef:
                  name: komplio-notification-secrets
                  key: gmail-user
            - name: GMAIL_APP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: komplio-notification-secrets
                  key: gmail-app-password
          envFrom:
            - configMapRef:
                name: komplio-app-config
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8082
            initialDelaySeconds: 10
            periodSeconds: 30
```

---

### 4.6 Redis StatefulSet (session cache)

Redis is stateful — it must be deployed as a StatefulSet to maintain a stable network identity and persistent storage.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: komplio-redis
  namespace: komplio-prod
spec:
  serviceName: komplio-redis
  replicas: 1
  selector:
    matchLabels:
      app: komplio
      component: redis
  template:
    metadata:
      labels:
        app: komplio
        component: redis
    spec:
      serviceAccountName: komplio-redis-sa
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: komplio-redis-secrets
                  key: redis-password
          command: ["redis-server", "--requirepass", "$(REDIS_PASSWORD)"]
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          volumeMounts:
            - name: redis-data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

---

## 5. Services and Their Types

### 5.1 Service type selection rationale

| Service | Type | Reason |
|---|---|---|
| Frontend | LoadBalancer | Public internet access — users open app.komplio.com |
| Backend API | ClusterIP | Internal only — frontend calls it within the cluster |
| Task Agent | ClusterIP | Internal only — API calls Whisper agent internally |
| Notification Service | ClusterIP | Internal only — API triggers notifications internally |
| Admin Panel | NodePort | Restricted access — Komplio admin only, via VPN |
| Redis | ClusterIP | Internal only — cache accessed by API pods |

---

### 5.2 Frontend Service (LoadBalancer)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: komplio-frontend-svc
  namespace: komplio-prod
  annotations:
    service.beta.kubernetes.io/do-loadbalancer-name: "komplio-prod-lb"
    service.beta.kubernetes.io/do-loadbalancer-protocol: "https"
    service.beta.kubernetes.io/do-loadbalancer-tls-ports: "443"
spec:
  type: LoadBalancer
  selector:
    app: komplio
    component: frontend
  ports:
    - name: https
      port: 443
      targetPort: 8000
    - name: http
      port: 80
      targetPort: 8000
```

---

### 5.3 Backend API Service (ClusterIP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: komplio-api-svc
  namespace: komplio-prod
spec:
  type: ClusterIP
  selector:
    app: komplio
    component: api
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

---

### 5.4 Task Agent Service (ClusterIP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: komplio-task-agent-svc
  namespace: komplio-prod
spec:
  type: ClusterIP
  selector:
    app: komplio
    component: task-agent
  ports:
    - name: http
      port: 8081
      targetPort: 8081
```

---

### 5.5 Notification Service (ClusterIP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: komplio-notification-svc
  namespace: komplio-prod
spec:
  type: ClusterIP
  selector:
    app: komplio
    component: notification-service
  ports:
    - name: http
      port: 8082
      targetPort: 8082
```

---

### 5.6 Admin Panel Service (NodePort)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: komplio-admin-svc
  namespace: komplio-prod
spec:
  type: NodePort
  selector:
    app: komplio
    component: admin
  ports:
    - port: 9000
      targetPort: 9000
      nodePort: 30090   # access via VPN only — not exposed publicly
```

---

### 5.7 Redis Service (ClusterIP — headless for StatefulSet)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: komplio-redis
  namespace: komplio-prod
spec:
  type: ClusterIP
  clusterIP: None    # headless — required for StatefulSet stable DNS
  selector:
    app: komplio
    component: redis
  ports:
    - port: 6379
      targetPort: 6379
```

---

## 6. Resource Requests and Limits

Resource requests guarantee minimum resources. Limits prevent any one pod from consuming the entire node.

```
Rule: requests = what the pod needs to function correctly
      limits   = maximum it can consume (cap to protect neighbours)
```

### 6.1 Resource summary table

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---|---|---|---|---|
| Frontend | 100m | 500m | 128Mi | 256Mi |
| Backend API | 200m | 1000m | 256Mi | 512Mi |
| Task Agent (Whisper) | 500m | 2000m | 512Mi | 2Gi |
| Notification Service | 100m | 500m | 128Mi | 256Mi |
| Admin Panel | 100m | 300m | 128Mi | 256Mi |
| Redis | 100m | 500m | 128Mi | 256Mi |

> **Note:** Task Agent has the highest limits because Whisper AI transcription is CPU and memory intensive. A 30-second voice note transcription can spike to 1.5–2 CPU cores briefly.

### 6.2 Horizontal Pod Autoscaler — Backend API

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: komplio-api-hpa
  namespace: komplio-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: komplio-api
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## 7. ConfigMaps

ConfigMaps store non-sensitive configuration. Any value that is not a secret — URLs, feature flags, timeouts, plan limits — goes in a ConfigMap.

### 7.1 Application ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: komplio-app-config
  namespace: komplio-prod
data:
  # Application URLs
  APP_URL: "https://app.komplio.com"
  ADMIN_URL: "https://admin.komplio.com"
  API_BASE_URL: "http://komplio-api-svc:8080"
  TASK_AGENT_URL: "http://komplio-task-agent-svc:8081"
  NOTIFICATION_SERVICE_URL: "http://komplio-notification-svc:8082"

  # JWT Configuration (non-secret values only)
  JWT_EXPIRY_HOURS: "24"
  JWT_ALGORITHM: "HS256"

  # Feature flags
  VOICE_NOTES_ENABLED: "true"
  WHATSAPP_ENABLED: "true"
  EMAIL_FALLBACK_ENABLED: "true"
  AUDIT_LOG_ENABLED: "true"

  # Plan limits
  STARTER_MAX_USERS: "5"
  PROFESSIONAL_MAX_USERS: "15"
  BUSINESS_MAX_USERS: "100"

  # File upload limits
  MAX_FILE_SIZE_MB: "50"
  MAX_VIDEO_SIZE_MB: "50"
  ALLOWED_FILE_TYPES: "jpg,png,pdf,docx,xlsx,mp4,mov,m4a,mp3"

  # Whisper configuration
  WHISPER_MODEL: "whisper-1"
  WHISPER_LANGUAGE: "ur"           # Urdu primary
  WHISPER_FALLBACK_LANGUAGE: "en"  # English fallback

  # Notification settings
  OVERDUE_CHECK_CRON: "0 9 * * *"  # Daily 9 AM digest
  WA_MAX_RETRIES: "3"
  WA_RETRY_DELAY_SECONDS: "30"

  # Environment
  ENVIRONMENT: "production"
  LOG_LEVEL: "INFO"
```

### 7.2 Notification templates ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: komplio-notification-templates
  namespace: komplio-prod
data:
  task_assigned: |
    Assalamu Alaikum {name} 👋
    New task assigned by {assigner}:
    📋 {task_title}
    📅 Due: {due_date}
    🔴 Priority: {priority}
    Update status: {task_url}

  task_overdue: |
    ⚠️ Task Overdue — {task_title}
    Assigned to: {assignee}
    Was due: {due_date}
    Please update status immediately: {task_url}

  task_completed: |
    ✅ Task Completed
    {assignee} completed: {task_title}
    Completed at: {completed_at}
    View task: {task_url}
```

---

## 8. Secrets Management and Expiry Handling

### 8.1 Secrets inventory

| Secret Name | Keys | Rotation Period | Sensitivity |
|---|---|---|---|
| `komplio-db-secrets` | supabase-url, supabase-key, supabase-service-key | 90 days | Critical |
| `komplio-app-secrets` | jwt-secret, app-secret-key, admin-master-key | 90 days | Critical |
| `komplio-notification-secrets` | whatsapp-access-token, whatsapp-phone-number-id, gmail-user, gmail-app-password | 60 days | High |
| `komplio-ai-secrets` | openai-api-key | 90 days | High |
| `komplio-redis-secrets` | redis-password | 180 days | Medium |

### 8.2 Creating secrets

```bash
# Database secrets
kubectl create secret generic komplio-db-secrets \
  --namespace=komplio-prod \
  --from-literal=supabase-url='https://xxx.supabase.co' \
  --from-literal=supabase-key='your-anon-key' \
  --from-literal=supabase-service-key='your-service-role-key'

# Application secrets
kubectl create secret generic komplio-app-secrets \
  --namespace=komplio-prod \
  --from-literal=jwt-secret='your-256-bit-secret' \
  --from-literal=app-secret-key='your-app-secret' \
  --from-literal=admin-master-key='your-master-key'

# Notification secrets
kubectl create secret generic komplio-notification-secrets \
  --namespace=komplio-prod \
  --from-literal=whatsapp-access-token='your-wa-token' \
  --from-literal=whatsapp-phone-number-id='your-phone-id' \
  --from-literal=gmail-user='your@gmail.com' \
  --from-literal=gmail-app-password='your-app-password'

# AI secrets
kubectl create secret generic komplio-ai-secrets \
  --namespace=komplio-prod \
  --from-literal=openai-api-key='sk-your-key'
```

### 8.3 Secret expiry and rotation strategy

Kubernetes Secrets do not natively expire. Expiry must be enforced through process and tooling.

**Approach 1 — Manual rotation (recommended for launch):**

```yaml
# Label every secret with creation date and expiry date
apiVersion: v1
kind: Secret
metadata:
  name: komplio-notification-secrets
  namespace: komplio-prod
  annotations:
    komplio.io/created: "2026-04-03"
    komplio.io/expires: "2026-07-03"      # 90 days
    komplio.io/rotation-owner: "devops-team"
    komplio.io/last-rotated: "2026-04-03"
type: Opaque
data:
  whatsapp-access-token: <base64-encoded-value>
```

**Rotation process:**
```
1. 2 weeks before expiry → alert sent to DevOps team via email/Slack
2. DevOps generates new token from Meta WhatsApp Business Portal
3. kubectl create secret generic komplio-notification-secrets --dry-run=client -o yaml | kubectl apply -f -
4. kubectl rollout restart deployment/komplio-notification-service
5. Verify notifications still working
6. Update annotation: komplio.io/last-rotated and komplio.io/expires
```

**Approach 2 — Automated rotation with External Secrets Operator (recommended for scale):**

```yaml
# ExternalSecret pulls fresh values from AWS Secrets Manager automatically
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: komplio-notification-secrets
  namespace: komplio-prod
spec:
  refreshInterval: 1h           # check for updates every hour
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: komplio-notification-secrets
    creationPolicy: Owner
  data:
    - secretKey: whatsapp-access-token
      remoteRef:
        key: komplio/prod/whatsapp
        property: access_token
```

**What happens when a secret expires:**

| Scenario | Impact | Resolution |
|---|---|---|
| WhatsApp token expires | Notifications silently fail, no WA messages sent | Rotate token in Meta portal → update K8s secret → rolling restart |
| Supabase key expires | All API calls return 401, app unusable | Rotate in Supabase dashboard → update secret → rolling restart |
| OpenAI key expires | Voice note transcription fails, tasks still work | Rotate in OpenAI portal → update secret → rolling restart |
| JWT secret rotated | All active sessions invalidated — users must log in again | Schedule rotation during low-traffic window |

---

## 9. RBAC — Roles and RoleBindings

RBAC controls what Kubernetes resources each service account and human operator can access.

### 9.1 ServiceAccounts

Each component gets its own ServiceAccount with minimum required permissions.

```yaml
# ServiceAccounts — one per component
apiVersion: v1
kind: ServiceAccount
metadata:
  name: komplio-api-sa
  namespace: komplio-prod
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: komplio-frontend-sa
  namespace: komplio-prod
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: komplio-agent-sa
  namespace: komplio-prod
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: komplio-notification-sa
  namespace: komplio-prod
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: komplio-redis-sa
  namespace: komplio-prod
```

### 9.2 Role — Backend API

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: komplio-api-role
  namespace: komplio-prod
rules:
  # API needs to read its own secrets and config
  - apiGroups: [""]
    resources: ["secrets", "configmaps"]
    verbs: ["get", "list"]
  # API needs to check pod health for service discovery
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: komplio-api-rolebinding
  namespace: komplio-prod
subjects:
  - kind: ServiceAccount
    name: komplio-api-sa
    namespace: komplio-prod
roleRef:
  kind: Role
  name: komplio-api-role
  apiGroup: rbac.authorization.k8s.io
```

### 9.3 Role — Task Agent (Whisper)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: komplio-agent-role
  namespace: komplio-prod
rules:
  # Agent only needs its own AI secrets and app config
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["komplio-ai-secrets"]  # scoped to specific secret
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["komplio-app-config"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: komplio-agent-rolebinding
  namespace: komplio-prod
subjects:
  - kind: ServiceAccount
    name: komplio-agent-sa
    namespace: komplio-prod
roleRef:
  kind: Role
  name: komplio-agent-role
  apiGroup: rbac.authorization.k8s.io
```

### 9.4 Role — Notification Service

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: komplio-notification-role
  namespace: komplio-prod
rules:
  # Notification service only needs notification secrets
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["komplio-notification-secrets"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["komplio-app-config", "komplio-notification-templates"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: komplio-notification-rolebinding
  namespace: komplio-prod
subjects:
  - kind: ServiceAccount
    name: komplio-notification-sa
    namespace: komplio-prod
roleRef:
  kind: Role
  name: komplio-notification-role
  apiGroup: rbac.authorization.k8s.io
```

### 9.5 Human operator roles

```yaml
# DevOps engineer — full access to komplio-prod namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devops-full-access
  namespace: komplio-prod
subjects:
  - kind: User
    name: devops@komplio.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
---
# Developer — read-only access to prod, full access to staging
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-readonly-prod
  namespace: komplio-prod
subjects:
  - kind: User
    name: developer@komplio.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

---

## 10. Inter-Service Communication

### 10.1 Communication matrix

```
User Browser
    │
    ▼ HTTPS (443)
LoadBalancer Service
    │
    ▼ HTTP (8000)
Frontend Pod (Jinja2)
    │
    ├──► komplio-api-svc:8080 (ClusterIP) ──► FastAPI Pod
    │         │
    │         ├──► komplio-task-agent-svc:8081 (ClusterIP) ──► Whisper Agent Pod
    │         │         │
    │         │         └──► OpenAI Whisper API (external HTTPS)
    │         │
    │         ├──► komplio-notification-svc:8082 (ClusterIP) ──► Notification Pod
    │         │         │
    │         │         ├──► Meta WhatsApp Cloud API (external HTTPS)
    │         │         └──► Gmail SMTP (external TLS)
    │         │
    │         ├──► komplio-redis:6379 (ClusterIP headless) ──► Redis StatefulSet
    │         │
    │         └──► Supabase PostgreSQL (external HTTPS)
    │
Admin Browser
    │
    ▼ NodePort (30090) — VPN only
Admin Panel Pod
```

### 10.2 DNS-based service discovery

Within the cluster, services communicate using Kubernetes DNS:

```python
# FastAPI calling the Task Agent internally
import httpx

TASK_AGENT_URL = os.getenv("TASK_AGENT_URL")
# Value from ConfigMap: "http://komplio-task-agent-svc:8081"

async def transcribe_voice(audio_file: bytes) -> str:
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{TASK_AGENT_URL}/transcribe",
            content=audio_file,
            headers={"Content-Type": "audio/m4a"},
            timeout=30.0
        )
    return response.json()["transcript"]
```

### 10.3 NetworkPolicy — restrict pod communication

```yaml
# Only allow API pod to call Task Agent — nothing else can
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: task-agent-ingress-policy
  namespace: komplio-prod
spec:
  podSelector:
    matchLabels:
      component: task-agent
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              component: api     # only API pod can reach Task Agent
      ports:
        - protocol: TCP
          port: 8081
---
# Only allow API pod to call Notification Service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: notification-ingress-policy
  namespace: komplio-prod
spec:
  podSelector:
    matchLabels:
      component: notification-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              component: api
      ports:
        - protocol: TCP
          port: 8082
```

---

## 11. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│  Namespace: komplio-prod                                             │
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌─────────────────────┐   │
│  │   Frontend   │    │  Backend API │    │    Task Agent       │   │
│  │  Deployment  │───▶│  Deployment  │───▶│    Deployment       │   │
│  │  2 replicas  │    │  2 replicas  │    │  (Whisper AI)       │   │
│  │  LB Service  │    │  ClusterIP   │    │  ClusterIP Service  │   │
│  └──────────────┘    └──────┬───────┘    └─────────────────────┘   │
│                             │                                        │
│                    ┌────────┼────────┐                              │
│                    ▼                 ▼                              │
│           ┌──────────────┐  ┌──────────────┐                       │
│           │ Notification │  │    Redis     │                       │
│           │   Service    │  │ StatefulSet  │                       │
│           │  Deployment  │  │  ClusterIP   │                       │
│           │  ClusterIP   │  │   (headless) │                       │
│           └──────────────┘  └──────────────┘                       │
│                                                                      │
│  ConfigMaps: komplio-app-config, komplio-notification-templates     │
│  Secrets:    komplio-db-secrets, komplio-app-secrets,               │
│              komplio-notification-secrets, komplio-ai-secrets        │
│  RBAC:       5 ServiceAccounts, 4 Roles, 4 RoleBindings             │
└─────────────────────────────────────────────────────────────────────┘
          │                          │
          ▼ (external)               ▼ (external)
   Supabase PostgreSQL         Meta WhatsApp API
   Supabase Storage            OpenAI Whisper API
                               Gmail SMTP
```

---

## Summary

| Item | Count | Details |
|---|---|---|
| Deployments | 5 | Frontend, API, Task Agent, Notification Service, Admin |
| StatefulSets | 1 | Redis |
| Services | 6 | 1 LoadBalancer, 4 ClusterIP, 1 NodePort |
| ConfigMaps | 2 | App config, Notification templates |
| Secrets | 5 | DB, App, Notification, AI, Redis |
| Namespaces | 4+ | prod, dev, staging, monitoring, per-tenant |
| ServiceAccounts | 5 | One per component |
| Roles | 4 | Scoped per component |
| RoleBindings | 6 | Components + human operators |
| NetworkPolicies | 2 | Task Agent, Notification Service ingress |
| HPA | 1 | Backend API auto-scaling |
