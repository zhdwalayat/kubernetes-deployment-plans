# Plan 1: Kubernetes Deployment Plan — AI Native Task Manager

## Application Overview

| Component | Role | External Access |
|---|---|---|
| UI Interface | Frontend dashboard for users | Yes — public facing |
| Backend APIs | Core business logic and data layer | No — internal only |
| Todo Agent | AI-powered task processing worker | No — internal only |
| Notification Service | Sends alerts via UI and Backend | No — internal only |

---

## 1. Namespace

All components run inside a dedicated namespace for isolation.
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: task-manager
```

> **Why:** Namespaces isolate resources, prevent naming conflicts, and allow
> separate RBAC policies per environment.

---

## 2. Deployments

All four components are stateless, so `Deployment` is used (not `StatefulSet`).

### UI Interface
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui-interface
  namespace: task-manager
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ui-interface
  template:
    metadata:
      labels:
        app: ui-interface
    spec:
      serviceAccountName: ui-sa
      containers:
      - name: ui-interface
        image: ui-interface:latest
        ports:
        - containerPort: 3000
        envFrom:
        - configMapRef:
            name: app-config
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "300m"
            memory: "256Mi"
```

### Backend APIs
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-apis
  namespace: task-manager
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-apis
  template:
    metadata:
      labels:
        app: backend-apis
    spec:
      serviceAccountName: backend-sa
      containers:
      - name: backend-apis
        image: backend-apis:latest
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

### Todo Agent
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-agent
  namespace: task-manager
spec:
  replicas: 2
  selector:
    matchLabels:
      app: todo-agent
  template:
    metadata:
      labels:
        app: todo-agent
    spec:
      serviceAccountName: todo-agent-sa
      containers:
      - name: todo-agent
        image: todo-agent:latest
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: app-config
        resources:
          requests:
            cpu: "300m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
```

### Notification Service
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-service
  namespace: task-manager
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
      serviceAccountName: notification-sa
      containers:
      - name: notification-service
        image: notification-service:latest
        ports:
        - containerPort: 4000
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: notification-secret
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "300m"
            memory: "256Mi"
```

---

## 3. Services

| Component | Service Type | Port | Reason |
|---|---|---|---|
| UI Interface | LoadBalancer | 80 | Needs public internet access |
| Backend APIs | ClusterIP | 8080 | Internal only — called by UI and Todo Agent |
| Todo Agent | ClusterIP | 5000 | Internal only — called by Backend APIs |
| Notification Service | ClusterIP | 4000 | Internal only — connects to UI and Backend |
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ui-interface-svc
  namespace: task-manager
spec:
  type: LoadBalancer
  selector:
    app: ui-interface
  ports:
  - port: 80
    targetPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: backend-apis-svc
  namespace: task-manager
spec:
  type: ClusterIP
  selector:
    app: backend-apis
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: todo-agent-svc
  namespace: task-manager
spec:
  type: ClusterIP
  selector:
    app: todo-agent
  ports:
  - port: 5000
    targetPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: notification-svc
  namespace: task-manager
spec:
  type: ClusterIP
  selector:
    app: notification-service
  ports:
  - port: 4000
    targetPort: 4000
```

---

## 4. ConfigMaps

Non-sensitive configuration stored as ConfigMaps.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: task-manager
data:
  BACKEND_API_URL: "http://backend-apis-svc:8080"
  TODO_AGENT_URL: "http://todo-agent-svc:5000"
  NOTIFICATION_URL: "http://notification-svc:4000"
  LOG_LEVEL: "info"
  ENVIRONMENT: "production"
  MAX_TASKS_PER_USER: "100"
```

---

## 5. Secrets

Sensitive data stored as Kubernetes Secrets. Never stored in ConfigMaps or source code.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: task-manager
  annotations:
    secret-expiry-date: "2025-12-31"
    rotation-schedule: "every-90-days"
type: Opaque
data:
  DB_PASSWORD: <base64-encoded-value>
  API_KEY: <base64-encoded-value>
  JWT_SECRET: <base64-encoded-value>
---
apiVersion: v1
kind: Secret
metadata:
  name: notification-secret
  namespace: task-manager
  annotations:
    secret-expiry-date: "2025-12-31"
    rotation-schedule: "every-90-days"
type: Opaque
data:
  NOTIFICATION_API_KEY: <base64-encoded-value>
```

**Secret Management Rules:**
- Secrets are annotated with `secret-expiry-date` for rotation tracking
- Each component accesses only its own Secret — no shared master secret
- Secrets rotated every 90 days
- In production: integrate with HashiCorp Vault or AWS Secrets Manager

---

## 6. RBAC — Roles and RoleBindings

Each component runs under its own ServiceAccount with minimum required permissions (least privilege principle).
```yaml
# ServiceAccounts
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ui-sa
  namespace: task-manager
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: task-manager
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: todo-agent-sa
  namespace: task-manager
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: notification-sa
  namespace: task-manager
---
# Backend APIs Role — can read configs and secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-role
  namespace: task-manager
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-rolebinding
  namespace: task-manager
subjects:
- kind: ServiceAccount
  name: backend-sa
  namespace: task-manager
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: backend-role
---
# Todo Agent Role — most restricted, only reads configmaps
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: todo-agent-role
  namespace: task-manager
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: todo-agent-rolebinding
  namespace: task-manager
subjects:
- kind: ServiceAccount
  name: todo-agent-sa
  namespace: task-manager
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: todo-agent-role
---
# Notification Service Role — reads its own secret and configmap
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: notification-role
  namespace: task-manager
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: notification-rolebinding
  namespace: task-manager
subjects:
- kind: ServiceAccount
  name: notification-sa
  namespace: task-manager
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: notification-role
```

---

## 7. Inter-Service Communication
```
Internet
    │
    ▼
UI Interface (LoadBalancer :80)
    │
    ├──► Backend APIs (ClusterIP :8080)
    │         │
    │         └──► Todo Agent (ClusterIP :5000)
    │
    └──► Notification Service (ClusterIP :4000)
               │
               ├──► Backend APIs (ClusterIP :8080)
               │
               └──► UI Interface (ClusterIP :3000)
```

| From | To | Protocol | Port | Direction |
|---|---|---|---|---|
| User (Internet) | UI Interface | HTTP | 80 | Inbound |
| UI Interface | Backend APIs | HTTP | 8080 | Internal |
| UI Interface | Todo Agent | HTTP | 5000 | Internal |
| Todo Agent | Backend APIs | HTTP | 8080 | Internal |
| Notification Service | Backend APIs | HTTP | 8080 | Internal |
| Notification Service | UI Interface | HTTP | 3000 | Internal |

All internal calls use Kubernetes DNS:
`<service-name>.<namespace>.svc.cluster.local`

Example: `http://notification-svc.task-manager.svc.cluster.local:4000`


## 8. HorizontalPodAutoscaler — Todo Agent

The Todo Agent handles AI workloads that spike unpredictably. HPA scales it automatically based on CPU usage.
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: todo-agent-hpa
  namespace: task-manager
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: todo-agent
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## Summary Table

| Component | Type | Replicas | Service Type | CPU Limit | Memory Limit |
|---|---|---|---|---|---|
| UI Interface | Deployment | 2 | LoadBalancer | 300m | 256Mi |
| Backend APIs | Deployment | 3 | ClusterIP | 500m | 512Mi |
| Todo Agent | Deployment | 2–8 (HPA) | ClusterIP | 1000m | 1Gi |
| Notification Service | Deployment | 2 | ClusterIP | 300m | 256Mi |
